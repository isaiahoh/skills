# Worked Example: Declarative ImGui Wrapper

End-to-end refactor of an imperative ImGui block into a declarative `Window{...children}.render()` form, using both layers of the methodology. ImGui is the canonical case: its `Begin/End` blocks are already pretending to be a tree.

This is the target shape for the planned vkd ImGui wrapper. Use it as the reference design when proposing a refactor against ImGui-shaped code.

---

## Starting point — fully imperative ImGui

```cpp
void draw_settings(float& speed, bool& vsync, int& msaa) {
  ImGui::Begin("Settings");
    ImGui::Text("Render");
    ImGui::SliderFloat("speed", &speed, 0.f, 10.f);
    ImGui::Checkbox  ("vsync",  &vsync);
    ImGui::SliderInt ("msaa",   &msaa, 1, 8);
    if (ImGui::Button("Apply")) { apply_changes(); }
  ImGui::End();
}
```

The hidden tree:

```
Window("Settings")
├─ Text     ("Render")
├─ Slider   ("speed", &speed, 0..10)
├─ Checkbox ("vsync", &vsync)
├─ Slider   ("msaa",  &msaa, 1..8)
└─ Button   ("Apply", apply_changes)
```

Everything in this file should make that tree literal.

---

## Step 1 — Layer 1, isolate the Begin/End scope

Even before touching the children, factor `Begin`/`End` into a single helper that takes a callable. This is the simplest scope-bracket idiom — common to ImGui, RAII, transactions, GL state pushes.

```cpp
template <typename F>
void window(const char* title, F&& body) {
  ImGui::Begin(title);
  std::forward<F>(body)();
  ImGui::End();
}

void draw_settings(...) {
  window("Settings", [&]{
    ImGui::Text("Render");
    ImGui::SliderFloat("speed", &speed, 0.f, 10.f);
    ImGui::Checkbox  ("vsync",  &vsync);
    ImGui::SliderInt ("msaa",   &msaa, 1, 8);
    if (ImGui::Button("Apply")) { apply_changes(); }
  });
}
```

The body is now a *region* — explicitly the children of the window. But we still call ImGui imperatively inside it. Continue.

---

## Step 2 — Layer 2, Move 1: a spec per widget kind

```cpp
struct TextSpec {
  const char* text;
  void render() const { ImGui::Text("%s", text); }
};

struct SliderFloatSpec {
  const char* label;
  float*      value;
  float       lo, hi;
  void render() const { ImGui::SliderFloat(label, value, lo, hi); }
};

struct SliderIntSpec {
  const char* label;
  int*        value;
  int         lo, hi;
  void render() const { ImGui::SliderInt(label, value, lo, hi); }
};

struct CheckboxSpec {
  const char* label;
  bool*       value;
  void render() const { ImGui::Checkbox(label, value); }
};

struct ButtonSpec {
  const char*           label;
  std::function<void()> on_click;
  void render() const {
    if (ImGui::Button(label) && on_click) on_click();
  }
};
```

Five tiny structs, each holds *exactly the data ImGui needs* and contains the one-line call. The `render()` member is uniform across all of them — that's what lets Move 4 dispatch generically.

**The leap of faith.** These structs are about to be born and die on a single line of every frame. That's fine. They're descriptions, not objects with identity.

---

## Step 3 — Layer 2, Moves 3 & 4: a `Window` spec with children

```cpp
template <typename... Children>
struct Window {
  const char*             title;
  std::tuple<Children...> children;

  // Variadic ctor so callers don't write std::tuple{} explicitly
  Window(const char* t, Children... cs)
    : title(t), children(std::move(cs)...) {}

  void render() const {
    ImGui::Begin(title);
    std::apply(
      [](auto const&... c) { (c.render(), ...); },
      children
    );
    ImGui::End();
  }
};

// Deduction guide so `Window{"...", Text{...}, Button{...}}` works:
template <typename... C>
Window(const char*, C...) -> Window<C...>;
```

The fold expression `(c.render(), ...)` is the engine — it expands to `c0.render(); c1.render(); ...` in left-to-right order, which matches ImGui's call-order semantics exactly.

---

## Step 4 — call site

```cpp
void draw_settings(float& speed, bool& vsync, int& msaa) {
  Window{"Settings",
    TextSpec       {"Render"},
    SliderFloatSpec{"speed", &speed, 0.f, 10.f},
    CheckboxSpec   {"vsync", &vsync},
    SliderIntSpec  {"msaa",  &msaa, 1, 8},
    ButtonSpec     {"Apply", apply_changes},
  }.render();
}
```

The function body now *looks like the tree*. There is no `Begin`/`End`, no per-widget call, no order to maintain manually — the structure of the literal *is* the structure of the UI.

The mental model when reading this:
- `Window{...}` declares "a window named Settings with these children"
- `.render()` says "now bring it into being"
- Everything between the braces is data, not action

---

## Optional polish

### Cosmetic suffix `_` to drop the `Spec`

If the suffix bothers you, rename the structs (or `using`-alias them):

```cpp
namespace ui {
  using Text     = TextSpec;
  using Slider   = SliderFloatSpec;
  using Slider_i = SliderIntSpec;
  using Check    = CheckboxSpec;
  using Button   = ButtonSpec;
}

ui::Window{"Settings",
  ui::Text  {"Render"},
  ui::Slider{"speed", &speed, 0.f, 10.f},
  // ...
}.render();
```

### A `frame()` engine instead of inline `.render()`

If you want to call all top-level windows in one place:

```cpp
template <typename... W>
void frame(W const&... windows) { (windows.render(), ...); }

// Per-frame:
frame(
  Window{"Settings", /* ... */},
  Window{"Stats",    /* ... */},
  Window{"Tools",    /* ... */}
);
```

### Conditional children

A bare `if` inside the brace list won't work — it's an expression context. Two options:

1. **Filter at construction time** with `std::optional`-style spec wrappers:
   ```cpp
   struct Maybe {
     bool show;
     std::function<void()> body;
     void render() const { if (show) body(); }
   };
   // ...
   Maybe{advanced_visible, [&]{
     SliderFloatSpec{"jitter", &jitter, 0.f, 1.f}.render();
   }}
   ```
2. **Branch outside the literal**: build two different `Window{...}` expressions in an `if`/`else` and call `.render()` on the chosen one. Cleaner for two-state branches; gets unwieldy for more.

A polished wrapper provides a `When{cond, child...}` spec to keep the declarative form intact.

---

## Tradeoffs to mention in the proposal

When proposing this refactor to the user, name these explicitly:

- **Per-frame allocation:** the spec types are stack-allocated POD-ish structs. No heap traffic. The compiler is excellent at eliminating them when `render()` is inlined.
- **Compile time:** templates and tuples cost something. For a typical ImGui app (dozens of windows, thousands of widgets total at compile time) this is invisible.
- **Error messages:** template errors when a spec is missing `render()` are less friendly than runtime errors. A C++20 `concept Renderable = requires (T t, ...) { t.render(); };` constraint on `Window`'s `Children...` makes this concrete: "Children must be Renderable" instead of a 40-line tuple-apply error.
- **Conditional & dynamic UIs:** see above. Static UIs are the sweet spot. UIs whose shape is computed from runtime data want a different structure (a `std::vector<std::variant<...>>` of specs, or virtual dispatch).
- **ImGui's `IDs`:** ImGui uses string-label collisions for widget identity. The declarative form doesn't change that — the same labels still produce the same IDs, in the same order. If you build duplicates dynamically you'll need `PushID`/`PopID`, which you can model as a `Scope{id, children...}` spec.

---

## Mapping this back to your wrapper design

When you start building the actual vkd ImGui wrapper, expect these files:

| File (suggested) | Contains |
|---|---|
| `vkd-ui-spec.cppm` | The widget specs (`Text`, `Button`, `Slider*`, `Checkbox`, `Window`, `Scope`, `When`) |
| `vkd-ui.cppm` | The `frame(...)` engine, top-level rendering loop |
| `vkd-ui-concepts.cppm` | The `Renderable` concept and any other constraints |

Match vkd's existing module-partition style (see `src/vkd-pipeline.cppm` for the builder-pattern precedent already in the codebase — the declarative form is a natural sibling to that style).
