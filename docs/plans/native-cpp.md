# Native C++

What changes when a recipe's C++ is plain C++: a recipe includes other recipes' headers, C++ objects pass between nodes over connections, and a recipe may include the engine's own headers by hand. Proposed 2026-10-03 for step 13 of the [IDE port](ide-port.md#order), from the lead's direction:

> For C++ to C++ we want to be able to include other C++ files, so we can pass it natively, needing no extra abstractions or deviations. If you understand C++ you should be able to use it inside of Vulpen without restrictions. I guess even baseclasses should be able to be included by hand. Perhaps a boundary at runtime we can consider, as it needs to be clear where things could break.

That is Goal-01: "If you know C++ and GLSL, Vulpen should feel familiar". It changes V11, RV00, RV05 and two rules of [live code](live-code.md#rules), so those change first, as their own step (A05), once the lead decides (A00). Nothing below is built.

## What a person writes

Two recipes in one view. `rects` lays rectangles out in C++; `hit` finds the one under a point, also in C++. The type they share lives in a header of the recipe that writes it:

```cpp
// recipes/rects/Rect.h: a header other recipes include, as any C++ header
#pragma once

#include <glm/vec2.hpp>

#include <vector>

namespace VP_VIEW { // this view's own namespace, which the build names (rule 4)

struct Rect {
  glm::vec2 at;
  glm::vec2 size;
};

using Rects = std::vector<Rect>;

} // namespace VP_VIEW
```

The writer gets the object a connection carries once, in `bind`, and uses it as any object:

```cpp
// recipes/rects/Rects.cpp
#include "rects/Rect.h"
#include "runtime/Operator.h"

namespace {

class Rects final : public VP::Operator {
  VP_VIEW::Rects *_rects = nullptr;

  void bind(VP::Bind &node) override {
    _rects = &node.output<VP_VIEW::Rects>("rects"); // the engine's object, by reference
  }
  void cook(VP::Cook &) override {
    _rects->clear();                                // plain std::vector calls
    _rects->push_back({{10, 10}, {200, 24}});
  }
};

} // namespace

VP_RECIPE(registry) {
  registry.add<Rects>("Rects");
}
```

The reader includes the other recipe's header, as C++ does anywhere:

```cpp
// recipes/hit/Hit.cpp
#include "rects/Rect.h"     // another recipe's header: the view's copy of it (V03)
#include "runtime/Operator.h"

namespace {

class Hit final : public VP::Operator {
  const VP_VIEW::Rects *_rects = nullptr;

  void bind(VP::Bind &node) override {
    _rects = &node.input<VP_VIEW::Rects>("rects"); // const: only the writer changes it
  }
  void cook(VP::Cook &) override {
    for (const VP_VIEW::Rect &rect : *_rects) {    // what rects wrote this frame
      // …
    }
  }
};

} // namespace
```

The manifest joins them, as it joins shader ports today:

```ini
[connection "rects"]
from = rects.rects   # the port the writer's output names
to   = hit.rects     # the port the reader's input names
```

A recipe that needs the engine includes it by hand, and owns what it makes:

```cpp
#include "baseclasses/Engine.h"
#include "baseclasses/Resources.h"

class Scratch final : public VP::Operator {
  std::optional<VP::Buffer> _buffer; // made and destroyed by this operator (A01, CPP09)

  void bind(VP::Bind &node) override {
    const VP::Resources &resources = node.engine().resources(); // borrowed until next bind
    _buffer.emplace(resources.buffer(4096, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
                                     VP::Memory::upload));
  }
};
```

## How it works

- **Includes.** The build adds the view's `recipes/` folder to each recipe's include path, and in the library the `parts/` folder, where all its code lives (RV06). So `#include "rects/Rect.h"` means the same in the library and in a view's copy. Engine headers are on the path already. The compiler's dependency files make every recipe that includes `rects/Rect.h` build again when it changes.
- **C++ connections.** `node.output<T>(port)` and `node.input<T>(port)` hand a reference to one `T` per connection, which the engine owns (A01). The writer binds first, since a connection orders its writer first, and makes the object. Each reader gets it `const`, checked to be the same type. Every rebuild binds every node again, so the references are always fresh. Nothing is looked up during a frame.
- **Engine by hand.** `node.engine()` returns the `Engine`, which `runtime/Operator.h` only declares, so a recipe that does not use it compiles as fast as today. The executable exports its symbols (CMake's `ENABLE_EXPORTS`), so a module can call the engine, which still exists once.
- **C++ and the GPU.** A connection with C++ at one end and a shader at the other keeps the plan of [D5](../architecture/migration-and-implementation.md#open-decisions) and A2: the C++ struct names its members, and the loader checks them against the shader's reflection (RA03).

## The runtime boundary

Where native C++ can break, and what holds instead. The engine checks what it can, loudly (A02); the rest is a rule a person keeps, and the log says when it applies.

| # | Rule | Why it holds | What breaks it | What you see |
| --- | --- | --- | --- | --- |
| 1 | A reference from `bind` lasts until the next `bind` | every rebuild and swap binds every node again before the next frame | keeping it elsewhere: a static, a thread, a lambda that outlives the operator | nothing; C13 forbids the static, and a recipe's thread must join in its destructor |
| 2 | A module swap empties the C++ connections its node writes | the object's destructor is code of that module, so it goes before the module unloads, and is made new at the next `bind` | nothing a person does: this is what a swap costs | `{mod} … swapped; emptied rects` |
| 3 | Writer and readers share one type | the build rebuilds every recipe that includes a changed header, and the swap takes them together; `bind` compares the type's mangled name, size and alignment | a failed build, which swaps nothing | `node hit: input rects is a vp_ui::Rects of 16 bytes, but rects writes …` |
| 4 | Two views never share a type | each view's headers open `namespace VP_VIEW`, which the build names per view, so two views' copies of a recipe (V03) stay two types in one release binary | a header without it, if two views' copies differ: the release linker would keep one definition for both (ODR) | the gate fails the build, naming the header |
| 5 | Recipes share headers, not source files | each module holds its own code, so dev and release link the same way | a non-inline function declared in a shared header and defined in its `.cpp` | the module fails to load, naming the missing symbol |
| 6 | What a recipe makes with the engine, it destroys | RAII in the operator, destroyed before its module unloads | holding an engine object past the next `bind`: the window's render pass changes when it opens or closes, and `bind` runs again then | a validation error, or a crash in a debug build |

Rules 1 and 6 are the boundary a person keeps; the rest the engine and the build keep.

## What changes first

Proposed text, for the lead to approve (A00), as its own commit before any code (A05):

- **V11**: "Two recipes share only the contract on the connection between them" becomes "Two recipes share only what they include and the contract on the connection between them: a GLSL contract for data on the GPU, and a C++ header for data between C++ nodes."
- **RV00**: "Recipes never include each other's code." becomes "A recipe includes another recipe's headers, never its source files, and only from its own view's copies (V03)."
- **RV05** gains: "The C++ type of a connection between C++ nodes is declared in a header of the recipe that writes it; deploying a recipe copies the recipes whose headers it includes, as it copies contracts."
- **[Live code](live-code.md#rules)**:
  - rule 1 gains "Types other recipes include live in `namespace VP_VIEW`."
  - rule 2 becomes "A recipe reaches the engine through `runtime/Operator.h`, or by hand through any engine header, within the runtime boundary."
- **[Include map](../architecture/include-map.md#recipe-code)**: recipe code may also include another recipe's header as `<recipe>/<file>.h`, and any engine header.
- **Memory of the direction**: the 2026-09-29 note says not to bring back direct engine access without asking; this proposal is that asking.

## What it costs

About 150 lines, in four steps, each of which runs:

1. **The principles, the gates and the build**: the text above; the include-map gate's recipe rule; a gate that every header under a `recipes/` folder opens `namespace VP_VIEW` (rule 4, a text search); include folders and `VP_VIEW` per view in `recipe.cmake`; `ENABLE_EXPORTS`. About 40 lines.
2. **C++ connections**: `output<T>` and `input<T>`, one object per connection in the schedule, the type check and the swap that empties. About 90 lines.
3. **Engine by hand**: `node.engine()`. About 10 lines.
4. **`recipe drop`** copies the recipes whose headers a dropped recipe includes. About 10 lines.

## Decisions for the lead (A00)

1. **The principle changes above.**
2. **References, taken once in `bind`**, rather than a handle looked up each frame (`frame.read(handle)`): plain C++, nothing looked up during a frame, and fresh at every `bind` (rule 1).
3. **`VP_VIEW`, one namespace per view**, for every header another recipe includes. The other way is one release binary per view, which the CLI cannot be while it hosts other views.
4. **Headers only** between recipes (rule 5). The other way links each recipe's sources into every module that includes it, which needs the build to know who includes whom.
5. **`node.engine()`** as the one way to the engine's objects, and an exported executable.
