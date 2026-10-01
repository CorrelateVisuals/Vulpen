# Live code

How a view's C++ and GLSL are built, loaded and swapped while Vulpen runs, and how a node's manifest entry, C++ and GLSL meet. It proposes an answer to [D2](../architecture/migration-and-implementation.md#open-decisions) and replaces the POC's in-process shader compile, module ABI and bundle machinery. A proof of concept runs in this tree ([below](#proof-of-concept)). Proposed 2026-09-29; the project lead decides (A00).

## Decision

- **The graph is live.** Nodes, connections and params change at runtime (V00, V06, V08). A graph edit re-runs the schedule, never a compiler.
- **The build compiles all code.** CMake compiles every recipe's GLSL to SPIR-V and its C++ to machine code. Nothing compiles inside the running process.
- **A dev build swaps what the build changed.** Recipe C++ builds as one small module per recipe. The runtime notices a saved file, runs the build off the frame thread, and swaps only the shaders and modules the build rewrote; the rest of the view keeps running, GPU buffers included.
- **A release build links the same sources.** Only the CMake target type differs (`MODULE` or `OBJECT`), so dev and release run the same code, and a player is the release build.

## Why

No principle asks for code to change inside a running process, and the POC's way of doing it broke several:

| POC | Breaks |
| --- | --- |
| Loading a view compiles its shaders into `<view>/.vulpen/spv/`, so a load writes files | RA04 |
| `VP_VIEW_EXPORT` is an `#ifdef _WIN32` outside the platform files | RA01 |
| The engine archive linked into both the host and each `.so` gave two copies of every singleton (ABI v3); operators register through static initializers | C13, A01 |
| `VulpenShaderCompile` repeats the rules CMake already compiles shaders by | C02 |
| The fused player, ABI stamp, host marker and prebuilt check exist only to survive the `dlopen` boundary | C00, C04 |

Generations, the app Goal-04 is about, runs none of it: its chain is `constexpr` and CMake compiles its shaders.

For GLSL the swap needs no loader at all: SPIR-V is data, and Vulkan builds a pipeline from it at runtime. For C++ a running process takes new machine code only through the OS loader, so a module is the one way to swap C++ while the rest keeps running. The other routes cost more: a JIT bundles a compiler (RP03), and restarting to replay the log (V08) loses the GPU state a live edit is meant to keep.

## Nodes

A node is one `[node]` entry in the manifest (V00). It names the recipe it comes from (the view's own copy, V03), and in that recipe's folder at most one C++ operator class and its shaders: one compute shader, or a vertex and a fragment shader, which make the node a draw of `invocations` vertices. A view with a draw opens a window; any other runs headless (V07). It holds its params and its log level (V09). Connections are entries of their own, each one buffer from one node's port to other nodes' ports.

```ini
[node "wave"]
recipe      = wave
operator    = Wave
shader      = Wave.comp
invocations = 1024
param       = amplitude=1.0
param       = speed=0.02

[connection "values"]
from = wave.values
to   = probe.values
```

**Names join the three, when the view loads.** Nothing joins them earlier: the build compiles each recipe folder's C++ and GLSL apart and never reads the manifest, so a shader edit never recompiles C++, and the manifest may pair any class with any shader. At load, and again after every swap, the loader reflects the SPIR-V and runs the operator's `bind`, which asks for values, params and read-backs by name. It then checks the three against each other:

- every param is read by the shader or by the operator;
- every value in the pass block is set by a param or by the operator, never both;
- every buffer a shader only reads is connected or written by the operator, never both;
- a node's shaders declare the same pass block, and a draw's shaders only read buffers;
- both ends of a connection agree on the element type and size;
- the invocations fill whole workgroups.

A mismatch is logged with a message that names the node, the name and the fix (A02). At the first load it stops the run before the first frame; after a swap it leaves only that node out, and the rest of the view runs. During a frame nothing is looked up.

**Binding.** `operator = Wave` is looked up in what the node's recipe registered. A recipe's C++ exports one entry, `VP_RECIPE(registry) { registry.add<Wave>("Wave"); }`. A dev build loads the recipe's module from `<build>/views/<view>/recipes/<recipe>/recipe.so` the first time a node needs it and runs that entry; a release build finds the same entry linked in, in a table CMake generates. Names are scoped by recipe, so two recipes may each define a `Text`. `shader = Wave.comp` is `<build>/views/<view>/recipes/<recipe>/Wave.comp.spv`, since the build tree mirrors the view: the path is the binding, with no index to keep (C02).

**Seam: by name, checked at load, with the C++ type checked there too.** `node.value<float>("phase")` fails at load if the shader declares `phase` as a `uint`, or not at all. A compile-time check would need a C++ copy of each GLSL layout, kept by hand (RA03 forbids that) or generated from reflection by the build. Generation would make every recipe's C++ compile depend on its shaders, so a GLSL edit that touches the pass block would recompile and swap C++ too, and it would fix in C++ which shader a class runs with, which the manifest decides at load (V00). RV05 already has C++ read contracts by reflection. A02 accepts a mistake found at load, which comes before the first frame, and the frame pays nothing for it: `bind` turns each name into an offset once. The cost is that the loader, not the compiler, finds a misspelled name, about a second after the save; a build step that loads every view headless, as the `wave` test does, would find it at build.

**Reach: handles it borrows, never an object that owns the GPU.** A recipe's C++ includes only `runtime/Operator.h`. While it binds it gets names in and handles out: a `Value<T>` is an offset, a `Readback<T>` or an `Upload<T>` an index, and a param comes parsed as `T`. `T` is a scalar or, from glm, a vector, and the loader checks it against the GLSL type. Each frame it writes values, fills upload spans, reads a read-back span that is valid for that call, and logs. It never sees `Engine`, `Resources`, `Buffer`, `Pipelines` or a Vulkan handle. The schedule owns every buffer, block and pipeline (A01). A recipe that needs more gets a general port, not the engine (V05). Its module then needs no engine symbol, and a swap cannot leave it holding a pointer to a GPU object the schedule replaced: `bind` resolves its handles again. The engine also places each buffer from the declarations: one an operator writes or reads back lives where the CPU can map it, a first step towards [D5](../architecture/migration-and-implementation.md#open-decisions).

## How it works

- **Build.** [`src/runtime/cmake/recipe.cmake`](../../src/runtime/cmake/recipe.cmake) gives each recipe folder of a view one call. It compiles each shader to `<build>/views/<view>/recipes/<recipe>/`, with a depfile so an edited include recompiles what uses it. It compiles the folder's `.cpp` files to a module (dev) or to objects linked into `vulpen` (release).
- **Load.** The runtime reflects each node's SPIR-V when the view loads, never at build time, so reflection, the GPU layout and pipeline creation never learn where the bytes came from (RA03).
- **Swap.** The runtime scans the view folder every 100 ms. A change seen twice runs `cmake --build --target vulpen_recipes` on a thread, so a half-written file never compiles and the frame loop never waits (VK02). When the build finishes, between frames with the GPU idle:
  - the runtime unloads the modules the build rewrote, after destroying their operators;
  - it reads the manifest again if it changed since it was read, so the edits typed since then stay;
  - it builds a new schedule, which takes over from the old one what the build left alone: operators of recipes whose module stayed, pipelines of unchanged SPIR-V, and buffers of unchanged shape, contents included.

  One path serves a shader swap, a module swap and a graph edit.
- **Failure.** A failed build swaps nothing, and its compiler output is logged, so the running code stays. The include-map gate fails a build as a compile error does. Recipe code has no rows in the map, so a new recipe file swaps in at its next save, and an include the rule for recipe code refuses keeps the running code until a save fixes it. A manifest that does not parse keeps the running graph. A node that fails to bind is left out, with its errors, until a later build fixes it.

An IDE recipe needs no path of its own: it saves through the file port, and the same scan picks the change up, as it does for an external editor.

## Rules

1. **Recipe classes live in an unnamed namespace.** The only exported symbol is the recipe's entry. Any number of recipes, and copies of one recipe in several views, then link into one binary without the ODR clash that D2's note warns of.
2. **A recipe reaches the engine only through the interfaces `runtime/Operator.h` hands it.** Its module then needs no engine symbol, and the engine exists once, in the executable.
3. **GCC compiles recipe modules with `-fno-gnu-unique`.** Without it, a recipe using `<regex>` gets 30 `STB_GNU_UNIQUE` symbols, `dlclose` cannot unload it, and a swap silently keeps running the old code (measured). Every load first asks the OS whether the file is still mapped, with `RTLD_NOLOAD`, so any other cause fails loudly (A02).
4. **Unload before load.** A `dlopen` of a path that is still loaded returns the old code.
5. **A module swap resets its operators' CPU state; a shader swap keeps it.** State that must survive a module swap belongs in params or GPU buffers.
6. **An engine-header edit needs a restart.** The entry's name carries a hash of every engine header, so a module built against other headers fails to load, saying to restart, instead of crashing the host.
7. **The build replaces a module file and never writes into it.** GNU ld unlinks the old file before it writes the new one, so a loaded module keeps its pages while the build runs (measured); a linker that wrote in place would change code under the running host. Windows locks a loaded DLL, so there the build could not replace it at all: the loader maps a copy in the temp folder, one per process and module, deletes it on unload, and before the next load asks the OS whether the copy is still mapped (rule 3).

Two traps. libstdc++ puts the file clock's epoch in the year 2174, so a scan that starts from `file_time_type{}` never sees a change; start from `min()`. And GCC's `-Wextra` flags every designated initializer that leaves fields out, which CPP11 relies on, so the build turns off `-Wmissing-field-initializers`.

## Proof of concept

An engine in `src/baseclasses/` and `src/runtime/` that computes headless and draws into a window, and two views. The first, [`src/examples/wave/`](../../src/examples/wave/view.vlp), runs headless with two recipes that each hold an operator and a shader:

- `wave` keeps a phase on the CPU and turns it into 1,024 values on the GPU. The values glide towards their target, so they live in the buffer across frames.
- `probe` samples eight of the values on the GPU, reads them back and prints them once a second.

```bash
./run.sh src/examples/wave/view.vlp --log info    # then save any file under src/examples/wave/
```

A second view, [`src/examples/triangle/`](../../src/examples/triangle/view.vlp), opens a window and draws. Its one recipe's operator writes a triangle's corners into a buffer and its tint into a `vec4` every frame, and its vertex and fragment shaders only draw them. It swaps like `wave`, on NVIDIA, RADV and llvmpipe, without validation errors, and it runs clean through a swapchain remade every 20 frames (forced in a test build).

Measured on this machine (RTX 5070 Laptop, g++ 13, Make, Debug), from the save to the swap, over three sessions:

| Edit | Save to swap |
| --- | --: |
| `Wave.comp` | 280 to 380 ms |
| `Wave.cpp` | 700 to 780 ms |
| `Probe.cpp`, which includes `<format>` | 1,200 to 1,270 ms |
| `view.vlp`, nothing compiles | 250 to 330 ms |

Of each, 100 to 200 ms is the wait for a change to be seen twice, and 120 ms is the build's fixed cost, 50 of it the include-map gate. The rest is the compiler, so a recipe's includes set its swap time.

Also verified:

- a C++ swap of one recipe leaves the other's operator and state running;
- a shader swap makes a new pipeline but no new buffer, so the buffer's contents carry over;
- a misspelled name is caught when the schedule loads: after a swap only its node drops out, and at the first load the run stops with exit code 1;
- a compile error leaves the running code in place;
- a recipe folder dropped into the running view builds and runs on first use, with no edit to the include map;
- an engine-header edit fails the modules loudly;
- the release build links the same recipes and prints the same values;
- the modules have no undefined engine symbols, no unique symbols, and one export each;
- NVIDIA, AMD (RADV) and llvmpipe print the same values;
- the run is free of validation errors, and an error does reach the log;
- `ctest` passes on both presets;
- on Windows (MSVC 17.14, the Visual Studio generator, NVIDIA), both views run, a C++ and a shader edit swap in, a failed build keeps the running code, no module copy outlives the run, and `ctest` passes on both presets.

When this was proposed, before the window and draws landed, the engine (`src/baseclasses/`, `src/runtime/` with its CMake, and `main.cpp`) was 2,420 lines of code. Of them, 244 existed only so code can change while it runs:

| Where | Lines |
| --- | --: |
| `runtime/Runtime.cpp`: the scan, the build thread, the swap | 101 |
| `runtime/Recipes.h`, `Recipes.cpp`: loading and unloading modules | 55 |
| `baseclasses/Platform.h`, `Platform.cpp`: `Library` | 36 |
| `runtime/cmake/recipe.cmake`: the dev option, header stamp and module target | 30 |
| `runtime/Schedule.cpp`: taking over what a swap left alone | 22 |
| **Total** | **244** |

The POC spends 2,420 lines on the same job, as many as this whole engine, counted the same way: the module loader, ABI stamp, fused player, `vulpen gen`, in-process shader compile, disk watch and layout emitter. That count leaves out the POC's reload orchestration (329 lines), whose counterpart the 244 include, so the comparison favours the POC. On top of that, its player, bundle and export hold 1,645 lines that exist because its editing host and its shipped player were two programs.

## To approve

These change structure, so they wait for the project lead (A00):

- **Include map:** 13 new files and 4 new edges: `main.cpp` → `runtime/Runtime.h`, `runtime/Recipes.h` → `baseclasses/Platform.h` and `runtime/Operator.h`, and `runtime/Schedule.h` → `runtime/Recipes.h`.
- **Manifest words (V04):**
  - `[manifest]` with `version`;
  - `[node]` with `recipe`, `operator`, `shader` (once for a compute shader, twice for a vertex and a fragment shader), `invocations`, `param` and `log`;
  - `[connection]` with `from` and `to`.
- **D2** as above: linked in release, one module per recipe in dev.

## Open

- **RV05.** The example's connection carries plain floats through buffer types in `baseclasses/GpuLayout.glsl`; a struct a connection carries belongs in `recipes/contracts/`.
- **D5.** An operator reads a struct buffer back as its size only; reading its members by reflection is the next step towards operator ends of a connection.
- **V08 in a dev session.** A command log pins the graph, and the commit pins release code, but a session with code swaps runs code no commit holds. Logging each swap with a hash of what it loaded would let a replay refuse to run on different code.
- **Commands (V06).** A swap is not a command: nothing registers commands in this tree yet.
- **Frame block.** The GPU layout's frame block (time, resolution, cursor) does not exist yet, so a draw cannot follow the window's aspect: the triangle stretches with the window.
- **Frames in flight.** One: the CPU waits for each frame before the next, so the frame loop and the GPU never overlap.
- **Gates first.** Recipe targets do not wait for the gates, so a live build compiles while the include-map gate runs; a failing gate still fails the build, and nothing swaps. Making them wait keeps the rule in `CMakeLists.txt` that a broken rule fails the build before any object is built, and adds up to 50 ms to a swap.
- **Two builds on one tree.** The POC took a build lock because two hosts, or a host and a terminal, building one tree at once corrupt it. On Linux, `flock` around the build command covers hosts.
- **Views outside this repo (Goal-03).** A project that attaches Vulpen builds its views with its own CMake, calling the same functions.
