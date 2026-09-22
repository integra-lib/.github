# integra-lib

Architecture-independent C++20 components shared between firmware projects.
One repository per component: a project adds only what it uses and moves each
component's version on its own.

Header-only except `transaction-engine`, no exceptions, no RTTI.

| Repository | Provides | Reach for it when |
|---|---|---|
| [ring-buffer](https://github.com/integra-lib/ring-buffer) | `RingBuffer<T, SIZE>` | a fixed-size queue with no dynamic allocation |
| [function](https://github.com/integra-lib/function) | `Function<R(Args...)>` | holding a callback where `std::function` is unavailable or too much |
| [bit-ops](https://github.com/integra-lib/bit-ops) | `AssembleBytes`, `GetByteByIndex` | taking integers apart and back together in a protocol |
| [enum-utils](https://github.com/integra-lib/enum-utils) | `EnumValue` | an enumerator's numeric value |
| [const-map](https://github.com/integra-lib/const-map) | `ConstMap<Key, Value, SIZE>` | a small lookup table known at compile time |
| [ema-filter](https://github.com/integra-lib/ema-filter) | `EmaFilter` | smoothing a sensor reading |
| [crc](https://github.com/integra-lib/crc) | `Crc8Nrsc5`, `Crc16Ccitt`, `Crc32IsoHdlc`, `Crc32Stream` | checksums: sensors, protocols, verifying a firmware image |
| [dedup-cache](https://github.com/integra-lib/dedup-cache) | `DedupCache<CAPACITY>` | telling a retransmission from a new message |
| [work-queue](https://github.com/integra-lib/work-queue) | `IWorkQueue` | an interface for deferred execution |
| [periodic-clock](https://github.com/integra-lib/periodic-clock) | `IPeriodicClock` | an interface for a monotonic clock and a periodic tick |
| [transaction-engine](https://github.com/integra-lib/transaction-engine) | frame format, transport contract, sender and receiver | acknowledged command exchange over an unreliable link |
| [ci-shared](https://github.com/integra-lib/ci-shared) | pipeline template and style configs | shared by the components, not by consumers |

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

`transaction-engine` is the one component with dependencies — `crc`, `bit-ops`
and `dedup-cache`, added next to it rather than inside it, so a project can never
end up with two copies of the same component. A missing or out-of-range
dependency stops the CMake configure with a message naming the version found.

## Where this is going

GitHub is a staging ground. The target is the GitLab group
`internal-projects/integra-lib`; the move changes only the URL in a consumer's
`.gitmodules`.

`docs/integra-lib.confluence.txt` in this repository is the same description in
Confluence wiki markup, ready to paste.
