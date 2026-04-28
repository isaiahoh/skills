# The Three Techniques (Layer 1)

The three foundational techniques from the talk (17:42–18:31), applied **in order**. Each one removes a kind of duplication. Stop at the earliest step that removes the duplication you actually have — don't escalate gratuitously.

We'll walk one toy example through all three steps so the progression is concrete: a panel that needs three buttons.

---

## Starting point — fully imperative

```cpp
void make_toolbar(Panel& panel) {
  auto* open = new Button("Open");
  open->set_id(1);
  open->set_callback(on_open);
  panel.add(open);

  auto* save = new Button("Save");
  save->set_id(2);
  save->set_callback(on_save);
  panel.add(save);

  auto* quit = new Button("Quit");
  quit->set_id(3);
  quit->set_callback(on_quit);
  panel.add(quit);
}
```

The duplication: every block does *new Button → set_id → set_callback → panel.add*. Three times.

---

## Technique 1: Isolate Common Code

Extract the shared *create + configure + attach* shape into one helper.

```cpp
void add_button(Panel& panel, const char* label, int id, Callback cb) {
  auto* b = new Button(label);
  b->set_id(id);
  b->set_callback(cb);
  panel.add(b);
}

void make_toolbar(Panel& panel) {
  add_button(panel, "Open", 1, on_open);
  add_button(panel, "Save", 2, on_save);
  add_button(panel, "Quit", 3, on_quit);
}
```

**Diff:** 16 lines → 4. The repetitive scaffolding is gone; what remains is the data (label, id, callback) per row. The call site now reads as a list of buttons, not a recipe for making buttons.

**When to stop here:** every site that needs this exact pattern is a button. If you also need to add labels, sliders, etc. with similar shapes, continue.

---

## Technique 2: Value Parameters

Lift any *value* that varies between similar call sites into a parameter. You already did this for `label`, `id`, `cb` in step 1 — value parameters is the principle behind it. The further move is to lift things you may have left hardcoded:

- A default style that varies by caller: take a `Style` parameter
- A position: take an anchor parameter
- A format string used to build the label: take the format args

```cpp
void add_button(Panel& panel,
                const char* label,
                int id,
                Callback cb,
                Style style = Style::Default) {
  auto* b = new Button(label);
  b->set_id(id);
  b->set_callback(cb);
  b->set_style(style);
  panel.add(b);
}
```

**Principle:** if two call sites differ only by a value, the value belongs in the parameter list. If they differ by *kind*, that's the next step.

**When to stop here:** all the variation between call sites is data. You have one helper, called many ways, with all variation expressed through arguments.

---

## Technique 3: Type Parameters (templates)

When the *kind* of object also varies — sometimes a button, sometimes a label, sometimes a slider — value parameters can't help. The type itself differs. Use a template.

```cpp
template <typename W, typename... Args>
W& create_and_add(Panel& panel, Args&&... args) {
  auto* w = new W(std::forward<Args>(args)...);
  panel.add(w);
  return *w;
}

void make_toolbar(Panel& panel) {
  create_and_add<Button>(panel, "Open").set_callback(on_open);
  create_and_add<Label> (panel, "—");
  create_and_add<Button>(panel, "Save").set_callback(on_save);
  create_and_add<Slider>(panel, 0.f, 1.f);
}
```

**Diff:** one helper now serves every widget kind. The toolbar reads as a list of *what to make*, not how to make each one.

**When to stop here:** the construction code is fully generic, and the call site is a flat sequence of `create_and_add<Kind>(...)` calls. For many APIs (logging facades, plugin registration, simple toolbars) this is the end of the road.

**When to escalate:** the call site is *still* a flat sequence of side-effect statements when what you actually mean is a *tree* (children inside a window, rows inside a table, items inside a menu). Then the data wants to be a value, not a sequence of calls. Read `type-state-pattern.md`.

---

## Why this order matters

The three techniques attack different axes:

| Step | What varies between call sites | What's removed |
|------|--------------------------------|----------------|
| 1. Isolate | Nothing — pure scaffolding | The repeated create/configure/attach lines |
| 2. Value params | Configuration values | Hardcoded literals at the call site |
| 3. Type params | The *kind* of object | The kind-specific helpers (one per type) |

Applying them out of order doesn't break anything but produces awkward intermediates. Step 1 first gives you something to parameterize. Step 2 second gives you a generic-on-data function. Step 3 last gives you a generic-on-type-too function.

---

## What Layer 1 cannot do

- Express **nesting**. `create_and_add<Window>(panel, ...)` doesn't naturally give you a way to also describe the window's children in the same expression.
- Express **heterogeneous siblings as data**. The call site is still a sequence of statements, not a value you can store, pass, or generate.
- Separate **declaration from execution**. Even with templates, calling `create_and_add` *is* the execution.

If you need any of these, escalate to Layer 2 (`type-state-pattern.md`).
