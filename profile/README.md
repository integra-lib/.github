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
| [battery-monitor](https://github.com/integra-lib/battery-monitor) | `BatteryMonitor<Gauge>`, `CoulombGauge`, `SocFromOcv` | a battery's state of charge from a coulomb counter such as the STC3100, resuming across MCU resets |
| [bresenham-modulator](https://github.com/integra-lib/bresenham-modulator) | `BresenhamModulator` | power through a solid-state relay in whole mains half-cycles, spread evenly: the level set from a task, stepped from an interrupt |

### Execution

Namespace and include prefix: `hwlib::execution`, `hwlib/execution/`.

| Repository | Provides | Reach for it when |
|---|---|---|
| [work-queue](https://github.com/integra-lib/work-queue) | `IWorkQueue` | an interface for deferred execution |
| [periodic-clock](https://github.com/integra-lib/periodic-clock) | `IPeriodicClock` | an interface for a monotonic clock and a periodic tick |
| [worker](https://github.com/integra-lib/worker) | `Worker<Mutex>` | posting work from any context and running it all on one |
| [coro](https://github.com/integra-lib/coro) | `Task`, `SyncWait`, `WhenAllReady`, `AsyncScope`, `AsyncGenerator`, `IoService` | C++20 coroutines on a microcontroller: tasks, waiting for several at once, generators, a scheduler with timers on its own thread |

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
| [can-filter-codec](https://github.com/integra-lib/can-filter-codec) | `EncodeCanFilters`, `DecodeCanFilters` | setting the acceptance filters of a CAN controller from a phone or a PC: the filter list as a few versioned TLV bytes |
| [mqtt-topic](https://github.com/integra-lib/mqtt-topic) | `MatchTopic` | routing an MQTT message to the subscription whose filter it matches |
| [zigbee-app](https://github.com/integra-lib/zigbee-app) | `ZigbeeApp`, `DecideZigbeeSignal` | a Zigbee device on ZBOSS (nRF Connect SDK): endpoints, join and leave, End Device sleep and polling; the signal policy is host-testable |

### Drivers

Namespace and include prefix: `hwlib::drivers`, `hwlib/drivers/`. A driver takes
the bus as a template parameter — any type with the members the driver's concept
names — so it carries no platform code; the application adapts its own HAL.

| Repository | Provides | Reach for it when |
|---|---|---|
| [i2c-bus](https://github.com/integra-lib/i2c-bus) | `I2cWrite`, `I2cRead`, `I2cWriteRead`, `I2cBus`, `FakeI2cBus` | always, next to an I2C driver: the bus concepts it takes, and a fake bus to test it with |
| [stc3100](https://github.com/integra-lib/stc3100) | `Stc3100<Bus>` | an ST STC3100 battery monitor: coulomb counter, battery voltage, temperature, RAM kept across MCU resets |
| [sht40](https://github.com/integra-lib/sht40) | `Sht40<Bus>` | a Sensirion SHT40 humidity and temperature sensor, read without waiting inside the driver |
| [spi-bus](https://github.com/integra-lib/spi-bus) | `SpiDevice`, `FakeSpiDevice` | always, next to an SPI driver: the device concept it takes, and a fake device to test it with |
| [mfrc522](https://github.com/integra-lib/mfrc522) | `Mfrc522<Spi>`, `Iso14443aActivation` | an NXP MFRC522 NFC reader: the UID of an ISO/IEC 14443 A card — MIFARE, NTAG — without waiting inside the driver |
| [ad7797](https://github.com/integra-lib/ad7797) | `Ad7797<Spi>` | an Analog Devices AD7797 24-bit bridge converter: a load cell or strain gauge in µV, overrange as an error |
| [max31856](https://github.com/integra-lib/max31856) | `Max31856<Spi>` | a MAX31856 thermocouple converter: types B to T in m°C, with the cold junction and the faults of every reading |
| [linear-actuator](https://github.com/integra-lib/linear-actuator) | `LinearActuator<Bridge>`, `HBridge` | a DC-motor linear actuator with position and current feedback: finds its ends, moves to positions, stops on overload or stall |

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
