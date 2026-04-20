# C++26 Preview: Reflection, Contracts & More

## Table of Contents
1. Status of C++26
2. Static Reflection
3. Contracts
4. Other Notable C++26 Features
5. Compiler Support & Experimentation

---

## 1. Status of C++26

The C++26 draft was feature-frozen in June 2025 at the Sofia meeting. The
standard is expected to be formally published in late 2026. The three headline
features are static reflection, contracts, and `std::execution` (sender/receiver).
Compiler support is actively being developed.

## 2. Static Reflection

Static reflection is widely considered the most transformative C++ feature in
decades. It enables compile-time introspection of program entities (types,
members, enumerators, etc.) and code generation based on those observations.

### Core primitives

| Syntax / API                   | Purpose                                    |
|-------------------------------|-------------------------------------------|
| `^T`                          | Reflect on T — yields `std::meta::info`   |
| `[:refl:]`                    | Splice — turn reflection back into code   |
| `std::meta::info`             | Opaque type representing a reflected entity |
| `std::meta::name_of(refl)`   | Get the name of a reflected entity        |
| `std::meta::members_of(refl)`| Get members of a class/enum/namespace     |
| `std::meta::type_of(refl)`   | Get the type of a reflected entity        |
| `std::meta::is_public(refl)` | Check access specifier                    |
| `template for`                | Expansion statement — iterate over reflections |

### Example: enum to string
```cpp
template <typename E>
    requires std::is_enum_v<E>
constexpr std::string enum_to_string(E value) {
    template for (constexpr auto e : std::meta::members_of(^E)) {
        if (value == [:e:]) {
            return std::string(std::meta::name_of(e));
        }
    }
    return "<unnamed>";
}

enum Color { red, green, blue };
static_assert(enum_to_string(Color::red) == "red");
```

### Example: struct-to-JSON serialization
```cpp
template <typename T>
std::string to_json(const T& obj) {
    std::string result = "{";
    bool first = true;
    template for (constexpr auto member : std::meta::members_of(^T)) {
        if (!first) result += ", ";
        first = false;
        result += std::format("\"{}\":{}", 
            std::meta::name_of(member),
            to_json_value(obj.[:member:]));
    }
    return result + "}";
}
```

### Example: automatic comparison
```cpp
template <typename T>
constexpr bool generic_equal(const T& a, const T& b) {
    template for (constexpr auto member : std::meta::members_of(^T)) {
        if (a.[:member:] != b.[:member:]) return false;
    }
    return true;
}
```

### What reflection enables
- Automatic serialization/deserialization (JSON, protobuf, etc.)
- Enum-to-string and string-to-enum conversions
- Automatic comparison operators
- ORM-style database binding
- GUI property editors
- Dependency injection frameworks
- Test framework improvements (auto-discovery of test cases)
- Replacing macros used for code generation

### Expansion statements (`template for`)
Unlike regular `for` loops, `template for` expands at compile time — each
iteration can produce different types. This is essential for iterating over
struct members that have different types.

## 3. Contracts

Contracts add first-class support for design-by-contract: preconditions,
postconditions, and assertions.

### Syntax
```cpp
int safe_divide(int a, int b)
    [[pre: b != 0]]                    // precondition
    [[post r: r >= 0 || a < 0]]        // postcondition (r = return value)
{
    contract_assert(b != 0);           // assertion within function body
    return a / b;
}
```

### Evaluation semantics

The standard defines four evaluation modes for contract assertions:

| Mode          | Check?  | On violation          |
|---------------|---------|-----------------------|
| `ignore`      | No      | —                     |
| `observe`     | Yes     | Log + continue        |
| `enforce`     | Yes     | Log + terminate       |
| `quick_enforce`| Yes    | Terminate (no log)    |

The mode is selected by the implementation (typically via compiler flags), not
in source code. This allows the same codebase to be built with contracts
enabled (debug/test) or disabled (optimized release).

### Key design decisions
- Contract predicates must have no side effects (UB if they do).
- The `r` in `[[post r: ...]]` names the return value.
- Contracts are not exceptions — violation terminates by default.
- Virtual functions inherit contracts from their base class declarations.
  (planned for a future revision — not in the initial C++26 feature set).

### Practical usage
```cpp
class Stack {
public:
    void push(int value)
        [[pre: !full()]]
    {
        data_[size_++] = value;
    }

    int pop()
        [[pre: !empty()]]
        [[post r: size() == old_size - 1]]  // "old_size" is illustrative
    {
        return data_[--size_];
    }

    bool empty() const { return size_ == 0; }
    bool full() const { return size_ == capacity_; }

private:
    int data_[100];
    int size_ = 0;
    static constexpr int capacity_ = 100;
};
```

## 4. Other Notable C++26 Features

### `std::execution` (Sender/Receiver)
A framework for structured asynchronous programming. Senders represent lazy
asynchronous operations; receivers consume their results. This replaces ad-hoc
thread pool and future patterns with a composable, type-safe model.

```cpp
auto work = std::execution::schedule(pool.get_scheduler())
    | std::execution::then([] { return compute_data(); })
    | std::execution::then([](auto data) { return process(data); });

std::this_thread::sync_wait(std::move(work));
```

### `#embed` directive
Include binary resources directly in source code:
```cpp
constexpr unsigned char icon[] = {
    #embed "icon.png"
};
```

### `std::simd` (data-parallel types)
SIMD vector types in the standard library for portable data parallelism:
```cpp
using floatv = std::simd<float>;
floatv a = {1.0f, 2.0f, 3.0f, 4.0f};
floatv b = {5.0f, 6.0f, 7.0f, 8.0f};
floatv c = a * b;  // element-wise multiply using SIMD instructions
```

### Hardened standard library
Bounds-checked versions of standard library operations (e.g., `operator[]` on
`std::vector` with bounds checking in hardened mode). Already deployed at scale
at Apple and Google with minimal performance overhead.

### Pack indexing
Access individual elements of parameter packs by index:
```cpp
template <typename... Ts>
using first_t = Ts...[0];

template <typename... Ts>
using last_t = Ts...[sizeof...(Ts) - 1];
```

### `= delete("reason")`
Provide a reason when deleting special member functions:
```cpp
class NonCopyable {
    NonCopyable(const NonCopyable&) = delete("use clone() instead");
};
```

## 5. Compiler Support & Experimentation

As of early 2026, C++26 features are in various stages of implementation:

- **Reflection**: Bloomberg's Clang/P2996 fork has the most complete support.
  GCC is actively implementing. Use Compiler Explorer (godbolt.org) to experiment.
- **Contracts**: Experimental Clang fork by Eric Fiselier; some GCC variants
  on Compiler Explorer. Not yet in mainstream releases.
- **`std::execution`**: Available in libunifex (Meta), stdexec (NVIDIA), and
  being integrated into standard library implementations.
- **`#embed`**: Clang 19+, GCC 15+.
- **Pack indexing**: Clang 19+, GCC 14+, MSVC latest preview.

For experimenting with C++26 features, use Compiler Explorer with the latest
compiler versions and the `-std=c++26` or `-std=c++2c` flag.
