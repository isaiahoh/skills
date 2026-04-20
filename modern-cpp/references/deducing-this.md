# Deducing `this` / Explicit Object Parameter (C++23)

## Table of Contents
1. What It Is
2. Basic Syntax
3. De-quadruplication
4. Return Types and Forwarding (the practical rules)
5. Recursive Lambdas
6. Simplified CRTP
7. Pass-by-Value this
8. Restrictions
9. Compiler Support

---

## 1. What It Is

C++23 allows making the implicit `this` pointer of a member function explicit by
adding it as the first parameter, prefixed with the `this` keyword. When combined
with template deduction, the type and value category of the calling object are
deduced automatically — hence "deducing this."

By convention, the parameter type is named `Self` and the parameter `self`,
mirroring Python's convention.

## 2. Basic Syntax

### Explicit (non-deduced) object parameter
```cpp
struct Widget {
    void method(this Widget& self)       { /* lvalue */ }
    void method(this const Widget& self) { /* const lvalue */ }
    void method(this Widget&& self)      { /* rvalue */ }
};
```

### Deduced object parameter (the power feature)
```cpp
struct Widget {
    template <typename Self>
    void method(this Self&& self) {
        // Self deduces to Widget&, const Widget&, or Widget&&
        // depending on how the object is used
    }
};
```

### Shorthand with `auto`
```cpp
struct Widget {
    void method(this auto&& self) {
        // equivalent to the template version above
    }
};
```

## 3. De-quadruplication

The primary motivation: before C++23, providing optimal overloads for all
cv/ref qualifier combinations required up to four duplicated functions.

### Before (4 overloads)
```cpp
class TextBuffer {
    std::string data_;
public:
    std::string& content() & { return data_; }
    const std::string& content() const& { return data_; }
    std::string&& content() && { return std::move(data_); }
    const std::string&& content() const&& { return std::move(data_); }
};
```

### After (1 function)
```cpp
class TextBuffer {
    std::string data_;
public:
    template <typename Self>
    auto&& content(this Self&& self) {
        return std::forward<Self>(self).data_;
    }
};
```

The deduced version correctly returns `std::string&` for lvalue access,
`const std::string&` for const access, and `std::string&&` for rvalue access.

For simpler cases, `std::forward_like` (C++23) provides a cleaner spelling:
```cpp
class TextBuffer {
    std::string data_;
public:
    auto&& content(this auto&& self) {
        return std::forward_like<decltype(self)>(self.data_);
    }
};
```

## 4. Return Types and Forwarding (the practical rules)

Deducing `this` only delivers on its promise when the return type and the
return expression cooperate. Getting either wrong silently collapses the
value category back to a plain lvalue, and you end up with the same bugs
the duplicated overloads were supposed to fix.

### Rule 1 — return `decltype(auto)`, not `auto&`

`auto&` always deduces to an **lvalue reference**, so it cannot propagate
`T&&` from an rvalue-qualified call, and it cannot cover an accessor whose
return expression is a prvalue (e.g. a `std::span` built from a pointer
and a size). `decltype(auto)` preserves whatever the return expression
actually is — lvalue reference, rvalue reference, or prvalue.

```cpp
class Pool {
    std::vector<Component> data_;
public:
    // BAD: auto& flattens everything to an lvalue ref.
    // On `std::move(pool).get(i)` the caller loses the xvalue.
    template <class Self>
    auto& get(this Self&& self, size_t i) {
        return self.data_[i];
    }

    // GOOD: decltype(auto) + std::forward.
    // - Pool&        → Component&
    // - Pool const&  → Component const&
    // - Pool&&       → Component&& (caller can move out)
    template <class Self>
    decltype(auto) get(this Self&& self, size_t i) {
        return std::forward<Self>(self).data_[i];
    }
};
```

### Rule 2 — you **must** `std::forward<Self>(self)` in the return expression

A bare `self.member` access is an lvalue regardless of how `Self` was
deduced, because `self` itself is a named parameter (which is always an
lvalue inside the function body). Without the forward, `decltype(auto)`
sees an lvalue and you get back `T&` no matter what.

```cpp
// BAD — self.data_ is an lvalue even when Self = Pool&&
template <class Self>
decltype(auto) get(this Self&& self, size_t i) {
    return self.data_[i];                   // always Component&
}

// GOOD
template <class Self>
decltype(auto) get(this Self&& self, size_t i) {
    return std::forward<Self>(self).data_[i];
}
```

### Rule 3 — prefer named `template <class Self>` when you need to forward

`this auto&& self` is the shorthand, but forwarding from it requires
`std::forward<decltype(self)>(self)`, which is verbose and obscures
intent. A named template parameter reads better:

```cpp
// Shorthand — fine when you don't need to forward
decltype(auto) peek(this auto&& self) { return self.size_; }

// Named Self — preferred when the body forwards
template <class Self>
decltype(auto) get(this Self&& self, size_t i) {
    return std::forward<Self>(self).data_[i];
}
```

### Rule 4 — skip the forward when there is nothing to forward

Forwarding is not a magic incantation; it only matters when the return
expression has a value category you want to propagate. These cases do
**not** need (or benefit from) `std::forward`:

- The accessor constructs a **non-owning view** by value from unrelated
  pieces, e.g. `std::span{self.data_.data(), self.data_.size()}`. The
  span is a prvalue built from a raw pointer plus a size; there is no
  reference to the stored object flowing through the return.
- The member is a **non-reference pointer or integer** — `self.ptr_` or
  `self.count_` is already a copy; value category is irrelevant.
- The method is **const-only** (no mutable counterpart). Deducing `this`
  buys nothing; just write `void foo() const`.

```cpp
// No forward needed — span is a prvalue built from pointer + size.
template <class Self>
auto data(this Self&& self) noexcept {
    return std::span{self.data_.data(), self.data_.size()};
}
```

When in doubt: if removing `std::forward<Self>(self)` does not change the
deduced return type, the forward was decorative and you can drop it.

### Rule 5 — chained calls need `.template`, and the forward has to wrap the whole chain

When a deducing-`this` method forwards to **another** deducing-`this`
method on the same object, two things trip people up:

1. The inner method is a template member of a dependent expression, so
   it must be called with `.template`:
   ```cpp
   std::forward<Self>(self).template inner<T>()
   ```
2. The forward has to be on the outermost object in the chain, not on
   the intermediate call. Otherwise the rvalue qualification stops at
   the first link and the rest of the chain sees an lvalue.

```cpp
template <class... Cs>
class Registry {
    std::tuple<Pool<Cs>...> pools_;

    // Deducing-this pool accessor.
    template <class T, class Self>
    decltype(auto) pool(this Self&& self) noexcept {
        return std::get<Pool<T>>(std::forward<Self>(self).pools_);
    }

public:
    // Forward self through the chain so an rvalue Registry yields an
    // rvalue Pool, which in turn yields an rvalue Component.
    template <class T, class Self>
    decltype(auto) get(this Self&& self, Entity e) {
        return std::forward<Self>(self).template pool<T>().get(e);
        //     └───── forwarded ─────┘ └ .template ┘
    }
};
```

### Checklist before committing a deducing-`this` accessor

- [ ] Return type is `decltype(auto)` (unless you have a specific reason
      to narrow it, or the return is a prvalue view like `std::span`).
- [ ] The return expression begins with `std::forward<Self>(self).…`,
      unless the member is a value-type field that can't carry value
      category.
- [ ] If the body calls another deducing-`this` template member, the
      call site uses `.template` **and** the forward is on the outer
      object, not the inner call.
- [ ] If the method is const-only (no mutable twin), it is a plain
      `void foo() const` — deducing `this` adds noise without benefit.
- [ ] For a pair that would otherwise be `&` / `const&` only, consider
      whether `this Self&&` is worth the template cost over a plain
      const-only member returning `T const&`. Deducing `this` shines when
      you genuinely need all four qualifier combinations — not just two.

## 5. Recursive Lambdas

Before C++23, writing a recursive lambda required `std::function` (with
heap allocation overhead) or a Y-combinator. With deducing this, the lambda
can simply refer to itself:

```cpp
auto factorial = [](this auto self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
};

std::println("{}", factorial(5));  // 120
```

### Overloaded visitor pattern
Combine with the overload pattern for `std::variant` visitation:

```cpp
template <typename... Fs>
struct overloaded : Fs... { using Fs::operator()...; };

std::variant<int, double, std::string> v = "hello";

std::visit(overloaded{
    [](this auto, int i) { std::println("int: {}", i); },
    [](this auto, double d) { std::println("double: {:.2f}", d); },
    [](this auto, const std::string& s) { std::println("string: {}", s); },
}, v);
```

## 5. Simplified CRTP

The Curiously Recurring Template Pattern (CRTP) traditionally requires the base
class to be templated on the derived class, with `static_cast` to access derived
members. Deducing this eliminates both requirements:

### Before (classic CRTP)
```cpp
template <typename Derived>
struct Countable {
    static int count;
    Countable() { ++count; }
    ~Countable() { --count; }
    static int get_count() { return count; }
};
template <typename D> int Countable<D>::count = 0;

struct Widget : Countable<Widget> { };
struct Gadget : Countable<Gadget> { };
```

### After (deducing this — no template on base)
```cpp
struct Base {
    void interface(this auto&& self) {
        // self deduces to the actual derived type
        self.implementation();
    }
};

struct Derived : Base {
    void implementation() {
        std::println("Derived::implementation");
    }
};

Derived d;
d.interface();  // calls Derived::implementation via deduction
```

This is simpler, more readable, and scales better with multiple inheritance
levels. No `static_cast`, no template parameter on the base class.

### Builder pattern with chaining
```cpp
struct BuilderBase {
    auto&& set_name(this auto&& self, std::string name) {
        self.name_ = std::move(name);
        return std::forward<decltype(self)>(self);
    }
protected:
    std::string name_;
};

struct WidgetBuilder : BuilderBase {
    auto&& set_size(this auto&& self, int w, int h) {
        self.width_ = w;
        self.height_ = h;
        return std::forward<decltype(self)>(self);
    }
    int width_ = 0, height_ = 0;
};

auto builder = WidgetBuilder{}
    .set_name("MyWidget")   // returns WidgetBuilder&&, not BuilderBase&&
    .set_size(800, 600);    // chaining works correctly
```

## 6. Pass-by-Value this

You can take the object by value, creating a copy:

```cpp
struct Immutable {
    int value;

    Immutable increment(this Immutable self) {
        self.value++;
        return self;  // returns a modified copy
    }
};

auto a = Immutable{1};
auto b = a.increment();  // a is unchanged, b.value == 2
```

This is useful for functional-style APIs where methods return modified copies.

## 7. Restrictions

- **Cannot be `virtual`**: explicit object parameters are resolved at compile
  time, so virtual dispatch is not supported.
- **Cannot be `static`**: the whole point is the object parameter.
- **No cv/ref qualifiers on the function**: the qualifiers are on the parameter
  type instead (`this const Widget&`, not `void f() const`).
- **Must be the first parameter**: `(this auto&& self, int x)` — `self` comes
  first.
- **Capturing lambdas**: a lambda with captures can only use explicit object
  parameters whose type is related to the lambda's closure type.

## 8. Compiler Support

As of early 2026:
- **MSVC**: supported since VS 2022 17.2 (partial), 17.4+ (full)
- **GCC**: supported since GCC 14
- **Clang**: supported since Clang 18

All major compilers now have full support. Safe to use in projects targeting
C++23.
