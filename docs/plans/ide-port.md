# IDE port

What to port from the POC's working IDE (`vulpen/` in the Generations repo, `ide-full`) into this tree, split by function, with what each costs. The three asks:

1. the panels as parts, and the full IDE as the app the [recipe map](../architecture/recipe-map.md) sketches;
2. the IDE's CLI commands, one row per function, so each can be judged on its own;
3. the base that lets C++ and GLSL share every kind of Vulkan buffer and image.

Proposed 2026-09-30. It is a proposal: the project lead ticks what gets ported (A00). The ticks already on the page are the core rows, placed in the [order](#order) they would land. The POC's full code accounting is in [migration and implementation](../architecture/migration-and-implementation.md#code-accounting); this page only covers what the IDE needs. [CLI examples](cli-examples.md) walks two views through these rows.

## How to read the tables

- **Port**: `[x]` ports the row and `[ ]` does not. The core rows are ticked, for review.
- **POC**: lines of code in the POC for that function, counted by the rule in [`fetch-statistics.py`](../../src/tools/fetch-statistics.py). Measured at `bc0f4988`. A dash means the POC has no separate code for it.
- **New**: estimated lines of code in this tree, counting C++, GLSL and `.vlp` together. The estimates follow this tree's density. Its path from the manifest to GPU passes is about 1,500 lines where the POC's is about 8,500, and its live swap is 244 where the POC's is 2,420. Read each estimate as ±50%.
- **Needs**: rows or [open decisions](../architecture/migration-and-implementation.md#open-decisions) that must land first.
- **Advice**:
  - **core**: the IDE does not work without it;
  - **later**: useful once the IDE runs;
  - **cut**: not ported, and the row says why.

## Summary

| Section | Core, new | Later, new | POC |
| --- | --: | --: | --: |
| [GPU data](#1-gpu-data-c-and-glsl-share-every-buffer) | 720 | 590 | 3,757 |
| [Ports](#2-ports) | 1,020 | 260 | 7,212 |
| [Composition](#3-composition) | 550 | 0 | 2,938 |
| [Commands](#4-commands) | 185 | 660 | 3,477 |
| [Parts](#5-parts) | 2,000 | 220 | 11,680 |
| [Components and apps](#6-components-and-apps) | 400 | 70 | 4,163 |
| [Tooling](#7-tooling) | 0 | 300 | 1,491 |
| **Total** | **about 4,900** | **about 2,100** | **about 34,700** |

The POC column adds up each section's rows. Where one POC module serves two rows, only one of them counts it. The Commands row counts only the verb handlers that no port or part row already counts.

The engine today (`baseclasses/`, `runtime/` and `main.cpp`) is about 3,300 lines of C++. The core rows add about 2,200 lines to it: sections 1 and 2, and the deploys (C1) and hosted views (C3) of section 3. The other 2,650 are recipes.

## Where the lines go

- **No IDE in the engine.** The POC's engine holds 10,354 lines of IDE and input code, reached through capability structs (26 in all, many of them only for the IDE). It includes the dock (2,228), the node canvas (2,087), the text widget (1,551), relations (822), input (763), hover (521) and the popup (398). Here the engine only gains ports (section 2, about 1,000 lines), and every panel is a part built on them (V05).
- **One copy of each job.** The POC's panel chrome is the same file in four recipes, plus a vendored copy in `ide-full`. Its text panel is the same file in three places (`ide-text-panel`, `ide-terminal` and `ide-full`). Here each is one part that components deploy (V11).
- **143 verbs become 36 core commands and 19 later ones.** 61 verbs are cut because this design removes the reason they existed:
  - the layout verbs, because reflection gives the layout;
  - the reload verbs, because the live build swaps changes in;
  - the nest and dock verbs, because deploys and params replace them.

  Six verbs that fake input for tests become one `input` command.
- **The POC's 13 channel kinds become four things a shader declares**: a value, a buffer, an image and the frame block. The POC's path from the manifest to GPU passes is `Chain/` (4,614), bindless passes (2,003), images (1,061) and resources (716). This tree's path is about 1,500 lines, and rows A1–A7 add about 720 more.
- **No central command table.** The POC keeps each verb's spec in `command_spec.cpp` and its wiring in `commands.cpp`, 997 lines between them. Here a command registers with its spec where it is handled (RV04), and parts register their own commands.
- **Layout by params.** A dock seam is a param of `split`, so `param set` moves it and replaying the log puts it back. Tearing tabs out and dragging them between panels is a later row (D18).
- **Monospace text.** A glyph's place is its index times the font's advance, computed on the GPU, so no part shapes text on the CPU (C00).
- **`runtime/Operator.h` stays the only engine header most recipes include.** It sets how fast a C++ swap is ([handoff 2026-09-30](../logs/development-20260930T075251Z.md)), so each port goes in as a small interface there and brings no heavy standard header with it. A part that reads the graph also includes `runtime/View.h` (B8), which includes nothing of ours. Recipe code has no rows in the include map; one rule checks it ([CLI examples](cli-examples.md#recipe-code-has-no-rows-in-the-include-map)).

## 1. GPU data: C++ and GLSL share every buffer

This tree already has:

- pass-block values (`Value<T>`), and params written into the pass block;
- buffers as device addresses, whose read or write access the shader's qualifier declares;
- connections, `Upload<T>` and `Readback<T>`;
- barriers placed from what each pass reads and writes (V10);
- name and type checks against reflection at load;
- swaps that keep buffers and their contents.

Of the POC's 13 channel kinds, `UBO`, `SSBO`, `STAGING_UPLOAD` and `READBACK` are covered by the list above. `PUSH` becomes A1, the two image kinds A5 and A10, `VIEW_RENDER` A7 and `INDIRECT` A11. `VERTEX_ATTR`, `INDEX`, `WORKGROUP_SHARED` and `TRIGGER` are cut (A16, A17).

| Port | ID | Function | POC | New | Needs | Advice |
| --- | --- | --- | --: | --: | --- | --- |
| [x] | A1 | Frame block (time, frame index, resolution, cursor): a buffer whose address goes in the push constant, as the GPU layout says (RV02). Every UI draw needs it to map pixels. `Pipelines` owns the one block and takes its layout from reflection (RA03); the engine writes it after acquiring the window's image, so a resize shows at once. Time counts frames at the run's rate, so a replay matches (C01); the cursor stays zero until B6. | – | 40 | — | core |
| [x] | A2 | Struct elements: a C++ struct names its members once, and the loader checks their names, types and offsets against reflection. Today only the size is checked. Contracts need this (RV05). | 198 (emitter) | 100 | — | core |
| [x] | A3 | CPU ends of a connection: one operator writes a buffer and a later operator reads it in the same frame, and the buffer lives where both can reach it. This is the first step of D5. The POC used a typed publish port per feature instead. | – | 80 | A2, D5 | core |
| [x] | A4 | Counts per frame: a draw runs `instance_count` instances, a number or a port whose buffer's used length sets it, and the buffer's writer sets that length each frame. `invocations` then counts the vertices of one instance. Text and rects change length every frame. The word, approved on 2026-10-03, takes a number today; the port form waits for A3. | – | 40 | A3 | core |
| [x] | A5 | Images: the set 0 arrays (`texture2D[]`, `sampler[]`, `image2D[]`), images the schedule owns (A01), a `Texture` handle in the pass block that the loader fills from a connection, and a one-time upload from the CPU. | 1,061 | 250 | — | core |
| [x] | A6 | One blend mode: every draw blends premultiplied `over`. Draws already run in graph order, then manifest order, so no order word is needed. | – | 10 | — | core |
| [x] | A7 | Offscreen targets: a draw's fragment output is a port. Connected to another node's `Texture`, the draw renders into an image the size of the window. Unconnected, it renders into the window, so which node reaches the screen stays a manifest fact. The Perform panel and child views show their output this way. | 365 | 200 | A5 | core |
| [ ] | A8 | Image readback: a frame or a target read back asynchronously (VK03). A part writes it to PNG and registers `screenshot` and `capture`. GUI tests compare these with goldens. | 160 | 80 | A5 | later |
| [ ] | A9 | Matrix, `bool` and array values in the pass block; vector params (`param = tint=1 0 0 1`). Example 2 sets a `mat4` from C++. | – | 30 | — | later |
| [ ] | A10 | Storage images: a compute shader writes an `image2D`. | (in A5's) | 40 | A5 | later |
| [ ] | A11 | Indirect draws and dispatches, with counts a shader writes. | – | 50 | A4 | later |
| [ ] | A12 | More than one frame in flight: blocks, uploads and readbacks per frame (VK02). | 338 | 120 | — | later |
| [ ] | A13 | GPU timestamps per pass (C03). | 417 | 80 | — | later |
| [ ] | A14 | Depth: every target gets a depth attachment, tested less-or-equal, so flat draws are unaffected and no manifest word is needed. Example 2's cubes need it. | – | 40 | — | later, for 3D recipes |
| [ ] | A15 | Image and mesh loaders. | 395 | 150 | A5 | later, as recipes |
| [ ] | A16 | Vertex and index buffers, vertex input layouts, mesh presets. | 496 + | 0 | — | cut: shaders fetch vertices through buffer addresses, as the triangle already does |
| [ ] | A17 | Channel kind, cadence, storage and layout as manifest words; channel presets. | 327 + | 0 | — | cut: the shader declares its data and reflection finds it (RA03), and memory follows who reads and writes it. Shared memory is plain GLSL, and triggers are commands |

## 2. Ports

The engine's general ports ([recipe map](../architecture/recipe-map.md#core-ports)). Every part is built on these, so they come first.

| Port | ID | Function | POC | New | Needs | Advice |
| --- | --- | --- | --: | --: | --- | --- |
| [x] | B1 | Command port: a command registers with its usage, help and completion, or it does not register at all (RV04). The port dispatches text, lists what is registered, and provides `quit`. | 1,424 | 200 | — | core |
| [x] | B2 | Primitive edits, which are the manifest's own words: `node add`, `node remove`, `node set`, `connect`, `disconnect`, `param set` and `param unset`. Loading a `.vlp` runs its lines through these. The schedule reruns on the swap path before the next frame, once for all the edits since the last, so a script is checked as a whole, and each error names the line that last added or changed its node. | 2,597 | 200 | B1 | core |
| [x] | B3 | Command log in groups: `source <log>` replays a log and `log save <file>` writes one (V08). | 142 | 80 | B1 | core |
| [x] | B4 | Save: the manifest writer, which writes a temp file and renames it (RA04) and keeps the comments a person wrote. Adds `view save`. `view migrate` lands with the manifest's version 2, since no older manifest exists to migrate (RV03, C00). | 522 | 150 | B2 | core |
| [x] | B5 | Recipe commands: an operator registers its own commands in `bind` (V05). They last while the node runs: each rebuild registers them again, and a node that goes, is swapped or stops in error takes them along, so the port never calls code that is gone. A name stays one command, so a second node registering it is refused. | 797 (not ported) | 60 | B1 | core |
| [x] | B6 | Input port: keys, text, pointer, wheel and focus. The `input` command produces the same events without a window (V07). | 763 | 130 | B1 | core |
| [x] | B7 | File port: read, watch and save (RA04). `node.file(path)` in `bind` opens a file the node holds, in its recipe's folder when the path is relative (RP02): `text(file)` reads it again once it changed on disk, and `save(file, text)` writes it. A rebuild's open shares the file, so the set stays bounded (A03). `read`, `save` and `list` by absolute path serve the files a command names; a `<file>` argument arrives resolved, beside the script that runs it. | 586 | 80 | — | core |
| [x] | B8 | Graph reads (D1): a command's handler reads the `const View&` it addresses, `call.view()`, as the edits so far left it: what a save would write. It replaces the POC's graph mirror, which is part of E13's figure. Reading it during cook waits for its first user, the graph part (E13); a node never keeps it, since a rebuild replaces it. | (in E13's) | 30 | D1 | core |
| [x] | B9 | Terminal port: lines from stdin, text to stdout. It reads standard input without a thread, once a frame and only when a node asks, so nothing waits on it at exit and a run in the background never reads the terminal. A node sends a line to the command port and gets back what its command answers. B11 gives it a thread of its own. | 279 | 60 | — | core |
| [x] | B14 | Window on demand: an edit that adds the first draw opens the window, and one that removes the last draw closes it. The instance takes every surface extension the loader offers and the device its swapchain where it has one, so a run that starts headless can open a window; a window at start still steers the choice of GPU. | – | 30 | B2 | core |
| [ ] | B10 | Undo and redo, one log group at a time. The log keeps `view save`, so a rebuild for undo skips saves. | 83 | 80 | B3 | later |
| [ ] | B11 | Terminal raw mode, so TAB completion works in a shell. The port reads on a thread of its own, as the Windows console needs too; its interface stays, so no recipe changes (the lead, 2026-10-03). | (in B9's) | 80 | B9 | later |
| [ ] | B12 | Clipboard, through GLFW. | 19 | 20 | B6 | later |
| [ ] | B13 | Named command blocks in the `.vlp`: the manifest word decided on 2026-09-27. | – | 80 | B1 | later, until a recipe needs one |

## 3. Composition

Components and apps are only a `view.vlp` that deploys other recipes (RV06), and the CLI and the IDE both edit a project that is not themselves (V03). Both need the rows below.

| Port | ID | Function | POC | New | Needs | Advice |
| --- | --- | --- | --: | --: | --- | --- |
| [x] | C1 | Deploy: a `view.vlp` deploys a recipe in a `[deploy]` section, sets params there (`param = <node>.<key>=<value>`), and connects its ports as `<deploy>.<node>.<port>`. The loader flattens the result: a deploy brings the nodes and connections of its recipe's `view.vlp`, from the view's `recipes/`, named `<deploy>.<name>`, and a recipe deploys others but never itself (RV06). `deploy add` and `deploy remove` join the primitive edits, and `param set` on `<deploy>.<node>` sets the deploy's param. The build compiles the library's recipes as one view, `library`, so an app runs from the library: a recipe of the library run as a view finds the recipes it deploys in the library's kind folders. No component or app loads without this. | 290 + 628 | 270 | a manifest word (V04) | core |
| [x] | C2 | `recipe list` and `recipe drop <recipe> <name>`, registered by the part `library` (E18). A drop copies a library recipe into the view with the recipes it deploys and the contracts they include (V02, V03), and deploys it, in one log group. A copy the view already has stays, since the view owns it. | 750 | 80 | B2, B7, C1 | core |
| [x] | C3 | Hosted views: a view hosts another view (V03), so the CLI and the IDE host the project they edit. Adds `child add`, `child remove` and `child list`. Hosting is session state, kept in the log and not in the host's manifest. A line `<name>: <command>` reaches a child, `:` the host, and a line a command sends reaches the view of the line that ran it. A child whose `view.vlp` is not written yet starts empty, and `view save` writes it, so a log replays from an empty folder. Each frame cooks the host, then the children in the order added, and the live scan watches every hosted view's folder. A child's draws reach the window unless A7 routes them into an image. | 395 | 200 | C1 | core |
| [ ] | C4 | The nest tree: `promote`, `demote`, `group`, `wrap`, `merge`, `split`, `ungroup`, `expose`, `cd`, `pwd`. | 875 | 0 | — | cut: deploys and folders give the structure |

## 4. Commands

Each POC verb, 143 in total, is on exactly one row below. In the POC they live in `runtime/src/operators/cli/` (11,921 lines). The port, composition and part rows already count most of it: the machinery (B1, E12), the graph edits (B2), history and undo (B3, B10), recipes (C1, C2), nesting (C4) and `ls` (E17). The other 3,477 lines are counted in this section's summary row: export (1,152), the engine verbs (596), wiring (518), the shell verbs (482), probe (299), the operator scaffold (213), dock layout (141) and the `lib` listings (76).

The engine registers only the primitive edits, the log, save and migrate, hosting, input and `quit`. Every other command is a recipe's, as decided on 2026-09-27 ([migration and implementation](../architecture/migration-and-implementation.md#commands-the-engine-keeps-the-mechanism-recipes-bring-the-vocabulary)). Where a row's cost is part of another row (for example "in B2"), the other row counts it.

| Port | ID | Command | POC verbs it replaces | Registered by | New | Needs | Advice |
| --- | --- | --- | --- | --- | --: | --- | --- |
| [x] | D1 | `help`, `complete`, `quit` | `help`, `complete`, `quit` | part `command-line` (`help`, `complete`); engine (`quit`) | in B1, E12 | B1, E12 | core |
| [x] | D2 | `node add/remove/set`, `connect`, `disconnect`, `param set/unset` | `node new`, `node remove`, `node bind`, `wire connect`, `connect`, `wire disconnect`, `param set`, `param unset` | engine | in B2 | B2 | core |
| [x] | D3 | `source`, `log save` | `source` | engine | in B3 | B3 | core |
| [x] | D4 | `view new <folder>`, `view load <folder>`, `view save`: new and load host the view in a folder, named for it | `view new`, `view load`, `view save`, `view close` | part `library` (`new`, `load`); engine (`save`) | 40 | B4, B7, C3 | core |
| [x] | D5 | `recipe list`, `recipe drop` | `recipe list`, `recipe drop`, `lib operators` | part `library` | in C2 | C2 | core |
| [x] | D6 | `child add/remove/list` | `child mount`, `child unmount`, `child list` | engine | in C3 | C3 | core |
| [x] | D7 | `ls`, `info <node>`: nodes, params, connections, pass-block fields and errors | `ls`, `node info`, `param list`, `wire show`, `select find`, `present info` | part `inspect` | 80 | B8 | core |
| [x] | D8 | `open`, `write`, `close`, `close!`, `find` | `open`, `write`, `close`, `close!`, `edit`, `panel select`, `panel type` | part `text` | in E11 | E11 | core |
| [x] | D9 | `present <node>`, `mode <name>` | `present`, `present frame`, `present status`, `mode` | part `modes` | in E14 | E14 | core |
| [x] | D10 | `panel open/close`, `tab add/close/select` | `panel list`, `panel open`, `panel close`, `panel tab add`, `panel tab close` | parts `split`, `list` | 60 | E7, E8 | core |
| [x] | D11 | `input key/text/pointer/wheel …` | `click`, `hover`, `popup open`, `popup list`, `popup pick`, `popup close` | engine | in B6 | B6 | core |
| [x] | D12 | `clear` | `clear` | part `command-line` | 5 | E12 | core |
| [ ] | D13 | `undo`, `redo` | `undo`, `redo` | engine | in B10 | B10 | later |
| [ ] | D14 | `schedule`: the passes, what each reads and writes, and sizes | `engine flatten`, `engine audit chain`, `probe passes` | part `inspect` | 40 | D7 | later |
| [x] | D15 | `recipe new draw <name>`, `recipe new dispatch <name>`: a recipe folder with its C++ and shaders, from the [template](cli-examples.md#a-recipe-file-shows-what-it-can-reach), every line of its menu commented. The kind is a word of the command, so completion and `help` show both | `file new`, `operator new`, `operator remove` | part `library` | 80 | E18 | core (the lead, 2026-10-03): the template is how a recipe starts |
| [ ] | D16 | `node rename` (a primitive: it renames a section and every endpoint that names it), `node duplicate`, `file rename` (renames the file and its `shader` word in one group) | `node rename`, `node duplicate`, `file rename`, `file move` | engine (`node rename`); part `library` (the others) | 50 | B2, E18 | later |
| [ ] | D17 | `recipe publish`: copy a view's recipe into the library, with a `view.vlp` for its nodes | `recipe save`, `recipe publish`, `recipe import` | part `library` | 50 | C2 | later |
| [ ] | D18 | `tab split`, `tab merge`, `panel move`: tearing out tabs and dragging them between panels | `panel tab split`, `panel tab merge`, `panel move`, `panel place` | part `split` | 250 | D10 | later |
| [ ] | D19 | `select` | `select add`, `select clear` | part `graph` | 30 | E13 | later |
| [ ] | D20 | `relations <kinds>` | `relations`, `tiles`, `route`, `deps` | part `relations` | 30 | E16 | later |
| [ ] | D21 | `keys` | `keys` | part `keys` | 20 | E10 | later |
| [ ] | D22 | `probe watch <connection>`: deploys a probe node on the connection, as the wave example's probe, and prints the min, max, NaN count and hash of what it reads back | `probe watch` | part `inspect` | 50 | C2 | later |
| [ ] | D23 | `heat`, `probe stats` | `heat`, `probe stats` | parts `graph`, `inspect` | 60 | A13 | later |
| [ ] | D24 | `screenshot`, `capture` | `engine screenshot`, `probe capture` | a part (in A8) | in A8 | A8 | later |

Core: the 36 commands in D1–D12 replace 51 POC verbs. Later: 19 commands replace 31 more. The remaining 61 are cut:

| Port | ID | POC verbs | Why they are cut |
| --- | --- | --- | --- |
| [ ] | X1 | `schema struct`, `schema ubo-field`, `schema push-field`, `wire channel`, `wire preset`, `mesh preset`, `lib presets`, `lib meshes` | The shader is the only place a layout is defined (RA03), and reflection finds its kind and size (A17). |
| [ ] | X2 | `node hook`, `node phase`, `engine cook`, `engine dispatch` | An operator has `bind` and `cook` and nothing else; an event is a command. |
| [ ] | X3 | `render depth`, `render cull`, `render order`, `render blend` | There is one blend mode and draws follow graph order (A6). Depth needs no word (A14). |
| [ ] | X4 | `engine compile`, `engine rebuild`, `engine view`, `view reload` (×4), `recipe prebuild`, `view export`, `view altitude` | The build compiles and the live scan swaps changes in. A player is the release build ([live code](live-code.md)). |
| [ ] | X5 | `engine audit lint` | The loader is the linter (A02). |
| [ ] | X6 | `record`, `record stop`, `census`, `probe off`, `panel debug`, `busy` | The log is always kept (B3). `census` belongs to the A03 soak test, and the rest are aids for capturing screenshots. |
| [ ] | X7 | `nest` (×9), `wire expose`, `cd`, `pwd`, `dock` (×4), `mode new`, `mode default`, `mode remove` | Deploys (C1) and folders give the structure; seams and modes are params. |
| [ ] | X8 | `stream` (×9) | Not part of the IDE; these come back with streams (D4). |

## 5. Parts

Parts are the panels' building blocks, with the jobs the [recipe map](../architecture/recipe-map.md#parts) gives them. `inspect` and `library` are new. D7, D14 and D22 need a part that reads the graph, and C2, D4, D15, D16 and D17 need a part that works on recipe and view folders. Both apps deploy both parts, and adding them changes the recipe map (A00).

| Port | ID | Part | Job | POC | New | Needs | Advice |
| --- | --- | --- | --- | --: | --: | --- | --- |
| [x] | E1 | `rects` | draws every Rect in a palette color | 218 | 40 | A1, A2, A4, A6 | core |
| [x] | E2 | `glyphs` | lays out and draws every Label in the font; monospace, so layout happens entirely in the shader | 342 | 150 | A1, A4, A5, A6, E5 | core |
| [x] | E3 | `curves` | draws every Curve: wires and relations | (in grid's 669) | 60 | A1, A4, A6 | core |
| [x] | E4 | `image` | draws one image into a Rect, letterboxed | 78 | 40 | A1, A5, A7 | core |
| [x] | E5 | `font` | bakes the glyph atlas once, with the vendored stb_truetype | 313 | 100 | A5, B7 | core |
| [x] | E6 | `palette` | reads `theme.ini` and publishes a Palette | 143 | 50 | B7 | core |
| [x] | E7 | `split` | divides a Rect by a tree of ratios; dragging a seam sends `param set` | (in dock's 2,228) | 80 | A2, A3, B2 | core |
| [x] | E8 | `list` | lays out Items in a row or a column, in a Rect or at an anchor: tab strips, the menubar, popups, completions | 149 | 80 | A2, A3 | core |
| [x] | E9 | `hit` | finds the Rect under the pointer: a press sends its Item's command, and hovering names the Item | 133 + 521 | 70 | A3, B1, B6 | core |
| [x] | E10 | `keys` | turns keymap chords into command text, and sends other keys to the focused part | 517 | 120 | B1, B6, B7 | core |
| [x] | E11 | `text` | holds the buffer, caret, selection and scroll; opens, writes, closes and finds through the file port; writes Labels and Rects | 1,475 + 467 | 500 | A2, A3, B1, B5, B6, B7 | core |
| [x] | E12 | `command-line` | one line with history and completion, sent to the command port and addressed to the view hosted most recently, as `view new` and `view load` leave it, unless the line names one; registers `help`, `complete` and `clear`; on the terminal port it is the CLI, and in a window it reads the input port | 1,227 | 150 | B1, B5, B9 | core |
| [x] | E13 | `graph` | nodes as Rects and Labels, connections as Curves, with pan and zoom; a drag sends a command | 882 + 669 + 2,087 | 510 | A2, A3, B1, B6, B8, E3 | core |
| [x] | E14 | `modes` | chooses which node reaches the screen: `present`, `mode` | 332 | 50 | B1, B2, B5 | core |
| [x] | E17 | `inspect` | reads the graph and registers `ls`, `info`, `schedule` and `probe watch` | 582 | in D7 | B5, B8 | core |
| [x] | E18 | `library` | works on recipe and view folders through the file port, and registers `recipe list`, `recipe drop`, `recipe publish`, `recipe new`, `view new`, `view load`, `node duplicate` and `file rename` | (in C2's) | in C2, D4 | B5, B7, C1 | core |
| [ ] | E15 | `command-items` | offers the commands for a target as Items, for menus and the popup | 398 | 60 | B1, B8 | later |
| [ ] | E16 | `relations` | finds CMake, doc and include relations and publishes them as Relations | 325 + 822 | 160 | B7 | later |

## 6. Components and apps

A component is only a `view.vlp` (RV06), so it costs manifest lines, and it can load only once C1 exists.

| Port | ID | Recipe | Deploys | POC | New | Needs | Advice |
| --- | --- | --- | --- | --: | --: | --- | --- |
| [x] | F1 | `panel` | list, hit, rects, glyphs | chrome, in five copies (in E1's and E8's) | 30 | C1, E1, E2, E8, E9 | core |
| [x] | F2 | `dock` | split, hit, rects | 2,228 (the POC's runtime dock, which E7 and D18 replace) | 20 | C1, E7, E9 | core |
| [x] | F3 | `text-area` | text, rects, glyphs | text panel, in three copies (in E11's) | 20 | E11 | core |
| [x] | F4 | `terminal` | command-line, list, rects, glyphs | the same text panel | 25 | B6, E12 | core |
| [x] | F5 | `graph-editor` | graph, rects, glyphs, curves; relations once E16 lands | node grid (in E13's) | 30 | E13 | core |
| [ ] | F6 | `menu` | command-items, list, hit, rects, glyphs | 298 | 30 | E15 | later |
| [ ] | F7 | `tooltip` | list, rects, glyphs | 112 | 20 | E8 | later |
| [ ] | F8 | `find-bar` | command-line, rects, glyphs | 20 | 20 | E11, E12 | later |
| [x] | G1 | app `cli` | command-line, inspect and library, on the terminal port, with no window of its own | – | 15 | B1–B5, B7, B9, C1, C3, E12, E17, E18 | core |
| [x] | G2 | app `ide` | palette, font, keys, modes, image, inspect, library; dock, panel ×4, text-area, terminal, graph-editor; the edited project as a hosted view | 1,438 | 200 | C1, C3, every core part | core |
| [x] | G3 | `theme.ini`, `keymap.ini` | the POC's files, cut down to the keys a part actually reads (A02) | 67 | 60 | E6, E10 | core |

## 7. Tooling

| Port | ID | Function | POC | New | Needs | Advice |
| --- | --- | --- | --: | --: | --- | --- |
| [ ] | H1 | GUI tests as command logs: `input` lines followed by `screenshot`, compared with a golden. They replace `panel_smoke.py`, `panel_cycle.py` and `cli_smoke.py`. | 972 | 120 | A8, B3, B6 | later |
| [ ] | H2 | Recipe-map gate: checks the deploys in each `view.vlp` against the recipe map, the way `include-map.py` checks includes. | – | 80 | C1 | later |
| [ ] | H3 | Random command sequences that check determinism (C01). | 519 | 100 | B3 | later |
| [ ] | H4 | Command-table gate: checks that every verb has a spec row. | in `check_structural.py` | 0 | — | cut: B1 refuses a command without its spec (RV04, A02) |

## Order

The ticked rows, in the order they would land. Each step needs only rows from earlier steps or its own, and each phase ends in something that runs. A command row lands with the row that counts its cost.

| Phase | Step | Rows | What runs after it |
| --- | --: | --- | --- |
| 1. CLI, headless (about 1,730) | 1 | B1 | the command port: a script of commands runs headless (`--source FILE`) |
| | 2 | B2, D2 | primitive edits; a `.vlp` loads through them |
| | 3 | B3, D3 | the log in groups; `log save` writes it and `source` replays it |
| | 4 | B4 | save |
| | 5 | B14 | the first draw opens the window |
| | 6 | B5 | operators register commands |
| | 7 | C1 | deploys |
| | 8 | B9, E12, D1, D12 | the terminal port and the command-line part, which the build compiles from the library |
| | 9 | B8, E17, D7 | graph reads; `ls` and `info` |
| | 10 | B7, C3, E18, C2, D4, D5, D6, D15 | the file port, hosted views and the library part |
| | 11 | G1 | the `cli` app. [Example 1](cli-examples.md#example-1-a-triangle) runs |
| 2. Text in a window (about 1,095) | 12 | A1, A6 | the frame block and one blend mode |
| | 13 | A2, A3, A4 | contracts on connections, CPU ends and counts per frame |
| | 14 | A5 | images |
| | 15 | B6, D11 | input, and `input` for headless tests |
| | 16 | E1, E5, E6, E2 | rects, font, palette and glyphs: text on screen |
| | 17 | E8, F4 | list; the terminal component in a window |
| 3. Panels (about 960) | 18 | E7, E9, F2 | split and hit: the dock |
| | 19 | F1, D10 | the panel, with tabs |
| | 20 | E10, G3 | keys, the theme and the keymap |
| | 21 | E11, D8, F3 | text: the editor |
| 4. Graph and Perform (about 1,090) | 22 | A7, E4 | offscreen targets and the image part |
| | 23 | E14, D9 | modes: `present` and `mode` |
| | 24 | E3, E13, F5 | curves and graph: the graph editor |
| | 25 | G2 | the `ide` app |

[Example 1](cli-examples.md#example-1-a-triangle) also needs D15 (`recipe new`), which moved to core at step 10. [Example 2](cli-examples.md#example-2-instanced-cubes-on-the-triangle) needs phase 1, A1 to A5 (steps 12 to 14) and A7 (step 22), plus three later rows: A9 (`mat4` values), A14 (depth) and D17 (`recipe publish`). It needs nothing else from phases 2 to 4, so it could run straight after phase 1 if those nine rows came next.

## Decisions this needs (A00)

- **D1, graph reads**: needed for B8 (step 9), and so for `inspect`, `graph` and `command-items`.
- **D5, operator ends of a connection**: needed for A2 and A3 (step 13), and so for every part that hands Rects, Items or Labels to another operator.
- **The `instance_count` word (V04)**: needed for A4 (step 13). `invocations` then counts one instance ([CLI examples](cli-examples.md#what-view-save-writes)).
- **The recipe template and `recipe new` (D15)**: moved to core by the lead on 2026-10-03, and built at step 10 with the `library` part ([CLI examples](cli-examples.md#a-recipe-file-shows-what-it-can-reach)).
- **The deploy word (V04)**: needed for C1 (step 7). The proposal is a `[deploy "<name>"]` section holding `recipe` and `param = <node>.<key>=<value>`, with ports reached as `<name>.<node>.<port>`. [Example 2](cli-examples.md#example-2-instanced-cubes-on-the-triangle) shows one. Built as proposed at step 7, for the lead to confirm before a library recipe uses it.
- **How the CLI and the IDE address the project they host (C3)**: a command names the child it edits (`triangle: node add …`), and the command-line part adds that name, so a typed line needs none and every log line stands alone (V08). Built so at step 10, with `:` alone for the host.
- **The recipe map**: it gains the `inspect` and `library` parts, and `graph-editor` ships without `relations` until E16 lands.
- **The choices example 2 makes** for images, draw outputs, depth and instancing: see [choices example 2 makes](cli-examples.md#choices-example-2-makes).
