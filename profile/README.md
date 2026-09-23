# integra-lib

Architecture-independent C++20 components shared between firmware projects.
One repository per component: a project adds only what it uses and moves each
component's version on its own.

Header-only except `transaction-engine`, no exceptions, no RTTI.

## Components

### Data structures

| Repository | Provides | Reach for it when |
|---|---|---|
| [ring-buffer](https://github.com/integra-lib/ring-buffer) | `RingBuffer<T, SIZE>` | a fixed-size queue with no dynamic allocation |
| [const-map](https://github.com/integra-lib/const-map) | `ConstMap<Key, Value, SIZE>` | a small lookup table known at compile time |
| [dedup-cache](https://github.com/integra-lib/dedup-cache) | `DedupCache<CAPACITY>` | telling a retransmission from a new message |

### Utilities

| Repository | Provides | Reach for it when |
|---|---|---|
| [function](https://github.com/integra-lib/function) | `Function<R(Args...)>` | holding a callback where `std::function` is unavailable or too much |
| [bit-ops](https://github.com/integra-lib/bit-ops) | `AssembleBytes`, `GetByteByIndex` | taking integers apart and back together in a protocol |
| [enum-utils](https://github.com/integra-lib/enum-utils) | `EnumValue` | an enumerator's numeric value |
| [hex-string](https://github.com/integra-lib/hex-string) | `HexToBytes`, `BytesToHex`, `BytesToHexReversed` | hex text to bytes and back, without allocating |

### Algorithms

| Repository | Provides | Reach for it when |
|---|---|---|
| [ema-filter](https://github.com/integra-lib/ema-filter) | `EmaFilter` | smoothing a sensor reading |
| [crc](https://github.com/integra-lib/crc) | `Crc8Nrsc5`, `Crc16Ccitt`, `Crc32IsoHdlc`, `Crc32Stream` | checksums: sensors, protocols, verifying a firmware image |

### Execution

| Repository | Provides | Reach for it when |
|---|---|---|
| [work-queue](https://github.com/integra-lib/work-queue) | `IWorkQueue` | an interface for deferred execution |
| [periodic-clock](https://github.com/integra-lib/periodic-clock) | `IPeriodicClock` | an interface for a monotonic clock and a periodic tick |

### Events

| Repository | Provides | Reach for it when |
|---|---|---|
| [event-manager](https://github.com/integra-lib/event-manager) | `EventManager<Payload, Queue>` | publish/subscribe over a queue you supply |
| [button-event](https://github.com/integra-lib/button-event) | `ButtonEventCore` | debouncing a button and telling a short press from a long one |

### Persistence

| Repository | Provides | Reach for it when |
|---|---|---|
| [settings-record](https://github.com/integra-lib/settings-record) | `WriteSettingsRecord`, `ReadSettingsRecord` | settings in flash that read back only if they verify |

### Communication

| Repository | Provides | Reach for it when |
|---|---|---|
| [transaction-engine](https://github.com/integra-lib/transaction-engine) | frame format, transport contract, sender and receiver | acknowledged command exchange over an unreliable link |

Shared tooling lives in [ci-shared](https://github.com/integra-lib/ci-shared) — the
pipeline template and the style configs, used by the components and never by a
consumer.

## Using a component

```bash
git submodule add git@github.com:integra-lib/crc.git external/integra/crc
```

```cmake
add_subdirectory(external/integra/crc)
target_link_libraries(app PRIVATE Integra::crc)
```

```cpp
#include <integra/crc.hpp>
```

Each component carries its own include directory, so a header stays unreachable
until its component is linked: a forgotten dependency is a compile error rather
than a build that happens to work.

Two components have dependencies: `transaction-engine` needs `crc`, `bit-ops` and
`dedup-cache`, and `settings-record` needs `crc`. They are added next to the component
that needs them rather than inside it, so a project can never end up with two copies
of the same component. A missing or out-of-range dependency stops the CMake configure
with a message naming the version found.

## Where this is going

GitHub is a staging ground. The target is the GitLab group
`internal-projects/integra-lib`; the move changes only the URL in a consumer's
`.gitmodules`.

`docs/integra-lib.confluence.txt` in this repository is the same description in
Confluence wiki markup, ready to paste.
