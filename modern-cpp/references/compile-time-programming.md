# Compile-Time Programming: constexpr, consteval, constinit

## Table of Contents
1. The constexpr Family at a Glance
2. constexpr Functions & Variables
3. consteval (Immediate Functions)
4. constinit
5. if consteval (C++23)
6. constexpr Containers (C++20/23)
7. Practical Patterns

---

## 1. The constexpr Family at a Glance

| Keyword       | Introduced | Meaning                                              |
|---------------|-----------|------------------------------------------------------|
| `constexpr`   | C++11     | *May* evaluate at compile time if inputs are constant |
| `consteval`   | C++20     | *Must* evaluate at compile time (immediate function)  |
| `constinit`   | C++20     | Variable must be initialized at compile time (not necessarily const) |
| `if consteval`| C++23     | Branch based on whether currently in constant evaluation |

### Key distinction
- `constexpr` = "compile-time capable"
- `consteval` = "compile-time only"
- `constinit` = "compile-time initialized, runtime mutable"

## 2. constexpr Functions & Variables

### constexpr variables
```cpp
constexpr int max_size = 1024;          // compile-time constant
constexpr double pi = 3.14159265358979; // compile-time constant
```

A `constexpr` variable is implicitly `const` and must be initialized with a
constant expression.

### constexpr functions
```cpp
constexpr int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i)
        result *= i;
    return result;
}

// Used at compile time:
constexpr int f10 = factorial(10);  // evaluated at compile time
static_assert(factorial(5) == 120);

// Used at runtime:
int n;
std::cin >> n;
int fn = factorial(n);  // evaluated at runtime — also fine
```

### What's allowed in constexpr (evolving by standard)

| Feature                          | C++11 | C++14 | C++17 | C++20 | C++23 |
|---------------------------------|-------|-------|-------|-------|-------|
| Simple expressions, recursion   | Yes   | Yes   | Yes   | Yes   | Yes   |
| Local variables, loops          | No    | Yes   | Yes   | Yes   | Yes   |
| if/switch                       | No    | Yes   | Yes   | Yes   | Yes   |
| `try`/`catch`                   | No    | No    | No    | Yes   | Yes   |
| Virtual functions               | No    | No    | No    | Yes   | Yes   |
| `dynamic_cast`, `typeid`        | No    | No    | No    | Yes   | Yes   |
| Transient memory allocation     | No    | No    | No    | Yes   | Yes   |
| `std::vector`, `std::string`    | No    | No    | No    | Yes*  | Yes   |
| `std::unique_ptr`               | No    | No    | No    | No    | Yes   |
| `std::optional`, `std::variant` | No    | No    | No    | No    | Yes   |

*Transient only in C++20 (allocated and freed within the same constant expression).
C++23 relaxes some of these further.

## 3. consteval (Immediate Functions)

A `consteval` function *must* be evaluated at compile time. If called with
non-constant arguments, it's a compile error.

```cpp
consteval int compile_time_hash(std::string_view sv) {
    int hash = 0;
    for (char c : sv) hash = hash * 31 + c;
    return hash;
}

constexpr int h = compile_time_hash("hello");  // OK — compile time
// int h2 = compile_time_hash(runtime_string);  // ERROR — not constant
```

Use `consteval` when:
- The function should *never* run at runtime (lookup tables, string hashing).
- You want to guarantee zero runtime cost.
- You want compile errors if someone accidentally uses it at runtime.

### consteval + static_assert for validation
```cpp
consteval void validate_config(int width, int height) {
    if (width <= 0 || height <= 0)
        throw "dimensions must be positive";  // compile error if violated
    if (width > 8192 || height > 8192)
        throw "dimensions too large";
}

constexpr int W = 1920, H = 1080;
// validate_config(W, H);  // would fail to compile if invalid
```

## 4. constinit

`constinit` guarantees that a variable with static or thread-local storage
duration is initialized at compile time, but the variable remains mutable at
runtime. This solves the **Static Initialization Order Fiasco**.

```cpp
// file1.cpp
constinit int global_counter = 0;  // zero-init at compile time, safe

void increment() { ++global_counter; }  // OK — mutable at runtime

// file2.cpp
extern constinit int global_counter;
// Guaranteed to be initialized before any dynamic initialization runs
```

### constinit vs const vs constexpr

| Keyword      | Compile-time init | Immutable | Static/thread storage only |
|-------------|-------------------|-----------|---------------------------|
| `const`      | No (may be)       | Yes       | No                        |
| `constexpr`  | Yes               | Yes       | No                        |
| `constinit`  | Yes               | No        | Yes                       |

## 5. if consteval (C++23)

`if consteval` replaces `std::is_constant_evaluated()` with cleaner syntax:

```cpp
constexpr double power(double base, int exp) {
    if consteval {
        // Compile-time path — can call consteval functions
        double result = 1.0;
        for (int i = 0; i < exp; ++i) result *= base;
        return result;
    } else {
        // Runtime path — can use non-constexpr facilities
        return std::pow(base, exp);
    }
}
```

The `if consteval` branch:
- Activates when the function is being constant-evaluated
- Can call `consteval` functions (which `if (std::is_constant_evaluated())`
  could not)
- Is a statement, not an expression — no ternary form

## 6. constexpr Containers (C++20/23)

### C++20: transient constexpr allocation
```cpp
constexpr int sum_of_first_n(int n) {
    std::vector<int> v;              // allocation happens at compile time
    for (int i = 1; i <= n; ++i)
        v.push_back(i);
    int sum = 0;
    for (int x : v) sum += x;
    return sum;                      // v is freed before returning
    // The allocation is "transient" — it doesn't escape the constant expression
}

static_assert(sum_of_first_n(10) == 55);
```

### C++23: expanded constexpr support
```cpp
constexpr auto make_optional_pair() {
    auto a = std::make_unique<int>(42);   // constexpr unique_ptr!
    auto opt = std::optional<int>(7);     // constexpr optional!
    return *a + *opt;
}

static_assert(make_optional_pair() == 49);
```

## 7. Practical Patterns

### Compile-time lookup tables
```cpp
consteval auto make_hex_table() {
    std::array<int, 128> table{};
    for (int i = 0; i < 128; ++i) table[i] = -1;
    for (int i = '0'; i <= '9'; ++i) table[i] = i - '0';
    for (int i = 'a'; i <= 'f'; ++i) table[i] = i - 'a' + 10;
    for (int i = 'A'; i <= 'F'; ++i) table[i] = i - 'A' + 10;
    return table;
}

constexpr auto hex_table = make_hex_table();
// hex_table is baked into the binary — zero runtime cost
```

### Compile-time string processing
```cpp
consteval std::size_t count_words(std::string_view sv) {
    std::size_t count = 0;
    bool in_word = false;
    for (char c : sv) {
        if (c == ' ' || c == '\t' || c == '\n') {
            in_word = false;
        } else if (!in_word) {
            in_word = true;
            ++count;
        }
    }
    return count;
}

static_assert(count_words("hello world") == 2);
```

### constexpr class with invariants
```cpp
class Fraction {
public:
    constexpr Fraction(int num, int den) : num_(num), den_(den) {
        if (den_ == 0) throw "zero denominator";
        auto g = gcd(num_ < 0 ? -num_ : num_, den_);
        num_ /= g;
        den_ /= g;
        if (den_ < 0) { num_ = -num_; den_ = -den_; }
    }

    constexpr int numerator() const { return num_; }
    constexpr int denominator() const { return den_; }

    constexpr Fraction operator+(Fraction rhs) const {
        return {num_ * rhs.den_ + rhs.num_ * den_, den_ * rhs.den_};
    }

private:
    int num_, den_;

    static constexpr int gcd(int a, int b) {
        return b == 0 ? a : gcd(b, a % b);
    }
};

static_assert(Fraction(1, 3) + Fraction(1, 6) == Fraction(1, 2));
```
