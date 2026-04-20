# Modules (C++20/23)

## Table of Contents
1. Why Modules
2. Module Unit Types
3. File Extensions & Conventions
4. Module Structure Examples
5. Partitions
6. Importing the Standard Library
7. Global Module Fragment
8. Build System Integration
9. Migration Strategy

---

## 1. Why Modules

Modules solve long-standing problems with `#include`:

- **Compilation speed**: a module interface is compiled once into a Binary Module
  Interface (BMI). Importers read the BMI instead of re-parsing headers.
  Benchmarks show 2-10× faster builds for large projects.
- **No macro leakage**: macros defined inside a module do not escape to importers.
- **No include-order dependencies**: modules have well-defined semantics
  regardless of import order.
- **Better encapsulation**: only explicitly `export`ed declarations are visible.
  Internal helpers stay private.

## 2. Module Unit Types

| Unit type                      | Keyword in declaration         | File convention   |
|-------------------------------|-------------------------------|-------------------|
| Primary module interface      | `export module M;`            | `M.cppm` / `M.ixx` |
| Module implementation unit    | `module M;`                   | `M.cpp`           |
| Module interface partition    | `export module M:part;`       | `M-part.cppm`     |
| Module implementation partition | `module M:part;`            | `M-part.cpp`      |

Every named module must have exactly one primary module interface unit.

## 3. File Extensions & Conventions

- **Clang**: recommends `.cppm` for module interface files.
- **MSVC**: recommends `.ixx` (will also accept `.cppm`).
- **GCC**: no preference, but `.cppm` is becoming the de facto standard.

Use a consistent suffix across the project. `.cppm` is the most widely
recognized convention.

Implementation files (non-interface) use the standard `.cpp` extension.

## 4. Module Structure Examples

### Minimal: single-file module
```cpp
// math.cppm
export module math;

export int add(int a, int b) { return a + b; }
export int multiply(int a, int b) { return a * b; }
```

```cpp
// main.cpp
import math;
int main() { return add(2, 3); }
```

### Separated interface and implementation
```cpp
// math.cppm — interface
export module math;

export int add(int a, int b);
export int multiply(int a, int b);
```

```cpp
// math.cpp — implementation
module math;

int add(int a, int b) { return a + b; }
int multiply(int a, int b) { return a * b; }
```

Note: implementation units implicitly import the primary interface unit.

### Module-internal helpers
Non-exported declarations are visible only within the module:

```cpp
// math.cppm
export module math;

// Internal — not visible to importers
int clamp_impl(int x, int lo, int hi) {
    return x < lo ? lo : (x > hi ? hi : x);
}

// Exported — visible to importers
export int safe_add(int a, int b) {
    return clamp_impl(a + b, INT_MIN, INT_MAX);
}
```

## 5. Partitions

For large modules, split into partitions. Partitions are visible only within
their parent module — external code imports the module, not its partitions.

```cpp
// mylib-core.cppm — interface partition
export module mylib:core;

export class Engine {
public:
    void start();
    void stop();
};
```

```cpp
// mylib-utils.cppm — interface partition
export module mylib:utils;

export std::string format_status(int code);
```

```cpp
// mylib.cppm — primary interface, re-exports partitions
export module mylib;

export import :core;
export import :utils;
```

```cpp
// mylib-core.cpp — implementation partition
module mylib:core;

import :utils;  // can import sibling partitions

void Engine::start() { /* ... */ }
void Engine::stop() { /* ... */ }
```

**Best practice**: use implementation partition units (non-exported) for internal
implementation files. This keeps BMIs small and reduces cascading recompilation.

## 6. Importing the Standard Library

C++23 provides `import std;` to import the entire standard library as a module:

```cpp
import std;

int main() {
    auto v = std::vector{1, 2, 3, 4, 5};
    std::println("sum = {}", std::ranges::fold_left(v, 0, std::plus{}));
}
```

This is typically faster than including individual headers because the standard
library BMI is precompiled once. There is also `import std.compat;` which
additionally provides C library names in the global namespace.

For C++20 without `import std;`, use `import <header>;` for header units:
```cpp
import <vector>;
import <string>;
import <iostream>;
```

## 7. Global Module Fragment

When a module needs to include traditional headers (e.g., system headers or
C libraries that cannot be imported), use the global module fragment:

```cpp
// network.cppm
module;                        // begins global module fragment
#include <sys/socket.h>        // C headers OK here
#include <netinet/in.h>
export module network;         // ends global module fragment

export class TcpSocket {
    int fd_ = -1;
public:
    void connect(const char* host, int port);
    // ...
};
```

The global module fragment (`module;` before the module declaration) is the only
place where `#include` should appear in a module unit. Macros from these headers
do not leak to importers.

## 8. Build System Integration

### CMake (3.28+)

CMake 3.28 has experimental C++20 module support:

```cmake
cmake_minimum_required(VERSION 3.28)
project(myproject CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(mylib)
target_sources(mylib
    PUBLIC FILE_SET CXX_MODULES FILES
        src/mylib.cppm
        src/mylib-core.cppm
        src/mylib-utils.cppm
    PRIVATE
        src/mylib-core.cpp
)

add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE mylib)
```

The `FILE_SET CXX_MODULES` tells CMake to scan these files for module
dependencies and compile them in the correct order.

### Manual Clang compilation
```bash
# 1. Compile module interface → BMI (.pcm)
clang++ -std=c++23 math.cppm --precompile -o math.pcm

# 2. Compile module interface → object file
clang++ -std=c++23 math.pcm -c -o math.o

# 3. Compile consumer with BMI
clang++ -std=c++23 -fprebuilt-module-path=. main.cpp math.o -o main
```

### Manual GCC compilation
```bash
gcc -std=c++23 -fmodules -c math.cppm   # produces math.o + gcm.cache/
gcc -std=c++23 -fmodules main.cpp math.o -o main
```

## 9. Migration Strategy

Migrating an existing header-based project to modules is non-trivial. Recommended
approach:

1. **Start with new code.** Write new components as modules.
2. **Wrap existing headers.** Create a module that includes your header in the
   global module fragment and re-exports the declarations.
3. **Migrate leaf libraries first.** Dependencies must be modularized before
   dependents.
4. **Use `import std;`** to immediately benefit from faster standard library
   compilation.
5. **Keep headers as a fallback.** Libraries can offer both `#include` and
   `import` paths during transition.

Note: once a project uses modules, its downstream dependents typically also need
module support. For shared libraries, consider providing both header and module
interfaces during the transition period.
