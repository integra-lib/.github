# Where the components came from

The components were consolidated in September 2026 out of copies that had drifted
across firmware projects. The per-component repositories start from a clean slate
by decision, so this file keeps the part of that history worth keeping: what the
code came from, and what was deliberately changed on the way.

The public API is grouped under `hwlib::<section>` and `include/hwlib/<section>/`:
`utilities`, `algorithms`, `data_structures`, `communication`, `events`,
`execution` and `persistence`. Repository names remain unchanged.

## The problem being solved

* `utils.hpp` existed in **eight** projects — a138-ble-gateway, a000-firmware-update-over-modbus,
  a151-sib-bathtub-controller, a168-waveform-generator, a084_one-home_pde37,
  a146-control-board, a159-bpu-firmware, a156-dl200p — in eight different sizes,
  each a truncation of the others.
* `function.hpp` existed in a160-oto-screening and a173-baep, differing by exactly
  one constructor: a160 had `Function(std::nullptr_t)`, a173 never received it.
  `ring_buffer.hpp` next to it was byte-identical, so the divergence had accumulated
  in one spot and gone unnoticed.
* CRC lived in five shapes: three CRC-32 copies with the same `0xEDB88320`
  polynomial and three different APIs (a000-bootloader, a168, a160's bootloader),
  a CRC-8 locked inside a138-ble-sensors' SHT40 driver, and a CRC-16 inside the
  transaction engine.
* `LockGuard` existed in three incompatible variants — over a `util::IMutex`
  interface, over `k_mutex` directly, and as a `SpinlockGuard` over `k_spinlock`.

## Changed on the way in, and why

* **`Function::operator=` did not compile.** It passed an object to
  `unique_ptr::reset` and never returned `*this`. Being a template, it was only
  instantiated on use, and nothing had ever used it. Rewritten with
  `std::make_unique`; the regression test fails to compile if the defect returns.
* **`Function` violated the rule of five** — copy constructor deleted, copy
  assignment `= default`. Now both are deleted, move is defaulted.
* **`ConstMap::at` threw `std::range_error`**, which does not survive a build with
  exceptions disabled. Replaced by `Find` returning `std::optional`; a missing key
  is a value, not an error.
* **`OutputIteratorTraits` was not carried over.** It relied on
  `std::raw_storage_iterator`, removed in C++20, and was unused even in the single
  project that had it.
* **`struct overload` became `Overload`**, because the shared `.clang-tidy` treats
  a lowercase struct name as an error and the library prefers one exception fewer.

## Left behind on purpose

`frame-transport-io` and `transaction-engine-log` from a174-hardware log through
Zephyr's `LOG_MODULE_REGISTER`. They belong to the adapter layer, and porting them
would mean redesigning the logging into an injected callback rather than moving
code.

## Known and unverified

`transaction-engine` is the only static library here. Under Zephyr a static library
must be given the SDK's compile flags or it is built with a different ABI than the
application linking it. This was never verified on a real NCS toolchain — do that
before the component goes into firmware. Header-only components are compiled as
part of the application and have no such concern.

## A second round: three components from a174-hardware

September 2026, after the first eleven. These came from one project rather than from
eight copies of the same file, so the reason to move them is different: each is logic
four applications in a174-hardware already share, wrapped in Zephyr calls that had
nothing to do with the logic.

The cut is the same in all three — the platform stays in the project, the decision
moves into the library:

* **event-manager** (`firmware/common/event-manager`). `k_msgq` became the
  `EventQueueLike` concept and the manager borrows a queue instead of owning one;
  `k_uptime_get_32()` became the `nowMs` parameter of `Push()`; the two `LOG_WRN`
  calls became callbacks. The a174 default payload — a `std::variant` of
  `std::monostate`, `float` and `std::uint8_t` — was not carried over: it is that
  project's domain choice, not a library default.
* **settings-record** (`firmware/common/settings-storage`). Only the record format
  came across. `NvsStorage` stayed behind with the flash device and the
  `FIXED_PARTITION(settings_storage)` name it needs, and the component talks to
  whatever satisfies `RecordStorageLike`.
* **button-event** (`firmware/common/drivers/button-event`). The state machine came
  across, and with it went `IGpioInputPin`, `k_work`, `k_work_delayable`,
  `CONTAINER_OF`, `util::ZephyrTimer` and `k_uptime_get()`. The caller samples the pin
  and acts on the `ButtonAction` the core returns.

## Changed on the way in, second round

* **`CalcCrc` was a fourth copy of CRC-32/ISO-HDLC.** Same `0xEDB88320`, same
  initial and final XOR, a fifth of a page of hand-written loop inside
  `settings-record.cpp`. It now calls `hwlib::algorithms::Crc32IsoHdlc`. A test
  pins the result against both the catalogue check value for `"123456789"` and the original's own loop,
  because the wrong checksum here would not fail a build — it would quietly reject
  every setting a device had already saved.
* **A settings payload that is not trivially copyable is now a `static_assert`.**
  The original compiled it and would have written a pointer to flash.
* **The storage contract takes `std::span`** instead of `const void*` and a length,
  so a call cannot pass a size that does not match the buffer. An existing
  `NvsStorage` needs its two signatures widened; the bodies do not change.
* **`ButtonEventController`'s two `std::atomic` flags became plain `bool`s.** They
  advertised a thread safety the class never had — `m_pressTime` sat next to them as
  a plain `int64_t` — and the real rule was always that the ISR-context edge and
  timeout are deferred to a work queue. That rule is now written down as the core's
  contract instead of being implied by two of five members.

## Left behind on purpose, second round

`NvsStorage` and the Zephyr adapters for the other two. A partition name, a flash
device, a `k_msgq` and a `k_work` are the project's, and moving them would mean the
library picking a platform.

## And one more: hexstrconv, in two projects

`hexstrconv` existed in a138-ble-gateway (`firmware/lib/utils`) and in scale
(`components/utils`) with the same two functions drifted apart — the `function.hpp`
story again. Each copy had caught a defect the other had not:

* scale's `FromHexStr` accepted `"0x"` as a single zero byte. `std::from_chars`
  reports success on a partial parse and neither copy checked that it had consumed
  both characters; a138's copy only escaped it by pre-scanning with `isxdigit`.
* that pre-scan passed a possibly negative `char` to `std::isxdigit`, which is
  undefined for anything but `unsigned char` values and `EOF`. One byte above 0x7F in
  the input is enough.

Both are fixed in `hex-string`, and both are pinned by a test. The API changed to
enter a library built for firmware: the `std::vector` and `std::string` returns became
a caller's buffer and a written count, the three `throw`s became `std::optional`, and
everything is `constexpr`. `ToHexStrInverted` became `BytesToHexReversed` — the a138
copy carried a `// TODO: fix mac -> string presentation` beside it, and reversed byte
order is the presentation a radio hands over, not a defect.

Left behind: `FromStr2Bytes`, which converts characters to bytes and has nothing to do
with hex, and `IntToHexStr`, which formats through `std::stringstream`.

## And three more: worker, debouncer, mqtt-topic

* **worker** is `BaseWorker` from a138-ble-gateway's `lib/shared-slip`, which the
  ESP32 and nRF halves of that project each aliased with their own mutex.
  `Post(const Work&& work)` copied every callable — `std::move` of a `const&&` is
  still `const`, so the push picked the copy constructor — and the drain copied the
  front of the queue before popping it; both now move. The `LockGuard` template
  parameter gave way to a `lock()`/`unlock()` concept and `std::scoped_lock`, which
  also retired the nRF worker's initialiser callback that threw from a constructor.
  `GetInstance()` and the project's `IUpdateableObject` base stayed behind.
* **debouncer** is `util::Debouncer` from a160-oto-screening's `lib/util`. The
  counting is unchanged. `Compare(Comparer)` became `Update(bool)` — the callable was
  invoked once, immediately — the conversion to `bool` became explicit, since the
  implicit one let `int n = debouncer;` compile, and the header now includes the
  `<algorithm>` and `<cstddef>` it used and had only been getting by include order.
* **mqtt-topic** is `MatchTopic` from a138-ble-gateway's
  `esp32/components/mqtt-helper`; the ESP-IDF client around it stayed in the project.
  It was rewritten to walk levels, because it compared a `#` filter as a string
  prefix — `sensors/#` matched `sensorsX/temp` — and missed the parent level after a
  `+`, so `+/#` did not match `a`. All 32 `static_assert` cases it carried are kept.

## And filters, from four projects

`filters` gathers what a138-ble-sensors (`MedianFilter`, `HysteresisFilter`),
a163-cgm-firmware (`MedianFilter`) and a139-bms48v-firmware (`sma`) each wrote for
themselves. Every copy had a defect, all four confirmed by running the original:

* a138's median computed an even result as `(a + b) / 2` in the sample type — four
  `std::uint32_t` samples of 4 000 000 000 gave 1 852 516 352;
* a163's median reset its counter in `SetWindowSize` but not its "ready" flag, so a
  full filter reconfigured over the air answered from the old samples and ignored the
  new ones;
* a138's hysteresis guarded the unsigned step down with `boundary >= band`, which also
  froze a signed filter in its upper state once a threshold was at or below the band;
* a139's average kept its sum in the sample type — four `std::uint8_t` of 200 averaged
  to 8 — and divided by the window from the first sample.

Left behind: `IFilter`, which nothing used polymorphically; a139's `cma`, `wma` and
`ema`, which only its own tests used; and a021-smart-helmet's integer
`ExponentFilterFast`, which belongs next to ema-filter rather than here.

## And littlefs-cpp, from a163-cgm-firmware

The wrapper is a163's `utils::Lfs`, `LfsFile`, `ScopedLfsFile` and `LittlefsIterator`.
Four defects, each confirmed by running it against littlefs v2.10.1 on a RAM block
device:

* it formatted on any mount failure — one transient read error at boot erased every
  file;
* `FileCount` left its directory on littlefs's open list with the `lfs_dir_t` on a
  finished stack frame, which the next file open read (AddressSanitizer:
  stack-use-after-return);
* a moved `ScopedLfsFile` kept believing it was open and closed a null handle,
  stopped by littlefs's own assertion — a null dereference under `LFS_NO_ASSERT`,
  which a163's release build sets;
* `LfsFile` passed a `std::string_view` to littlefs as a C string, so `"/a"` taken
  out of `"/ab"` opened `"/ab"`.

Not carried over: the `LFS_THREADSAFE` locking, never enabled and one static mutex
for every filesystem; and `lfs_api.hpp`, a163's ring-of-files layer on top.

## Known and unverified, second round

* `event-manager` and `button-event` hold their handlers in `std::function`, which
  allocates for a large enough capture. Registration happens once at startup, but a
  context that forbids the heap outright has to know.
* `button-event`'s one-context rule is a contract, not a compiler error. a174-hardware
  already honoured it; a new adapter that calls the core straight from an ISR would
  compile.
* Nothing here has run on a device yet. The components are verified by their own
  test suites on the host, under gcc and clang; periodic-clock has no test suite and
  its header was syntax-checked with both compilers.

## And one more: byte-codec

`byte-codec` consolidates integer, enum and floating-point serialization from packet
code in a169 and a130. It provides endian-aware `Store` and `Load` functions and a
`ByteReader` that borrows its input and keeps a sticky failure state. Array reads are
all-or-nothing. Shifts happen in an unsigned type matching the encoded width, avoiding
the signed integer-promotion undefined behavior in the original code.

## And boot-slots, from a184-480w-ups-controller

`boot-slots` is the bootloader logic of a184 (commit `a444a5e`):
`Modules/Bootloader_Logic`, the transitions of `Modules/Firmware_Manager_Update`,
and the metadata journal and image check of `Modules/Firmware_Manager`. a184 is a C
project under MISRA C:2012, so the component stayed C (C11, builds as C99 too) —
the only one in hwlib — with every hardware access moved behind a callback: flash,
slot reads, the CRC unit, the device-family rule, the watchdog feed.

The journal record and the image header are byte for byte a184's and
`tools/build_firmware.py`'s; the tests pin both against bytes generated with
Python's `zlib.crc32`. Changed on the way:

* **A read error could erase the journal.** a184's scan treated an unreadable
  sector like an empty one, and a save with "no free record" reclaims by erasing.
  `hwlib_boot_journal_save()` now fails instead.
* **Loading could not tell a first boot from a lost state.** a184's load returned
  `false` for an erased sector, a read error and a sector with no intact record
  alike, and the bootloader started from the factory state in every case.
  `hwlib_boot_journal_load()` returns which one it was; a read error is no longer a
  first boot. Found by an external review.
* **The write buffer left the commit bytes uninitialised** in the first C port; a
  mutation test that wrote the whole record at once exposed it. The record is now
  built in full and written in two parts.
* **The vector check's overflow guard was dead code:** a wrapped payload already
  fails the reset-handler range check. A surviving mutant showed it; it was removed.
* `firmware_manager_confirm_current` read `SCB->VTOR`; `hwlib_update_confirm` takes
  the running slot as an argument.

Known and unverified: the window between the reclaim erase and the first commit —
power lost there loses the state, as in a184; closing it needs a second sector and a
new format. Not yet run through cppcheck's MISRA addon (Rule 15.5, multiple returns,
will need suppressions in a184), not built with Keil, and not run on a device.

## And can-filter-codec, from a138-ble-sensors

`CanFilterCodec` sat in a138-ble-sensors' `shared-components/drivers/can`, between
a GATT characteristic and the CAN driver: it packs the list of acceptance filters a
phone or a PC sets into a versioned TLV message. It is the one part of that module
whose Zephyr use was incidental — `struct can_filter` and `sys_get_le32` — so it came
over; the GATT reassembly buffer, the NVS storage and the manager stayed.

The format did not change. The two codecs were run side by side on 200 000 randomly
damaged messages: they never produced different filters, and every message only
a138 accepted falls under one of the first two points below. That holds for the
compiler it ran on: a138 forms `p + 2` and `p + len` before comparing them with the
end, which on a short message is a pointer past one-past-the-end and undefined
behaviour, so its results elsewhere are not guaranteed.

* **A filter without its flags record was decoded with flags 0** and reported as a
  success — an extended filter silently became a standard one. It is refused.
* **Ids and masks wider than 29 bits passed through** to the CAN driver. They are
  refused on both sides.
* **256 filters were encoded as a message announcing none:** the count was
  `static_cast<uint8_t>(size)`. More than 255 are refused.
* **A refused message had already overwritten part of the output.** It is left
  untouched.

The input and output buffers must not overlap, in either direction; that is
documented, not checked. An external review found it — the decoder reads the
message again after writing — and no caller shares the storage.

Known and unverified: no client of the format was found in the a138 repositories,
so compatibility is checked against a138's firmware only, not against the app that
writes the filters.
