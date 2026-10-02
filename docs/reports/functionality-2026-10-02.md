# Vulpen today

The state of `development` at `e02850f`, on 2026-10-02: steps 1 to 4 of the [IDE port](../plans/ide-port.md#order)'s 25, plus three fixes (A to C). This page covers what Vulpen does, what it is good for now, what it cannot do yet, what was given up on purpose, and how its size compares with the plan. Each point names the [principles](../architecture/principles.md) and [requirements](../architecture/requirements.md) behind it.

## In short

Vulpen runs a graph of GPU passes that a manifest (`.vlp`) describes. Each node runs a recipe: C++ that sets values and moves data, and GLSL that does the work. A connection is a buffer one node writes and others read, and Vulpen places the barriers between them (V10). A view runs headless unless one of its nodes draws (V07). In a debug build, saving a recipe's C++ or GLSL, or the manifest, swaps the change in while the view runs. Every change to the graph is a command (V06), so a script can build a view, change it, record the session, replay it and save the result. There is no interactive front end yet: commands come from a file, before the first frame.

## What it does

### Views and the manifest

- A view is a folder holding a `view.vlp` and its own `recipes/` (V03).
- The manifest states its version (RV03) and has three sections, each with a closed set of words (V04):

  | Section | Words |
  | --- | --- |
  | `[manifest]` | `version` |
  | `[node "name"]` | `recipe`, `operator`, `shader`, `invocations`, `param` (`name=value`, repeatable), `log` |
  | `[connection "name"]` | `from` (`node.port`), `to` (`node.port`, repeatable) |

- A load runs every node and connection through the same edits a command uses, so a loaded view and a typed one pass the same checks (A05, C06).
- A mistake stops the load, naming its file and line (A02). Examples are an unknown word, anything given twice, a port in two connections, a connection that closes a cycle, and a name with characters other than letters, digits, `_` and `-`.

### Recipes: C++ and GLSL

- A node names its recipe, its operator (a C++ class) and its shaders: one `.comp`, or a `.vert` and a `.frag`.
- C++ reaches the engine only through [`runtime/Operator.h`](../../src/runtime/Operator.h). `bind` asks for handles by name, and `cook` runs every frame before the node's pass. The handles:
  - `Value<T>`: a pass-block value the operator sets each frame (`float`, `int`, `uint` and their glm vectors);
  - `Upload<T>`: a buffer the operator writes;
  - `Readback<T>`: a buffer the operator reads, one frame after the GPU wrote it;
  - params, read by name and parsed to their type.
- At load, every name is checked against the shader's SPIR-V reflection (RA03). A node is left out, with an error naming why, if:
  - a pass-block field is set by neither a param nor the operator;
  - a param is read by nobody;
  - a type or element size disagrees between C++ and the shader, or between a connection's two ends.
- The build compiles each view's recipes. Debug builds make one module per recipe; release builds link the recipes into `vulpen`.

### GPU model

- One pipeline layout serves every pipeline (RV02): the pass block at set 1, binding 0, and buffers as 64-bit device addresses (RV01).
- A shader declares each buffer `readonly` or `writeonly`, and the barriers between passes follow from those declarations (V10).
- A connection is one buffer: the `from` port writes it and the `to` ports read it. A written port with no connection gets a buffer of its own.
- A buffer holds as many elements as its writer has invocations. A new buffer starts zeroed.
- A buffer the CPU writes or reads lives in mapped memory; every other buffer stays on the GPU (VK03).
- A dispatch runs `invocations` threads in one dimension. A draw runs `invocations` vertices as one instance, into the window.
- Pass blocks have no limit but memory. A rebuild keeps every operator, pipeline, buffer and pass block that a change left alone, and buffers keep their contents.

### Commands, scripts, the log and save

- One command port serves everything (V06). A command registers with its usage and help, or not at all (RV04). There are 11 today:

  | Command | Registered by |
  | --- | --- |
  | `node add`, `node remove`, `node set`, `connect`, `disconnect`, `param set`, `param unset` | the edits |
  | `view save` | the views |
  | `source`, `log save`, `quit` | the command port |

- `--source FILE` runs a script before the first frame, and `source` runs one from inside a script.
  - A relative path resolves beside the script that names it (RP02).
  - The first line that fails stops the script, naming its file and line (A02).
  - A script that sources itself, directly or through another, is refused.
- The log keeps every edit and every `view save` exactly as typed, one undo group per line (V08). `log save` writes the log flat, and `source` replays it with no recipe needed.
- `view save` writes the manifest to a temp file and renames it over the old one (RA04).
  - The layout is fixed: `[manifest]`, then the nodes, then the connections, keys aligned.
  - Comments come back over the line they were written over, or at the end of the line they ended.
  - A manifest changed on disk since Vulpen read it is not overwritten.
  - A value holding `#` or a line break is refused at the edit, since a manifest could not hold it.
- Edits take effect before the next frame, all at once. A script is checked as a whole, and each error names the file and line that last added or changed its node, for example `edits.txt:7 node p1: values reads nothing`.

### Live code (debug builds)

- Saving any file under the view's folder builds its recipes on a background thread. The change then swaps in between frames, with the GPU idle:
  - a changed shader gets a new pipeline;
  - changed C++ gets a new operator, bound again;
  - a changed manifest is read again.

  Buffers keep their contents through every swap.
- A module built against other engine headers than the running `vulpen` refuses to load, naming why.

### Running

- `vulpen <view.vlp> [--frames N] [--first-frame N] [--fps N] [--log error|warn|info|debug] [--source FILE]`.
- `--first-frame` starts the frame count late, so a test reaches a century of frames in a moment (A03).
- The log is quiet by default (C09). A node's `log` word sets its own level (V09), and the first line names the commit (RC07).
- Output is deterministic (C01):
  - no fast-math and no fused multiply-add (RC06);
  - time follows from the frame index, kept in `double`;
  - the wave example prints the same bits run after run, in debug and in release.

### Measured

| What | Time |
| --- | --: |
| One headless frame of the wave view, lavapipe or RTX 5070 | 33 µs |
| The wave view, start to exit, lavapipe / RTX 5070 | 51 ms / 0.3 s |
| One frame of 1,000 nodes (2026-09-30) | 0.7 ms |
| Load a chain of 1,000 probes, release, lavapipe | 0.17 s |
| Build the same chain as 2,000 script edits | 0.33 s |

### Checks around it

| When | What runs |
| --- | --- |
| Every build | the include-map gate; code rules: no `shared_ptr` or naked `new` (RC03), OS calls only in the platform files (RA01), file writes only through `Files::save` (RA04); floating-point flags (RC06); `spirv-val` on every shader; no mutable globals after the link (C13) |
| `ctest` | the golden test on lavapipe, bit for bit (C01): from frames 0, a day, a year and a century; after edits that take the graph apart and put it back; after replaying their log; and the two saves must write the manifest back byte for byte. Also the triangle in a window, the barrier rule (V10), and 14 broken views that must fail naming their cause (A02) |
| Sanitizers | ASan, UBSan and LSan over every test, 300 mutated manifests and 60 mutated scripts (A03); TSan over six live swaps |
| Nightly | a 10-minute soak and 500 live swaps, both failing on growth (A03); 10,000 mutated manifests and 2,000 mutated scripts under sanitizers |

At `e02850f`, every preset passes: debug, release, asan, tsan (the windowed triangle skipped, with no display) and gpu-validation.

## What it is good for now

1. **Headless GPU compute as a graph.** A batch job is a view run with `--frames N --fps 0`, on a machine with no display (V07). Results leave through the log, from an operator that reads a buffer back, as the wave example's probe does.
2. **Writing shaders and their C++ live.** In a debug build, each save swaps in while the view keeps running, with its buffers kept.
3. **Building and changing views by script.** `--source` a file of edits, then `view save`. Building a 1,000-node chain this way takes 0.33 s.
4. **Bug reports and tests as logs.** `log save` records a session's edits, and `source` replays them headless, needing no recipe (V08).
5. **Golden tests of GPU output.** Output is bit for bit on the reference driver, and within a tolerance on other drivers (C01, GLSL02).
6. **A harness for the engine itself:** sanitizers, two fuzzers, soaks and GPU validation, which every later step runs through.

## What it cannot do yet

### Front end

- **No interactive front end.** There is no terminal, command line, `help` or completion (B9, E12, D1). Usages declare what completes, but nothing completes yet. Commands come only from `--source`, before the first frame.
- **No undo or redo** (B10). The log keeps the groups undo will need.
- **No blanks in typed values.** The port has no quoting, so a value or path with a blank cannot be typed.
- **No comments in scripts.** A `#` line is an unknown command.

### Graph and composition

- **One view per process.** There are no hosted views, deploys or recipe library yet (C3, C1, C2). The parts, components and apps under `src/recipes/` are stubs that do not load.
- **Only views the build compiled.** A view runs only if the build compiled its recipes: a folder in `src/examples/`, or a copy in a folder of the same name.
- **No ports for recipes.** Recipes cannot register commands, read input, files or the terminal, or write files (B5, B6, B7, B9). Those ports are empty classes.
- **No `view migrate`.** Only manifest version 1 exists, so there is nothing to migrate.

### GPU

- **Missing GPU features:**
  - no images, samplers or offscreen targets (A5, A7);
  - no frame block in the push constant, so shaders see no time, resolution or cursor (A1);
  - no depth (A14), instancing (A4) or indirect draws (A11).
- **Limited values.** Pass-block values are scalars and vectors. A param sets only a scalar, and there are no matrices or arrays (A9).
- **One dimension, and draws only read.** Dispatches are one-dimensional. Draws only read buffers, since storing from them needs features beyond the Vulkan floor (RVK01).
- **One frame in flight** (A12), and no GPU timings (A13).
- **No bounds check against the writer.** Reads are not checked against the size the writer set. Only GPU-assisted validation, on the machine's GPU, catches a read past the end (GLSL02).
- **No shared pipelines.** Every node builds its own pipeline, even when it runs the same SPIR-V as another.

### Runtime

- **The window opens only at start.** A draw added later asks for a restart (B14).
- **A failed rebuild ends the run.** Out of GPU memory or a lost device during a rebuild stops vulpen.
- **No clean end on signals.** SIGINT and SIGTERM end the run at once, without destructors or the footer (T31).
- **The log does not yet cover every session:**
  - It has no frame stamps, so a replay matches the session only for commands that ran before the first frame.
  - A live re-read of `view.vlp` is not a command. It drops the edits made since the last read, and the log stops describing the session after it.
  - It has no bound (A03), which matters once a drag sends a command every frame (E7).
- **Rebuilds grow with the square of the graph.** A rebuild looks up nodes and connections by scanning them. That costs little at 1,000 nodes, much more far beyond.

### Platforms and tooling

- **Linux first** (RP00). Windows was built on 2026-09-30 and not tested since. ARM64 and the Vulkan-floor runs are still to come (T35 to T37). Raspberry Pi waits for a fallback tier (C10).
- **Gaps in validation.** GPU-assisted validation crashes lavapipe, so it runs on the machine's GPU only. Synchronization validation cannot see accesses through buffer addresses, so only T17's model of the barrier rule checks barriers.

## What was given up on purpose

| Concession | For | What it costs |
| --- | --- | --- |
| No quoting in the command port; agreed on 2026-10-02 | C00, C04 | a text form for values and paths with blanks (V06) |
| `--source FILE` calls the source function, not the text `source FILE` | V06 in practice | nothing yet: the path never passes through the port's text |
| One rebuild before the next frame, not one per edit | C03, A02: one check of the whole script | each error is reported for the view a batch leaves, naming its node's line rather than the line that caused it; a node removed and added back before the next frame keeps its operator and buffers, as across a re-read of the manifest; a failed rebuild surfaces at the frame, not at its line |
| The log keeps `view save`; agreed on 2026-10-02 | V08 | a replay writes files, so undo must skip saves (B10) |
| A save writes one fixed layout | C04, V04 | the person's layout: section order, alignment and blank lines inside a section. Comments survive |
| A save refuses a manifest changed on disk | A02, RA04 | convenience: the person moves the file aside, or lets the live build read it again |
| `view migrate` waits for version 2; agreed on 2026-10-02 | C00 | nothing until a version 2 exists (RV03) |
| T40 is a gate, not a test that kills a save; agreed on 2026-10-02 | the test plan's cheap-gate rule | a behavioural check of `Files::save` itself, which review covers |
| A kept pass block starts zeroed | C01 | a value an operator sets only once does not survive a rebuild |
| Descriptor pools only grow | A03: memory stays bounded by the peak | memory after a view shrinks |
| Recipes get handles, never Vulkan objects ([live code](../plans/live-code.md)) | safe swaps, V05 | a recipe that needs more gets a general port first |
| Release builds link recipes in and have no live code | C03 | live editing in release |
| Every param must be read and every value set | A02 | defaults, and params kept for later |
| Goldens are bit for bit on one driver only | C10, GLSL02 | other drivers match within 0.001, since each computes `sin` to its own precision |

## Size against the plan

The plan predicts about 10,300 lines of C++, GLSL and `.vlp`: the 3,300-line engine it started from, about 4,900 lines for its core rows and about 2,100 for its later rows. Tests, tools and CMake come on top. It reads each row's estimate as ±50%.

The four steps so far, in lines of code:

| Step | Rows | Estimate | Actual | Ratio |
| --: | --- | --: | --: | --: |
| 1 | B1 | 200 | 142 | 71% |
| 2 | B2, D2 | 200 | 342 | 171% |
| 3 | B3, D3 | 80 | 83 | 104% |
| 4 | B4, without `view migrate` | 150 | 144 | 96% |
| | **Rows** | **630** | **711** | **113%** |
| | Structure: the skeleton and the seams | – | 153 | |
| | Fixes A, B and C | – | 47 | |
| | **Engine growth since the port began** | **630** | **911** | **145%** |

The skeleton also added 180 lines of recipe stubs. Today the engine holds 4,337 lines and the recipes 216, which makes 4,553. The plan's rows still to come are estimated at 6,140: 1,570 of core engine, 2,470 of core recipes and 2,100 of later rows. So:

| If the rest lands at | Total |
| --- | --: |
| its estimates | about 10,700 |
| the rate of the rows so far, 113% | about 11,500 |
| that rate plus fixes like A to C, 120% | about 11,900 |

**So 10,000 no longer holds: about 11,500 is the better guess, between 10,700 and 12,000.** Two things drive the overrun:

- **B2.** Its 342 lines were 142 over its estimate.
- **Unplanned work:** the structure commits and fixes A to C.

The structure commits paid ahead for later rows, so applying the all-in 145% would overstate the rest (about 13,500). The 2,650 lines of parts, components and apps have no track record yet. They hold the largest single estimates: `graph` at 510 and `text` at 500.
