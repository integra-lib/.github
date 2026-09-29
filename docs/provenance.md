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

## And stc3100, from a160-oto-screening

Three projects talk to ST's STC3100 coulomb counter. a160-oto-screening has a C
driver of its own (`firmware/lib/board/src/mcu/stc3100.c`); a156-dl200p and
a159-bpu-firmware share another lineage (`gas_gauge.c` under a `BatteryMonitor`).
a160's was taken — it checked every transfer and converted to units — and the
fixes cover what the other two got wrong as well. Everything was checked against
the STC3100 datasheet, Rev 1.

* **Resetting the charge drove the IO0 pin low** in all three. They wrote 0x02 to
  REG_CTRL for GG_RST, which also writes IO0DATA = 0, "IO0 output is driven low".
  The driver writes IO0DATA = 1.
* **Unbounded waits.** a160's init looped until the voltage read 500 mV, which is
  refreshed only every 4 s, and forever without a battery; a156 spun on VTM_EOC and
  on retrying a CTRL write, a159 on the latter. The driver never waits.
* **Failed reads became values:** a160 returned the converted zero from its buffer
  along with the error code, a156 an uninitialised charge that went into the state
  of charge. Reads are `std::optional`.
* **Precision:** a160 returned current in whole milliamps and temperature in whole
  degrees; one current LSB across 33 mOhm is 357 uA and read as 0. The driver
  returns uA and m°C.
* a160 hard-coded the sense resistor; a156 read registers by casting bytes to
  `uint16_t`, little-endian only; a156 inverted the result of every register write.
* Left behind: a160's linear 3100–3700 mV battery level (the product's cell) and
  the three `BatteryMonitor`s, the state-of-charge algorithm, which is a component
  of its own.

Known and unverified: not run against a chip; the upper bits of the voltage
register are not masked, as the datasheet does not give its width.

## And battery-monitor, from a156, a159 and a160

The three STC3100 projects each had a `BatteryMonitor` on top of the chip: OCV at
start, coulomb counting, the offset in the chip's RAM with an inverted copy.
a156-dl200p and a159-bpu-firmware share one lineage, a160-oto-screening wrote its
own with a learned charging efficiency. The algorithm came over; the component
drives any gauge through a concept, and hwlib's stc3100 satisfies it.

* **The stored offset went stale after every correction.** All three zeroed the
  chip's counter when clamping — a160 also on every change between charging and
  discharging — without storing the new offset, so a reset of the MCU brought the
  old offset back over a zeroed counter. a156's `SaveBatterySoc(m_soc)` compared
  the value with itself and never saved. Corrections now move the offset only and
  store it; the counter is reset only near overflow, stored first.
* **An `int16_t` offset**, about 6.6 Ah at 33 mOhm, with a silent narrowing on
  every fold. It is `int32_t` uAh.
* **a160 let NaN through** its check on the float it read from RAM, and accepted a
  stored state past full. The state is an integer, checked to empty..full.
* **a156 waited for 3.4 V in a loop**, which a flat battery never gives;
  `Start()` reports eNotReady instead.
* a160 threw at low battery, timed itself with `system_clock` and shared `static`
  debouncers between instances; an OCV table with a repeated voltage divided by
  zero in all three.
* Left behind: a160's learned efficiency, a156's ×1.15 and 99 % while charging,
  a160's 10 %-is-empty, the charger-pin logic — product decisions.

Mutation testing: 47 of 49 mutants caught; the two left are equivalent, the
interpolation being continuous at a table point. One caught only under UBSan —
reading past the table at its last voltage. Not run on a device.

## And coro, from a139-bms48v-firmware

a139's `lib/coro` is a C++23 coroutine library in the manner of cppcoro — task,
sync_wait, when_all_ready, async_scope, async_generator, an io_service scheduler and
two queues — used by its nRF52840 BMS firmware with exceptions and RTTI on. It came
over whole, renamed to hwlib's style, C++20, with or without exceptions. Every
defect below was reproduced by building a139's own headers and running the case.

* **The scheduler lost coroutines.** Past its 32-slot queue it put the overflow back
  on its intrusive list by the tail instead of the head: of 100 coroutines scheduled
  at once, 33 ran. The scheduler is now an intrusive lock-free stack that the one
  running thread turns into a FIFO — no capacity, nothing to overflow.
* **`join()` did not wait for a single remaining work** and then resumed the joiner a
  second time: the count started at 0 where cppcoro's starts at 1, but `join()` kept
  cppcoro's `> 1`.
* **`task.hpp` did not link from two translation units**: a non-template member was
  defined in the header without `inline`.
* **Move assignment leaked a coroutine frame** in `when_all_task` and
  `async_generator`; **the queues never destroyed what was left in them.**
* `sync_wait()` on a task returning `T&` did not compile; the scheduler used
  `high_resolution_clock` (the system clock in libstdc++) and could release its
  `counting_semaphore<1024>` past the maximum.

An external review then found, and the component fixes: `Run()` could spin on
self-rescheduling work without reaching its timers or its stop request (it now
works in rounds); `AsyncScope` could be destroyed with work running or spawned into
after `Join()` (both end the program); `Resume()` and `Result()` on a finished or
empty task were unchecked. The clock must now be steady at compile time.

Found while testing: clang at `-O1` reuses `std::this_thread::get_id()` across a
`co_await` although the coroutine moved threads there — noted in the README.

Tests: 73, under gcc, clang, ASan+UBSan and TSan, with the README's examples among
them; mutation-tested file by file, each regression test checked to fail on
a139's behaviour. Known and unverified: not run on a device; frames are heap-allocated; Cortex-M0
needs `__atomic_*` from libatomic.

## And sht40, from a138-ble-sensors and a138-ble-gateway; and i2c-bus

a138 had two SHT40 drivers: `shared-components/drivers/sht40` in
a138-ble-sensors, behind its `II2cDevice` and `ILogger` interfaces, and
`firmware/esp32/components/sht40` in a138-ble-gateway on ESP-IDF. Neither was
taken whole; the driver was written against the SHT4x datasheet (version 6.5,
April 2024) and each defect of the two is pinned by a test.

* **gateway: a reading whose CRC failed came back as 0 °C and 0 %RH** — the zero
  initialiser, returned as a measurement. It is an error now.
* **gateway: a product's −3 °C offset in the driver; humidity uncropped** (the
  clamp commented out, so −6 and 119 %RH came out); exceptions.
* **sensors: `double` arithmetic**, software on a Cortex-M4F; **one failed reset in
  the constructor disabled the driver for good**; 0.0 when not initialised, which
  reads as a real 0 °C; negative temperatures logged as `-3.-45`.
* Both waited inside the driver, the gateway 100 ms for a 1.6 ms measurement.

The conversion is integer and 32-bit only — no software 64-bit division on a
Cortex-M0 — checked against the formula for all 65536 raw values.

The bus concept came out of stc3100 into **i2c-bus**, split per transfer
(`I2cWrite`, `I2cRead`, `I2cWriteRead`): the SHT40 reads without a register
address, which stc3100's write + write-read concept could not express, and two
drivers each declaring the same concept would not compile together. stc3100 0.2.0
takes it from there.

## And mfrc522, from a174-hardware; and spi-bus

a174's `firmware/common/drivers/nfc-reader-mfrc522` had three layers: registers
over Zephyr SPI, ISO/IEC 14443-3 activation, and a reader on a Zephyr work queue
with a callback. The first two were rewritten against the MFRC522 datasheet
(Rev. 3.9, April 2016) and ISO/IEC 14443-3 as `Mfrc522<Spi>` and a protocol-only
`Iso14443aActivation`; the third was not taken — polling is the caller's.

* **The SAK was taken without its CRC_A**, which neither the chip (RxCRCEn off
  after reset) nor the driver checked: one flipped cascade bit changed the UID's
  length.
* **A collision was not an error**: CollErr was missing from the error mask, so two
  cards read as a clean answer and failed later, if at all, on the BCC.
* **Any answer to HLTA counted as halted**; ISO 14443-3 makes an answer a refusal.
* **The cascade tag rather than the SAK decided the UID's length.**
* It busy-waited inside the driver — up to 5000 × (SPI transfer + 20 µs) per
  exchange — and ran every SELECT's CRC through the chip's coprocessor. The CRC_A
  is software now; an exchange starts in six transfers and a pending poll is one.
* The reader layer replaced its `std::function` callback while the work-queue
  thread could be calling it.

The tests drive a simulated MFRC522 with simulated cards; mutation testing
killed every mutant but one equivalent. An external review found that a pending
exchange is not bounded by the chip once an answer begins — the contract now asks
the caller for a deadline — and that a valid card with an 88h byte in its final
UID part was rejected; fixed. Not run against a chip.

**spi-bus** holds `SpiDevice`, one full-duplex `Transfer(tx, rx)` with the chip
selected, `rx` empty or as long as `tx`, and `FakeSpiDevice` — made a component of
its own at the first SPI driver, as i2c-bus was at the second I2C one.

## And ad7797, max31856 and linear-actuator, from a146-control-board

a146 is a coffee roaster's control board (GD32F470, FreeRTOS, Modbus RTU). Its
drivers live on the GitLab branch `develop`; the copy on the archive drive
predates them. Three were taken, each rewritten against its datasheet — the
AD7796/AD7797 Rev. B, the MAX31856 Rev 0 — with a simulated chip or a modelled
actuator under test, and mutation-tested until no mutant survived.

**ad7797**, the load-cell converter (`sensor-manager/ad7797`, `tenso`):

* the voltage sign-extended an offset-binary code, so a positive full scale read
  as −VREF/128; its `static_assert`s lacked an absolute value and passed anyway;
* the Modbus weight register carried the raw code × 10 as `uint16_t` — overflow,
  and an out-of-range float-to-integer conversion, undefined;
* ERR was ignored, so a clamped overrange went out as a weight;
* `SelectBurnout()` cleared U/B instead of BO; AD7793 bits in the header;
* endless waits on DOUT/RDY and on the ID; function-static state shared by every
  instance.

**max31856**, the thermocouple converter (`sensor-manager/max31856`, `tc-ctrl`):

* faults were probed once, at start: a thermocouple that opened later kept
  reporting whatever the chip read, and one connected later was never read;
* the type-K thermocouple-voltage conversion fed to Modbus was about 1.1 mV
  (some 27 °C) off — the NIST exponential term added ten times and not squared;
  not taken, since the chip linearizes;
* a missing chip only asserted; `ClearFault()` had no effect in comparator mode.

**linear-actuator** (`actuator-ctrl`), four roaster dampers:

* the position table was `reserve()`d and indexed at size 0 — undefined;
* positions and codes did not invert each other (`/ (N − 1)` against `/ N`);
* the calibration had no timeout, its declared `TIMEOUT_MS` unused;
* the sensor's direction was assumed; a FreeRTOS timer and the task both drove
  the motor; four actuators shared one overwriting notification slot.

`motor-actuator-ctrl` and `motor-ctrl`, the roaster's own door sequences and geared
motor, were not taken. None of the three has run on a board yet.

## And bresenham-modulator, from a146-control-board

a146's `heat-ctrl/bresenham.hpp` switches the roaster's heater through a
solid-state relay, one step per mains half-cycle. Taken as the algorithm alone;
the start/stop/fail state and the zero-crossing timer stay with the board.

* **Setting the value restarted the pattern**: `SetValue()` reset the error and
  the step count, so the duty depended on how often it was called — modelled on
  a146's code, set every 10 half-cycles, a period of 100 came out in whole tens:
  1 % and 5 % gave 0 %, 25 % gave 20 %, 37 % gave 40 %.
* **A race**: the task wrote three fields that the timer interrupt read and wrote.
  The level is now the only shared state, a 16-bit atomic — plain loads and stores
  even on a Cortex-M0, checked in the generated code.
* `GetSize()` truncated a 16-bit size to 8 bits.

Tested exhaustively for exactly `level` ons in every window of a period, gaps
within one step and a count within half a step of the ideal; mutation-tested.

## And fan5646, from a156-dl200p

a156's `indication/tiny_wire` programmed a FAN5646 LED blinker over TinyWire
generated on nRF SPIM's MOSI, one byte per bit. The encoding was right and is
kept; read against the FAN5646 datasheet, Rev. 1.0.3:

* `SendExec()` executed nothing: it held CTRL low for 200 µs, which sends the chip
  to IDLE; the chip executes on CTRL held *high*, which the application did later
  as a GPIO. That hand-over is now the documented, timing-critical contract.
* Each word was followed by 200 µs of low CTRL (`k_sleep` in the driver), IDLE
  between every register, and `BlinkSet()` was called twice "to ensure" it was
  received. All five words now go out in one continuous transfer.
* A static class on a fixed SPIM instance with a static buffer.

The tests decode the SPI stream by Figure 14 and Tables 12 and 13 in a simulated
chip. An external review questioned the leading low byte of each word — it has
no rising edge, so the first edge is A0's — and asked for the adapter's duties
(MSB first, no gaps, MOSI idle low) and the conservative 40 µs bound between
words to be stated and tested; both done. The part is end of life.

## And mcp4922, from a168-waveform-generator

a168's `lib/mcp4922` drove the dual DAC of a waveform generator over nRF SPIM3.
Its command format matched Register 5-1 of the datasheet (DS22250A) and is kept.

* **A board's limit lived in the driver**: every code was clipped to 3430 —
  `MAX_ALLOWED_DAC_500UA_VAL`, that board's 500 µA output limit — silently, while
  the application scaled to 4095. Codes are 0 to 4095 now; one over is refused.
* **A failed transfer only asserted**; a release build lost the write. Every write
  returns whether it happened.
* Pins, SPIM3 and the `latch` alias were fixed in the driver.

Noticed and not taken: the application shares LDAC between the two channels, so
latching channel A also latches channel B's pending value early. A static review
added two notes — do not pulse LDAC after a failed `WriteBoth()`, and a software
power-down is assumed, not stated, to wait for LDAC — and the simulator's limits.

## And adxl345, and a bus clear for i2c-bus, from a159-bpu-firmware

a159's `sensor_manager/adxl345` read the accelerometer over PIC32 I2C2, and its
`i2cCtrl::SwBusReset()` cleared a stuck bus. The register choices of the first — a
six-byte burst for a sample, the measure bit written last — were right. Checked
against the ADXL345 datasheet, Rev. G, and UM10204, 3.1.16.

* **Configuration could fail without a word.** `StartConfig()` ignored every
  write's result. `Configure()` reports a failed transfer and reads the
  registers back.
* **A timed-out read kept the old sample**, as if new. A failed read is an empty
  `std::optional`.
* **The timeouts broke at the counter wrap**: `_CP0_GET_COUNT() > start + 1800000`
  was true at once when the sum overflowed. The driver has no loops.
* **A chip left measuring was reconfigured so**; the datasheet recommends standby.
  `Configure()` enters it first, and turns the interrupt outputs off until they
  are routed.
* **The bus clear reported success when SDA never let go** — `i` ends at 17
  against a `TIMEOUT` of 16 — and **sent no STOP**: its pin helpers are named the
  other way round, so its closing "STOP" ran with SCL held low. `RecoverI2cBus()`
  in i2c-bus 0.1.1 sends up to nine clocks, each ending in an attempt at a STOP,
  as Linux does, and reports whether SDA or SCL stays held.

External reviews of both found a second `Configure()` rerouting a pending
interrupt, a read-back that cleared INT_SOURCE, SDA moving right after SCL fell,
and a first STOP without its setup time; all fixed. Neither has run on hardware.

## And ads129x, from a159-bpu-firmware

a159's `sensor_manager/ads1298` read two ADS1298 in a daisy chain on one chip
select. Its register values and its bit recombination were taken as the
reference; its transport timing and error handling were not. Checked against
SBAS459K (Rev. K).

* **Configuration could fail without a word**: `StartConfig()` ignored every
  write and read nothing back. `Configure()` reports a failed transfer and
  compares the registers.
* **The status word was never checked**, so a second chip missing from the chain,
  or a frame out of step, read as samples. Each device's 1100b header is checked.
* **The START pin was raised along with the START command**, which the datasheet
  asks to keep low; the driver never touches the pin.
* `SendStartCmd()` reported only its second transfer.
* The extra SCLK of the chain was bit-banged with the SPI peripheral off mid-frame,
  and the clock changed by writing SPI2BRG in the driver. The frame is now one
  transfer, realigned across the don't-care bit, and slow commands and fast
  frames take two `SpiDevice`s.
* CS rose 1 µs after the last bit of a register write, where the datasheet asks
  4 tCLK, about 1.95 µs: not guaranteed there, the adapter's now.

The driver covers the family, 4 to 8 channels, one chip or a chain. An external
review found `Reset()` without SDATAC in RDATAC mode, absent channels' registers
of an ADS1294 or ADS1296 written and read, and the DRDY and shared-clock
preconditions missing; all fixed. It has not run on hardware.
