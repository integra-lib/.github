# hwlib

Architecture-independent C++20 components shared between firmware projects,
published under the `integra-lib` GitHub organization.
One repository per component: a project adds only what it uses and moves each
component's version on its own.

Header-only except `transaction-engine`, no exceptions, no RTTI. One component,
`boot-slots`, is C (C11) with a C API instead, because bootloaders are often C.

## Components

### Data structures

Namespace and include prefix: `hwlib::data_structures`, `hwlib/data_structures/`.

| Repository | Provides | Reach for it when |
|---|---|---|
| [ring-buffer](https://github.com/integra-lib/ring-buffer) | `RingBuffer<T, SIZE>` | a fixed-size queue with no dynamic allocation |
| [const-map](https://github.com/integra-lib/const-map) | `ConstMap<Key, Value, SIZE>` | a small lookup table known at compile time |
| [dedup-cache](https://github.com/integra-lib/dedup-cache) | `DedupCache<CAPACITY>` | telling a retransmission from a new message |

### Utilities

Namespace and include prefix: `hwlib::utilities`, `hwlib/utilities/`.

| Repository | Provides | Reach for it when |
|---|---|---|
| [function](https://github.com/integra-lib/function) | `Function<R(Args...)>` | holding a callback where `std::function` is unavailable or too much |
| [bit-ops](https://github.com/integra-lib/bit-ops) | `AssembleBytes`, `GetByteByIndex` | taking integers apart and back together in a protocol |
| [enum-utils](https://github.com/integra-lib/enum-utils) | `EnumValue` | an enumerator's numeric value |
| [hex-string](https://github.com/integra-lib/hex-string) | `HexToBytes`, `BytesToHex`, `BytesToHexReversed` | hex text to bytes and back, without allocating |
| [byte-codec](https://github.com/integra-lib/byte-codec) | `Store`, `Load`, `ByteReader` | encoding integers, enums and floats as big- or little-endian bytes |

### Algorithms

Namespace and include prefix: `hwlib::algorithms`, `hwlib/algorithms/`.

| Repository | Provides | Reach for it when |
|---|---|---|
| [ema-filter](https://github.com/integra-lib/ema-filter) | `EmaFilter` | smoothing a sensor reading |
| [crc](https://github.com/integra-lib/crc) | `Crc8Nrsc5`, `Crc16Ccitt`, `Crc32IsoHdlc`, `Crc32Stream` | checksums: sensors, protocols, verifying a firmware image |
| [debouncer](https://github.com/integra-lib/debouncer) | `Debouncer<INC, DEC>` | confirming a condition over several samples before acting on it |
| [filters](https://github.com/integra-lib/filters) | `MedianFilter`, `HysteresisFilter`, `MovingAverage` | cleaning up a sensor reading: spikes, chatter at a threshold, noise |

### Execution

Namespace and include prefix: `hwlib::execution`, `hwlib/execution/`.

| Repository | Provides | Reach for it when |
|---|---|---|
| [work-queue](https://github.com/integra-lib/work-queue) | `IWorkQueue` | an interface for deferred execution |
| [periodic-clock](https://github.com/integra-lib/periodic-clock) | `IPeriodicClock` | an interface for a monotonic clock and a periodic tick |
| [worker](https://github.com/integra-lib/worker) | `Worker<Mutex>` | posting work from any context and running it all on one |

### Events

Namespace and include prefix: `hwlib::events`, `hwlib/events/`.

| Repository | Provides | Reach for it when |
|---|---|---|
| [event-manager](https://github.com/integra-lib/event-manager) | `EventManager<Payload, Queue>` | publish/subscribe over a queue you supply |
| [button-event](https://github.com/integra-lib/button-event) | `ButtonEventCore` | debouncing a button and telling a short press from a long one |

### Persistence

Namespace and include prefix: `hwlib::persistence`, `hwlib/persistence/`.

| Repository | Provides | Reach for it when |
|---|---|---|
| [settings-record](https://github.com/integra-lib/settings-record) | `WriteSettingsRecord`, `ReadSettingsRecord` | settings in flash that read back only if they verify |
| [littlefs-cpp](https://github.com/integra-lib/littlefs-cpp) | `Littlefs<Device>`, `Littlefs::File` | a filesystem on flash through littlefs, without allocating |
| [boot-slots](https://github.com/integra-lib/boot-slots) | `hwlib_boot_prepare`, `hwlib_boot_journal_load`/`_save`, `hwlib_image_check` (C) | an A/B bootloader: which image to start, trial boot and rollback, a power-loss-safe state journal, image header and CRC check |

### Communication

Namespace and include prefix: `hwlib::communication`, `hwlib/communication/`.

| Repository | Provides | Reach for it when |
|---|---|---|
| [transaction-engine](https://github.com/integra-lib/transaction-engine) | frame format, transport contract, sender and receiver | acknowledged command exchange over an unreliable link |
| [mqtt-topic](https://github.com/integra-lib/mqtt-topic) | `MatchTopic` | routing an MQTT message to the subscription whose filter it matches |
| [zigbee-app](https://github.com/integra-lib/zigbee-app) | `ZigbeeApp`, `DecideZigbeeSignal` | a Zigbee device on ZBOSS (nRF Connect SDK): endpoints, join and leave, End Device sleep and polling; the signal policy is host-testable |

Shared tooling lives in [ci-shared](https://github.com/integra-lib/ci-shared) — the
pipeline template and the style configs, used by the components and never by a
consumer.

## Using a component

```bash
git submodule add git@github.com:integra-lib/crc.git external/hwlib/crc
```

```cmake
add_subdirectory(external/hwlib/crc)
target_link_libraries(app PRIVATE Hwlib::crc)
```

```cpp
#include <hwlib/algorithms/crc.hpp>
```

Each component carries its own include directory, so a header stays unreachable
until its component is linked: a forgotten dependency is a compile error rather
than a build that happens to work.

Three components have dependencies: `transaction-engine` needs `crc`, `bit-ops` and
`dedup-cache`, `settings-record` needs `crc`, and `littlefs-cpp` needs littlefs itself,
a third-party library the project provides a CMake target for. They are added next to the component
that needs them rather than inside it, so a project can never end up with two copies
of the same component. A missing or out-of-range dependency stops the CMake configure
with a message naming the version found.

## Where this is going

GitHub repositories remain under `integra-lib`. Their GitLab counterparts belong
to subgroup `internal-projects/a000-hwlib`, organized by the section names above.
Changing hosts changes only the URL in a consumer's `.gitmodules`.

`docs/integra-lib.confluence.txt` in this repository is the same description in
Confluence wiki markup, ready to paste.
