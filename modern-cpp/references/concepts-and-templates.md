# Concepts & Constrained Templates (C++20)

## Table of Contents
1. Why Concepts
2. Using Standard Concepts
3. Defining Custom Concepts
4. requires Clauses & Expressions
5. Concept Overloading & Subsumption
6. Abbreviated Function Templates
7. Practical Patterns

---

## 1. Why Concepts

Before C++20, constraining templates required SFINAE — fragile, hard to read,
and producing inscrutable error messages. Concepts provide a first-class way to
name and enforce requirements on template parameters.

```cpp
// SFINAE (old way — avoid)
template <typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>
T old_gcd(T a, T b);

// Concepts (modern — prefer)
template <std::integral T>
T gcd(T a, T b);
```

When a concept constraint is not satisfied, the compiler produces a clear error
pointing at the concept, not a wall of template instantiation noise.

## 2. Using Standard Concepts

The `<concepts>` header provides commonly needed constraints:

| Concept                        | Meaning                                    |
|-------------------------------|--------------------------------------------|
| `std::integral`               | Integer type (char, int, long, etc.)       |
| `std::floating_point`         | float, double, long double                 |
| `std::signed_integral`        | Signed integer                             |
| `std::unsigned_integral`      | Unsigned integer                           |
| `std::same_as<T, U>`         | T and U are the same type                  |
| `std::derived_from<D, B>`    | D derives from B                           |
| `std::convertible_to<From,To>`| Implicit + explicit conversion exists      |
| `std::constructible_from<T, Args...>` | T can be constructed from Args    |
| `std::copyable`               | Copy constructible + copy assignable       |
| `std::movable`                | Move constructible + move assignable       |
| `std::regular`                | Copyable + default constructible + equality comparable |
| `std::invocable<F, Args...>` | F can be called with Args                  |
| `std::predicate<F, Args...>` | Invocable returning bool                   |
| `std::ranges::range`          | Has begin() and end()                      |
| `std::ranges::random_access_range` | Random access range                  |

## 3. Defining Custom Concepts

A concept is a named boolean expression evaluated at compile time:

```cpp
template <typename T>
concept Hashable = requires(T t) {
    { std::hash<T>{}(t) } -> std::convertible_to<std::size_t>;
};

template <typename T>
concept Serializable = requires(T t, std::ostream& os, std::istream& is) {
    { t.serialize(os) } -> std::same_as<void>;
    { T::deserialize(is) } -> std::same_as<T>;
};

template <typename C>
concept Container = requires(C c) {
    typename C::value_type;
    typename C::iterator;
    { c.begin() } -> std::same_as<typename C::iterator>;
    { c.end() } -> std::same_as<typename C::iterator>;
    { c.size() } -> std::convertible_to<std::size_t>;
};
```

Concepts can compose using `&&` and `||`:

```cpp
template <typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;
```

## 4. requires Clauses & Expressions

### requires clause (on a template or function)
```cpp
template <typename T>
    requires std::integral<T>
T absolute(T x) { return x < 0 ? -x : x; }
```

### Trailing requires clause
```cpp
template <typename T>
T absolute(T x) requires std::integral<T> { return x < 0 ? -x : x; }
```

### requires expression (anonymous concept inline)
```cpp
template <typename T>
    requires requires(T a, T b) { a + b; a - b; a * b; }
T dot(T a, T b) { return a * b; }
```

### Compound requirements
```cpp
requires(T t) {
    { t.size() } noexcept -> std::convertible_to<std::size_t>;
    // ↑ checks:  expression is valid
    //            expression is noexcept
    //            return type is convertible to size_t
}
```

### Nested requirements
```cpp
template <typename T>
concept SmallTrivial = requires {
    requires sizeof(T) <= 16;
    requires std::is_trivially_copyable_v<T>;
};
```

## 5. Concept Overloading & Subsumption

When multiple constrained overloads match, the compiler picks the *most
constrained* one (subsumption):

```cpp
template <typename T>
void process(T t) { /* generic fallback */ }

template <std::integral T>
void process(T t) { /* integer-specific */ }

template <std::signed_integral T>
void process(T t) { /* signed integer-specific — most constrained, wins */ }
```

`std::signed_integral` subsumes `std::integral` because its definition includes
`std::integral<T>` as a sub-expression. The compiler picks the most specific
match.

## 6. Abbreviated Function Templates

Use `auto` parameters constrained by concepts for concise generic functions:

```cpp
void print(const std::ranges::range auto& r) {
    for (const auto& elem : r) {
        std::println("{}", elem);
    }
}

auto add(std::integral auto a, std::integral auto b) {
    return a + b;
}
```

Each `auto` parameter introduces an independent template parameter.

## 7. Practical Patterns

### Constraining class templates
```cpp
template <std::regular T>
class Cache {
    std::unordered_map<std::string, T> store_;
    // T must be regular (copyable, default-constructible, equality-comparable)
};
```

### Constraining member functions
```cpp
class Serializer {
public:
    template <typename T>
        requires Serializable<T>
    void write(const T& obj);

    template <typename T>
        requires std::is_trivially_copyable_v<T>
    void write_raw(const T& obj);
};
```

### Static interface enforcement (replacing CRTP)
```cpp
template <typename T>
concept Drawable = requires(T t, Canvas& c) {
    { t.draw(c) } -> std::same_as<void>;
    { t.bounds() } -> std::same_as<Rect>;
};

void render(Drawable auto& shape, Canvas& canvas) {
    auto b = shape.bounds();
    if (canvas.visible(b)) shape.draw(canvas);
}
```

### Constraining lambdas
```cpp
auto transform = []<std::integral T>(T value) {
    return value * 2;
};
```
