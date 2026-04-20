---
name: modern-cpp
description: >
  Best practices and idioms for writing modern C++ (C++20, C++23, C++26).
  Use this skill whenever the user asks for help writing C++ code, reviewing C++ code,
  designing C++ APIs, or converting legacy C/C++ to modern style. Also trigger when the
  user mentions smart pointers, RAII, modules (.cppm/.ixx), concepts, ranges, coroutines,
  constexpr, consteval, constinit, designated initializers, deducing this, std::expected,
  std::optional, std::format, std::print, reflection, contracts, move semantics,
  value categories, templates, or any C++20/23/26 feature. Trigger liberally for
  anything involving C++ — even casual questions like "how should I structure this
  C++ class?" or "is there a better way to write this?" warrant consulting this skill.
---

# Modern C++ Programming Skill

This skill guides Claude to produce idiomatic, safe, and performant C++ targeting
C++20 as the baseline, with C++23 and C++26 features used when appropriate. The
philosophy: **write code that is correct by construction** — leverage the type system,
RAII, and compile-time checks so bugs are caught before runtime.

## Guiding Principles

1. **Ownership is explicit.** Every resource has a clear owner. Express ownership
   through value semantics, `std::unique_ptr`, or — only when truly shared —
   `std::shared_ptr`. Raw pointers mean "non-owning observer" and nothing else.

2. **Prefer value semantics.** Pass and return objects by value when they are
   cheap to copy or when move semantics make it efficient. Avoid heap allocation
   when the stack suffices.

3. **Compile-time over runtime.** Prefer `constexpr`/`consteval` computation,
   concepts for type constraints, and `static_assert` for invariants. Push errors
   to compile time wherever possible.

4. **No naked `new`/`delete`.** Use `std::make_unique`, `std::make_shared`, or
   containers. If you need custom allocation, wrap it in an RAII type.

5. **Minimal includes, prefer modules.** For new C++20+ projects, prefer
   `import std;` or granular module imports over `#include` when the build system
   supports it.

6. **Use the standard library first.** Reach for `<algorithm>`, `<ranges>`,
   `<format>`, `<expected>`, `<optional>`, `<variant>`, `<span>`, etc. before
   writing custom utilities.

---

## Quick Reference: What to Use When

| Legacy pattern                     | Modern replacement                              | Standard |
|------------------------------------|------------------------------------------------|----------|
| `new T` / `delete`                 | `std::make_unique<T>()` / `std::make_shared<T>()` | C++14+   |
| `NULL` / `0`                       | `nullptr`                                       | C++11    |
| `#include <header>`                | `import std;` or `import <header>;`             | C++20/23 |
| `typedef`                          | `using Alias = Type;`                           | C++11    |
| Error codes / exceptions only      | `std::expected<T, E>`                           | C++23    |
| Nullable pointer for optional      | `std::optional<T>`                              | C++17    |
| `printf` / `iostream` formatting   | `std::format` / `std::print`                    | C++20/23 |
| SFINAE for template constraints    | `concept` + `requires`                          | C++20    |
| Macro constants                    | `constexpr` / `consteval` / `constinit`         | C++11-20 |
| `const_cast` overload duplication  | Deducing `this`                                 | C++23    |
| CRTP boilerplate                   | Deducing `this`                                 | C++23    |
| Positional aggregate init          | Designated initializers                         | C++20    |
| Raw loops over containers          | Ranges + views                                  | C++20    |
| Manual enum-to-string              | Static reflection (`^`, `[:..:]`)               | C++26    |

---

## Core Topics

For detailed guidance on each area, read the corresponding reference file in
`references/`. Below is a summary of when and how to apply each topic.

### 1. RAII & Smart Pointers

Read: `references/raii-and-ownership.md`

RAII ties resource lifetime to object lifetime. Every resource — memory, files,
locks, sockets — should be managed by an RAII wrapper. The standard library
provides the key RAII types:

- **`std::unique_ptr<T>`** — sole ownership, zero overhead. Default choice.
  Create with `std::make_unique<T>(args...)`. Transfer with `std::move`.
- **`std::shared_ptr<T>`** — shared ownership via reference counting. Use only
  when multiple owners genuinely need independent lifetimes. Create with
  `std::make_shared<T>(args...)`.
- **`std::weak_ptr<T>`** — non-owning observer of a `shared_ptr`. Use to break
  cycles (e.g., parent↔child in a graph).
- **Custom RAII wrappers** — for non-memory resources, use `unique_ptr` with a
  custom deleter, or write a small class following the Rule of Zero/Five.

**Rule of Zero**: if your class only holds RAII members (smart pointers,
containers, `std::string`, etc.), don't write a destructor, copy/move
constructors, or assignment operators — the compiler-generated defaults are
correct.

**Rule of Five**: if you must manage a resource manually, implement *all five*:
destructor, copy constructor, copy assignment, move constructor, move assignment.

**When raw pointers are OK**: as non-owning observers passed to functions that
don't store the pointer. Use `T*` for nullable observers, `T&` for non-nullable.
`std::span<T>` and `std::string_view` are preferred for ranges of data.

### 2. Concepts & Constrained Templates (C++20)

Read: `references/concepts-and-templates.md`

Concepts replace SFINAE with readable, composable constraints:

```cpp
template <std::integral T>
T gcd(T a, T b) { return b == 0 ? a : gcd(b, a % b); }
```

Define custom concepts when standard ones don't fit:

```cpp
template <typename T>
concept Serializable = requires(T t, std::ostream& os) {
    { t.serialize(os) } -> std::same_as<void>;
};
```

Use `requires` clauses for ad-hoc constraints on functions or classes. Prefer
named concepts for reusability and error message clarity.

### 3. Ranges & Views (C++20/23)

Read: `references/ranges-and-views.md`

Ranges enable pipeline-style data processing with lazy evaluation:

```cpp
auto results = data
    | std::views::filter([](auto& x) { return x.active; })
    | std::views::transform(&Item::name)
    | std::views::take(10);
```

C++23 adds `std::views::chunk`, `std::views::slide`, `std::views::zip`,
`std::views::enumerate`, and fold algorithms. Prefer range-based algorithms
(`std::ranges::sort`, `std::ranges::find_if`) over their `<algorithm>`
predecessors — they accept ranges directly and offer projection parameters.

### 4. Modules (C++20/23)

Read: `references/modules.md`

Modules replace `#include` with `import`, eliminating redundant parsing and
macro leakage. Key file conventions:

- **`.cppm`** (Clang) / **`.ixx`** (MSVC) — module interface unit
- **`.cpp`** — module implementation unit

A minimal module:

```cpp
// math.cppm — module interface
export module math;

export int add(int a, int b) { return a + b; }
```

```cpp
// main.cpp — consumer
import math;
int main() { return add(1, 2); }
```

Use `import std;` (C++23) to import the entire standard library as a single
module — this is typically faster than `#include` chains.

Partition modules for large libraries:

```cpp
// mylib-core.cppm
export module mylib:core;
export class Engine { /* ... */ };

// mylib.cppm — primary interface
export module mylib;
export import :core;
```

### 5. Designated Initializers (C++20)

Use designated initializers for aggregate types to improve readability and
resilience to member reordering:

```cpp
struct Config {
    int width  = 1920;
    int height = 1080;
    bool fullscreen = false;
    int samples = 4;
};

auto cfg = Config{ .width = 2560, .height = 1440, .fullscreen = true };
// .samples keeps its default value of 4
```

Rules to remember:
- Designators must appear in declaration order (unlike C99).
- Cannot mix designated and positional initializers.
- Unlisted members use their default member initializers (or zero-init).

This pattern works especially well as a substitute for named function parameters
by passing a config struct:

```cpp
Texture create_texture(TextureDesc desc);
auto tex = create_texture({ .format = Format::RGBA8, .width = 512, .height = 512 });
```

### 6. Deducing `this` / Explicit Object Parameter (C++23)

Read: `references/deducing-this.md`

C++23 lets you make the implicit `this` parameter explicit, eliminating the need
to duplicate member functions for different cv/ref qualifiers:

```cpp
struct Widget {
    // One function handles &, const&, and && — no duplication
    template <class Self>
    decltype(auto) name(this Self&& self) {
        return std::forward<Self>(self).name_;
    }
private:
    std::string name_;
};
```

**Return-type rule of thumb**: use `decltype(auto)` (not `auto&`) and always
write `std::forward<Self>(self).member` in the return expression. `auto&`
collapses to an lvalue reference and can't carry an rvalue-qualified call
through; an unforwarded `self.member` is always an lvalue regardless of how
`Self` was deduced. Skip the forward only when there's no value category to
propagate (e.g. returning a `std::span` built from pointer + size).

For chained calls to another deducing-`this` template member, use
`.template`: `std::forward<Self>(self).template inner<T>().method()` — and
forward the *outermost* object, not the intermediate call.

Key use cases:
- **De-quadruplication**: one templated overload replaces up to four (const/non-const × lvalue/rvalue).
- **Recursive lambdas**: `auto fact = [](this auto self, int n) -> int { return n < 2 ? 1 : n * self(n-1); };`
- **Simplified CRTP**: base class no longer needs to be templated on the derived type.

Restrictions: explicit object parameters cannot be `virtual`, `static`, or have
cv/ref qualifiers on the function itself. See
`references/deducing-this.md` section 4 ("Return Types and Forwarding") for
the detailed rules and a pre-commit checklist.

### 7. `constexpr`, `consteval`, `constinit` (C++11–C++23)

Read: `references/compile-time-programming.md`

The constexpr family pushes computation to compile time:

- **`constexpr`** — *may* run at compile time if all inputs are constant
  expressions. In C++20 expanded to virtual functions, `try`/`catch`,
  `dynamic_cast`. In C++23 further expanded (e.g., `constexpr std::unique_ptr`,
  `constexpr std::optional`).
- **`consteval`** — *must* run at compile time (immediate function). Use for
  functions that should never appear in runtime code (lookup tables, hashing, etc.).
- **`constinit`** — ensures a variable is initialized at compile time but does
  not make it `const`. Solves the Static Initialization Order Fiasco.
- **`if consteval`** (C++23) — branch based on whether code is being
  constant-evaluated, replacing `std::is_constant_evaluated()`.

### 8. Error Handling: `std::expected` & `std::optional` (C++17/23)

Prefer `std::expected<T, E>` (C++23) for operations that can fail with a
specific error type. It makes the error path explicit in the type system:

```cpp
std::expected<Config, ParseError> parse_config(std::string_view path);
```

Use `std::optional<T>` when absence of a value is a normal (non-error) case.
Both support monadic operations (`.and_then`, `.transform`, `.or_else`) for
clean chaining in C++23.

Reserve exceptions for truly exceptional situations (I/O failures, out-of-memory,
programming errors). Avoid exceptions in hot paths.

### 9. Modern Formatting: `std::format` & `std::print` (C++20/23)

Replace `printf` and `iostream` formatting with `std::format` / `std::print`:

```cpp
std::println("Hello, {}! You scored {:.1f}%", name, score);
```

`std::format` returns a `std::string`; `std::print`/`std::println` write
directly to stdout (C++23), avoiding temporary allocations. Both use Python-like
format strings and are type-safe.

### 10. C++26 Preview: Reflection & Contracts

Read: `references/cpp26-preview.md`

**Static reflection** (the biggest C++26 feature) enables compile-time
introspection and code generation:
- `^T` — the reflection operator, yields a `std::meta::info` value
- `[:refl:]` — the splice operator, turns reflection back into code
- `std::meta::members_of`, `std::meta::name_of`, etc. — query reflected entities

**Contracts** add `[[pre:]]`, `[[post:]]`, and `contract_assert` for
design-by-contract:

```cpp
int safe_divide(int a, int b)
    [[pre: b != 0]]
    [[post r: r * b == a]]  // r names the return value
{
    return a / b;
}
```

These are draft features — compiler support is emerging. Use them in new projects
when your toolchain supports them.

---

## Code Style Conventions

- **Naming**: `snake_case` for functions, variables, namespaces; `PascalCase`
  for types and concepts; `UPPER_SNAKE` only for macros (which should be rare).
  Follow the project's existing style when contributing.
- **`auto`**: use when the type is obvious from context or unimportant; spell
  out the type when it aids readability.
- **Braces**: always use braces for `if`/`for`/`while` bodies, even single lines.
- **`const` by default**: make variables, parameters, and member functions
  `const` unless mutation is needed.
- **Namespaces**: wrap library code in namespaces. Never `using namespace std;`
  in headers. In `.cpp` files, prefer targeted `using` declarations.
- **Includes / imports**: group and sort. Prefer `import` where supported.

---

## Tooling Recommendations

- **Compiler**: GCC 13+, Clang 17+, or MSVC 2022 17.6+ for solid C++23 support.
  For C++26 reflection experiments, use the Bloomberg Clang/P2996 fork or
  latest GCC trunk.
- **Build system**: CMake 3.28+ with `set(CMAKE_CXX_STANDARD 23)`. For modules,
  CMake 3.28+ has experimental support; Clang's `--precompile` + `scan-deps`
  workflow is the most mature.
- **Static analysis**: enable `clang-tidy` with `modernize-*`,
  `cppcoreguidelines-*`, `bugprone-*`, and `performance-*` checks. Use
  `-Werror` in CI.
- **Sanitizers**: compile with `-fsanitize=address,undefined` during development.
- **Formatter**: `clang-format` with a project-wide `.clang-format` config.

---

## How to Use This Skill

1. **For new code**: write C++20 as baseline. Reach for C++23 features
   (`std::expected`, `std::print`, deducing `this`, `if consteval`) when the
   toolchain supports them. Mention C++26 features as forward-looking
   alternatives when relevant.

2. **For code review / modernization**: identify legacy patterns (raw
   `new`/`delete`, `typedef`, SFINAE, `#define` constants, raw loops) and
   suggest modern replacements from the table above. Be incremental — don't
   rewrite everything at once.

3. **For API design**: prefer value types, use `std::expected` for fallible
   operations, constrain templates with concepts, and use designated initializers
   for config/options structs.

4. **For explaining features**: use concise examples. Show the "before" (legacy)
   and "after" (modern) patterns. Mention compiler support status for newer features.

When in doubt, consult the reference files for deeper guidance:
- `references/raii-and-ownership.md` — smart pointers, RAII patterns, Rule of Zero/Five
- `references/concepts-and-templates.md` — writing and using concepts, requires clauses
- `references/ranges-and-views.md` — range adaptors, custom views, projections
- `references/modules.md` — module structure, partitions, build system integration
- `references/deducing-this.md` — explicit object parameters, CRTP replacement
- `references/compile-time-programming.md` — constexpr family, compile-time containers
- `references/cpp26-preview.md` — reflection, contracts, pattern matching outlook
