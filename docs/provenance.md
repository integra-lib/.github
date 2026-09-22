# Where the components came from

The components were consolidated in September 2026 out of copies that had drifted
across firmware projects. The per-component repositories start from a clean slate
by decision, so this file keeps the part of that history worth keeping: what the
code came from, and what was deliberately changed on the way.

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
  `settings-record.cpp`. It now calls `Integra::crc`. A test pins the result against
  both the catalogue check value for `"123456789"` and the original's own loop,
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

## Known and unverified, second round

* `event-manager` and `button-event` hold their handlers in `std::function`, which
  allocates for a large enough capture. Registration happens once at startup, but a
  context that forbids the heap outright has to know.
* `button-event`'s one-context rule is a contract, not a compiler error. a174-hardware
  already honoured it; a new adapter that calls the core straight from an ISR would
  compile.
* Nothing here has run on a device yet. The four components are verified by their
  own test suites on the host, under gcc and clang.

