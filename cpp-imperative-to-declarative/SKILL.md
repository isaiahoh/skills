---
name: cpp-imperative-to-declarative
description: >
  Refactor C++ code from imperative procedural style to declarative data-structure style
  using the three-techniques methodology (isolating common code, value parameters, type
  parameters/templates) and the Type+State+tuple+std::apply pattern. Use this skill
  whenever the user asks to refactor imperative C++ into declarative form, design a
  declarative wrapper around an imperative library (ImGui, wxWidgets, Win32, OpenGL,
  Vulkan), turn a sequence of construct-and-configure calls into data, "make this read
  like SwiftUI", build a DSL or builder API in C++, collapse a Begin/End block into a
  tree, or mentions imperative-to-declarative refactoring, declarative UI in C++,
  std::apply over tuples, heterogeneous compile-time collections, or "the code is
  secretly building a tree."
---

# C++ Imperative → Declarative Refactoring

Systematic refactoring workflow grounded in the methodology from [Refactoring an Imperative API into a Declarative One](https://www.youtube.com/watch?v=xu4pI72zlO4) (CppCon).

**Phases: detect → diagnose → propose → apply on confirmation.**

## Guiding Principles

1. **The hidden tree.** Imperative code that repeatedly does *create → configure → attach-to-parent* is secretly building a tree. The procedural form just obscures the structure. Your job is to expose the tree as data. (Talk: 14:15–15:45.)
2. **Separate *what* from *how*.** A declarative form states intent (the tree of widgets, the pipeline spec, the config). Execution (allocation, registration, side effects) belongs in a generic engine. The user's code should read like the *answer*, not the recipe. (Talk: 18:46–20:12.)
3. **Two layers, applied in order.** Layer 1 = the three foundational techniques (isolate common code → value parameters → type parameters). Layer 2 = the Type+State pattern (struct per kind → heterogeneous tuple → `std::apply`). Use Layer 1 first; only escalate to Layer 2 when Layer 1 still leaves repetition or kind-dispatch.
4. **The "leap of faith".** A declarative spec object often exists for the duration of a single expression. That is the point — it is a *description*, not a long-lived object. Don't recoil from "but it's destroyed immediately." (Talk: 19:40–21:50.)
5. **Behavior-preserving.** Refactoring changes structure, not observable behavior. Don't bundle in bug fixes or feature additions. If the code is generating side effects in a specific order, the declarative form must produce the same order.

## Workflow

### Phase 1: Detect

Read the code yourself first. Skim for the smells listed in `references/detection-checklist.md`. The strongest signals:

- A run of ≥3 statements of the form *make X → set fields → attach to parent*
- A `Begin(...) / ... / End(...)` block (ImGui, wxSizer, GL push/pop) where the body is mostly more configure-and-attach calls
- Repeated near-identical struct-init blocks differing only in field values
- A long function that, if you squint, traces out a tree

If the region doesn't match any signal, stop and tell the user this code isn't a candidate. Do not invent a refactoring.

### Phase 2: Diagnose

Read `references/three-techniques.md` and walk the candidate region through the techniques **in order**:

1. **Isolate common code** — extract the shared *create+configure+attach* shape into one helper function. Stop here if the helper alone removes the repetition.
2. **Value parameters** — lift the things that varied (labels, IDs, sizes, callbacks) into function arguments. Stop here if every variation can be expressed as data passed to one helper.
3. **Type parameters (templates)** — when the *kind* of object also varies, template the helper:
   ```cpp
   template <typename W, typename... Args>
   W& create_and_add(Container& c, Args&&... args);
   ```

If after step 3 you still have:
- Heterogeneous kinds that need to live side-by-side in one declaration, or
- A natural tree shape (parents containing children of multiple kinds)

…then escalate. Read `references/type-state-pattern.md` and apply the Type+State+tuple+`std::apply` transformation.

If the target is an ImGui-shaped API, also read `references/imgui-example.md` for the end-to-end worked refactor.

### Phase 3: Propose

Show the user a **before/after diff for one small slice** — never the whole file in the first proposal. Include:

- The chosen layer (1 or 2) and which techniques you applied
- The diff itself
- The "leap of faith" line if Layer 2: state explicitly that the spec structs are short-lived by design
- A note on side-effect order — confirm the new form preserves it
- Any C++ standard requirements (designated initializers need C++20; `std::apply` needs C++17; concepts/CTAD-on-aggregates need C++20)

**Stop and wait for confirmation before editing.** Even an obvious refactor.

### Phase 4: Apply

On confirmation:

1. Apply the proposed slice with `Edit`. Do not expand the scope — refactor exactly what was shown in the proposal.
2. Ask the user how to build/check the change (e.g., `xmake`, `cmake --build`, a specific test target). Do not assume.
3. Verify the build passes. If it doesn't, *stop and diagnose* — do not stack more changes on top of broken code.
4. If there are more candidate regions, propose the next one as a new slice. Refactor one region per round-trip.

If at any point a transformation feels forced — e.g., the Type+State pattern is being applied to code that doesn't actually have a tree shape — say so and back out. Not all imperative code should be declarative.

## Reference Files

- **`references/detection-checklist.md`** — Concrete signals that a region is a candidate, with C++ snippet illustrations. Read in Phase 1.
- **`references/three-techniques.md`** — Worked progression of one toy example through isolate → value-params → type-params, with the diff at each step. Read in Phase 2.
- **`references/type-state-pattern.md`** — The deeper Layer 2 transformation: struct-per-kind, heterogeneous tuple, `std::apply`, the leap of faith, and the C++17/20/23 enablers that make it ergonomic. Read in Phase 2 only when Layer 1 isn't enough.
- **`references/imgui-example.md`** — End-to-end worked refactor of an imperative ImGui block into a declarative tree with `Window{...children}.render()`. Read in Phase 2/3 when the target is ImGui-shaped.
