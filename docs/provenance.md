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
