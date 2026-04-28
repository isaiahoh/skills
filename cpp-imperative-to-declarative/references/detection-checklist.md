# Detection Checklist

Concrete signals that a C++ region is a candidate for imperative→declarative refactoring. Treat any single strong signal as enough to enter Phase 2; treat two or more weak signals as enough.

The unifying question: **does this code repeatedly do *create → configure → attach to a parent*?** If yes, it is almost certainly building a tree, and the tree wants to be data.

---

## Strong signals

### 1. Construct + configure + attach, repeated

Three or more consecutive statements that match the shape *make X → set fields → push into parent*.

```cpp
auto* btn1 = new Button("OK");
btn1->set_id(1);
btn1->set_callback(on_ok);
panel->add(btn1);

auto* btn2 = new Button("Cancel");
btn2->set_id(2);
btn2->set_callback(on_cancel);
panel->add(btn2);

auto* btn3 = new Button("Help");
btn3->set_id(3);
btn3->set_callback(on_help);
panel->add(btn3);
```

→ Layer 1 will collapse this to one helper. If the *kind* (Button) also varies later, escalate to Layer 2.

### 2. `Begin(...) / ... / End(...)` blocks

Any push/pop or scope-bracket API where the body is mostly more API calls into the same context.

```cpp
ImGui::Begin("Settings");
  ImGui::Text("Hello");
  ImGui::Button("OK");
  ImGui::SliderFloat("speed", &speed, 0.f, 10.f);
ImGui::End();
```

Strong signal because the indentation already *looks* like a tree but the syntax is flat. Classic in ImGui, wxSizer, OpenGL push/pop matrix, NVG `Save/Restore`, Cairo contexts.

### 3. The hidden tree

A function whose call graph, drawn out, is a tree — but whose source is a flat statement list. If you can answer "what does this function build?" with a tree diagram, the source should look like that diagram.

### 4. Repeated near-identical struct-init blocks

```cpp
VkPipelineShaderStageCreateInfo vs{};
vs.sType  = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
vs.stage  = VK_SHADER_STAGE_VERTEX_BIT;
vs.module = vert_mod;
vs.pName  = "main";

VkPipelineShaderStageCreateInfo fs{};
fs.sType  = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
fs.stage  = VK_SHADER_STAGE_FRAGMENT_BIT;
fs.module = frag_mod;
fs.pName  = "main";
```

Same field set, same order, only values differ. Layer 1 (value parameters) handles this; designated initializers help cosmetically but don't remove the duplication.

---

## Weak signals (need a partner)

### 5. Long parameter lists that always come together

If five fields are always set together with the same set of values "for this kind of thing", they are a Type+State pair waiting to be a struct.

### 6. Comment-driven structure

```cpp
// --- toolbar ---
panel->add(new Button("Open"));
panel->add(new Button("Save"));

// --- main area ---
panel->add(new TextEditor());

// --- statusbar ---
panel->add(new Label("ready"));
```

The comments are doing the job that nesting *should* do. The tree wants out.

### 7. "Setup" and "render" are the same code path

If the same function both decides *what* exists and *how* to bring it into being, separation of declaration from execution will pay off.

---

## Anti-signals — do NOT refactor

Some imperative code should stay imperative. Back out if you see:

- **Order-of-side-effect is the meaning.** A sequence of GL state changes whose interleaving with draw calls is the actual semantics. A declarative tree implies "execute in this order" only if you guarantee it.
- **One-shot, never repeated, ≤5 lines.** Three consecutive lines aren't a tree, they're three lines.
- **Branch-heavy construction.** If half the statements are `if (condition) { ... } else { ... }` over the construction, the data-structure form will reintroduce all those branches at the spec-construction site and gain nothing.
- **The "tree" only has one node.** If the parent only ever holds one child of one kind, Layer 1 step 1 is the answer; do not summon `std::tuple`.

---

## Quick heuristic

Read the candidate region and answer:

1. Could you describe what this code does as a *picture* (a tree, a list, a grid)? → strong candidate
2. Could you describe it as a *recipe* (do X, then Y, then Z, where ordering matters)? → likely not a candidate
3. Both? → it's a tree whose nodes have ordering constraints; refactor to declarative form *and* document the ordering as part of the engine's contract
