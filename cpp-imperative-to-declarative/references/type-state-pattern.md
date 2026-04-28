# The Type+State Pattern (Layer 2)

The deeper transformation from the talk (14:15–23:50). Use only when Layer 1 (`three-techniques.md`) leaves you with one of:

- A flat sequence of `create_and_add<Kind>(...)` calls when what you mean is a *tree*
- Heterogeneous siblings that should live side-by-side as one declaration
- A need to *pass around* the description as a value (store it, generate it, generic-process it)

Layer 2 turns the description into data. Execution becomes a separate, generic step.

---

## The four moves

Apply these in order. Each one isolates a piece of complexity.

### Move 1 — Encapsulate (Type, State) into a struct

For each kind of widget, define a struct holding *the data you'd have passed to `create_and_add<Kind>`* plus a member function that performs the create+attach.

```cpp
struct ButtonSpec {
  const char* label;
  Callback    on_click;

  void create_and_add(Container& c) const {
    auto* w = new Button(label);
    w->set_callback(on_click);
    c.add(w);
  }
};

struct LabelSpec {
  const char* text;

  void create_and_add(Container& c) const {
    c.add(new Label(text));
  }
};

struct SliderSpec {
  const char* label;
  float       lo, hi;
  float*      bound_to;

  void create_and_add(Container& c) const {
    auto* w = new Slider(label, lo, hi);
    w->bind(bound_to);
    c.add(w);
  }
};
```

The struct is *the type plus the state it needs*. The member function is the construction algorithm — the only place execution lives.

**Naming convention:** call the member the same thing across all specs (`create_and_add`, `render`, `apply`, whatever fits the domain). Uniformity is the whole point — Move 4 will iterate over them by name.

### Move 2 — The leap of faith

The spec structs above will be instantiated, used for one expression, and destroyed. That feels wrong the first time you see it. It is not wrong — it is the point.

```cpp
ButtonSpec{"OK", on_ok}.create_and_add(panel);  // born + dies on this line
```

This object exists as a *description*, not a long-lived participant. It is the C++ analog of the data inside a JSX literal: gone after rendering, no identity, no resources to manage.

Two things follow:

1. **Make specs trivial value types.** No virtual base, no destructors, no resources. They should be aggregate-initializable. Often `struct` of `const char*`, `int`, `float`, function pointers / `std::function`.
2. **Don't fight the compiler optimizing them away.** That's the engine compiling your declaration into the same imperative calls you would have written by hand — minus the duplication.

If you find yourself wanting a spec to live longer (cache it, reuse it across frames), check first whether you actually want to cache the *constructed widget*, not the spec. The spec is just paper.

### Move 3 — Heterogeneous collection (`std::tuple`)

Siblings of different kinds need to live in one container. `std::vector` won't work — the kinds are different types. `std::variant` works but forces a runtime tag and a switch in the engine. `std::tuple` keeps each element's static type and lets the engine dispatch by overload / generic lambda.

```cpp
auto ui = std::make_tuple(
  LabelSpec  {"Welcome"},
  ButtonSpec {"OK",     on_ok},
  ButtonSpec {"Cancel", on_cancel},
  SliderSpec {"speed", 0.f, 10.f, &speed}
);
```

With CTAD (C++17) and aggregate-CTAD (C++20) you can usually write this without naming the spec types repeatedly:

```cpp
auto ui = std::tuple{
  LabelSpec  {"Welcome"},
  ButtonSpec {"OK",     on_ok},
  ButtonSpec {"Cancel", on_cancel},
  SliderSpec {"speed", 0.f, 10.f, &speed}
};
```

This *is* the data structure. It's a value. You can return it, pass it, store it, generate it from configuration.

### Move 4 — Execute with `std::apply`

`std::apply(f, tuple)` calls `f` with the tuple's elements unpacked. Combined with a generic lambda and a fold expression, you get a one-liner that calls `create_and_add` on every spec:

```cpp
std::apply(
  [&](auto const&... spec) { (spec.create_and_add(panel), ...); },
  ui
);
```

The fold `(... , ...)` is a left-fold over the comma operator — equivalent to `spec₀.create_and_add(panel); spec₁.create_and_add(panel); ...`. Order is well-defined (left-to-right), which matters for APIs whose semantics depend on call order (ImGui most definitely included).

That tiny `std::apply` call is the *engine*. Every UI in the codebase goes through it.

---

## Putting the moves together

```cpp
auto build_settings_panel(Container& panel, float& speed) {
  auto ui = std::tuple{
    LabelSpec  {"Settings"},
    ButtonSpec {"Save", on_save},
    SliderSpec {"speed", 0.f, 10.f, &speed},
  };
  std::apply(
    [&](auto const&... spec) { (spec.create_and_add(panel), ...); },
    ui
  );
}
```

Compare to the original imperative form: the body now reads as **what the panel is**, not how to construct it. The construction algorithm is in one place (`create_and_add` on each spec) and the iteration is in one place (the `std::apply` line).

---

## Going further: nested trees

A spec can itself contain a tuple of children. That gives you nesting without leaving the language:

```cpp
template <typename... Children>
struct WindowSpec {
  const char* title;
  std::tuple<Children...> children;

  void create_and_add(Container& root) const {
    auto* w = new Window(title);
    root.add(w);
    std::apply(
      [&](auto const&... c) { (c.create_and_add(*w), ...); },
      children
    );
  }
};

// Helper so callers don't have to spell out the template args
template <typename... C>
WindowSpec(const char*, C...) -> WindowSpec<C...>;  // CTAD deduction guide

// Usage:
auto ui = WindowSpec{"Settings",
  std::tuple{
    LabelSpec {"Welcome"},
    ButtonSpec{"OK", on_ok},
  }
};
ui.create_and_add(root);
```

A more polished design uses variadic constructors and parameter packs in place of an explicit `std::tuple{}`, giving `Window{"title", LabelSpec{...}, ButtonSpec{...}}` directly. See `imgui-example.md` for the polished form.

---

## C++ standard requirements

Each move requires a specific standard:

| Move | Required C++ | Feature used |
|------|--------------|--------------|
| 1. Spec struct | C++11 | aggregate init |
| 1. Designated init for specs | C++20 | `ButtonSpec{.label="OK", .on_click=...}` |
| 3. CTAD on `std::tuple{...}` | C++17 | class template argument deduction |
| 3. Aggregate CTAD | C++20 | spec types deduce from braced-init |
| 4. `std::apply` | C++17 | unpacking tuples to a callable |
| 4. Generic lambda `[](auto&&...)` | C++14 / C++20 (in templates) | parameter-pack lambdas |
| 4. Fold expressions `(... , ...)` | C++17 | applying op over a pack |

vkd is C++23, so all of these are available. If targeting C++14/17 you'll need workarounds (manual `std::index_sequence` unpacking instead of `std::apply` + generic lambda, or `template <std::size_t I = 0> if constexpr` recursion).

---

## When NOT to use Layer 2

- **One-of-a-kind UI**: a debug overlay that's three hardcoded lines. The tuple machinery is overkill.
- **Order doesn't fold cleanly left-to-right**: e.g., children must be added *after* a sibling computes a layout that the second child depends on. Layer 2 still works but you'll be threading state through the lambda; consider whether that's clearer than the imperative version.
- **Specs would need polymorphism**: if a `Children...` pack would need runtime type dispatch (e.g., reading from a config file at runtime), `std::variant` or virtual dispatch is the right tool, not Layer 2.

The rule of thumb: if the description is *known statically*, Layer 2 is the right shape. If the description is *built dynamically from runtime input*, fall back to a runtime-polymorphic container (`std::vector<std::variant<...>>` or `std::vector<std::unique_ptr<ISpec>>`).
