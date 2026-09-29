# Migration and implementation

How code moves from the POC (`vulpen/` in the Generations repo) into this tree. Ported code is held to the [principles](principles.md) and [requirements](requirements.md) like new code.

## Porting

### Commands: the engine keeps the mechanism, recipes bring the vocabulary

Decided 2026-09-27. The POC's `vulpen/runtime/src/operators/cli/` (17,573 of the runtime's 42,027 lines, 42%) is not ported as engine code.

- **Floor, in the engine:** `CommandPort` registers a command with its usage, help and completion (RV04), dispatches text and keeps the log. It also owns the primitive edits, which are the manifest's own words: add and remove node, connect and disconnect, set param.
- **Load:** loading a `.vlp` runs its lines through the port as primitive commands. `Manifest` stays above `Commands` and shrinks to save and migrate (RV03).
- **Recipes:** every other command (drop, export, view new, list, undo UI, completion helpers) is a recipe: C++ on `Operator`, or a named block of primitive lines in the `.vlp`.
- **New manifest word:** the named command block. It passes V04, since no existing word defines a reusable command.
- **Log groups:** a log entry records the primitive commands it expanded to, as one group. Replay then needs no recipes (V08, C01), and undo removes one group.

The floor exists because:

- a headless replay with no recipes loaded still rebuilds the graph (V07, V08);
- a recipe command ends in a graph write, and if that write were not a command it would be a private way in (V06);
- a `.vlp` that defines commands cannot need them to load.

So the POC's `EngineControlCapability` is not ported: 47 `std::function` fields, bound in `RuntimeBindEngineControl.cpp`, that lend runtime-private operations to the verbs (1,238 lines with `EngineControl.h` and `Verb.h`). That is the private way in V06 forbids.

### What else weighs on the runtime

Measured 2026-09-27: the POC's `vulpen/src` and `vulpen/runtime/src` hold 88,970 lines of C++ (`.cpp` and `.h`), 84,027 without tests. Besides the commands, these groups are features, not ports (V05), or have no user (C00):

| Group | Lines | POC files | Port as |
| --- | --: | --- | --- |
| IDE dock, panels, tab drag | 3,648 | `RuntimeDock.cpp`, `Dock*` | recipe (V06: a GUI is a recipe on the command port) |
| Node canvas and graph mirror | 3,511 | `RuntimeGraph.cpp`, `RuntimeInputCanvas.cpp`, `RuntimeCanvasCamera.cpp`, `NodeCanvas.h`, `GraphSnapshot.h`, `VulpenNodeLayout.h` | canvas as a recipe; the mirror itself (about 1,250 lines of the 3,511) is deleted once operators can read the graph (D1) |
| IDE text widget, terminal mirror, find bar | 2,543 | `TextWidget*`, `TerminalLog.h`, `FindBar.h`, `Capabilities/Text.h` | recipe |
| Foreign-process streams | 2,280 | `vulpen_stream.h` (1,657, vendored by hosts), `VulpenStream*` | out of the engine (D4) |
| IDE popup, hover, modes, window node | 2,144 | `RuntimePopup.cpp`, `RuntimeHover.cpp`, `RuntimeModes.cpp`, `ModeState.h`, `WindowNode.*` | recipe; which node reaches the screen is a manifest fact |
| File watch, module rebuild, processes | 2,137 | `RuntimeWatch.cpp`, `VulpenProcess*` | the engine keeps a file-watch port and a minimal process spawn for glslang (RP03); theme and keymap watches go with their recipes; the module rebuild is build tooling |
| IDE chrome input, focus, keymap, theme | 1,907 | `RuntimeInputChrome.cpp`, `FocusRouter.*`, `KeyMap.*`, `KeyNames.*`, `GestureParams.cpp`, `PerformGestures.h`, `IdeTheme.*`, `IdePalette.h` | recipe on the engine's raw key and pointer port |
| Stats and profiling mirror | 1,759 | `RuntimeStatsMirror.cpp`, `VulpenFrameStats.*`, `VulpenGpuTimestamps.*`, `VulpenStatsChannel.h`, `VulpenProcMetrics*` | the engine keeps per-pass GPU timestamps (C03); the mirror and its display are a recipe |
| Feature builtin operators | 1,734 | `ImageLoader`, `MeshLoader`, `Ramp`, `KeyRoute`, `ReadbackProbe`, `StagingFiller`, `InputAdaptor`, `InputPublisher`, `TriggerLogger`, `Census`, `Noop`, `FrameIndexAdapter` | recipes; `Census` and `FrameIndexAdapter` have no user and are dropped; `Noop` is a test fixture |
| Bundle, fused player, ABI stamp | 1,684 | `VulpenPlayer.cpp`, `ViewFusedBuild.*`, `VulpenFused*`, `VulpenHostMarker.cpp`, `RuntimeAbiManifest.*` (its own SHA-256), `AuthoringRequirements.*`, `RuntimePaths.*`, `VulpenRecipePrebuilt.*` | export is a recipe (see commands); a player is a CMake target around an A04 `main()`; the ABI machinery depends on D2 |
| check, gen, lint | 1,439 | `VulpenCheck*`, `VulpenGen.cpp`, `VulpenAuditLint.cpp` | A02 makes the loader the validator, so `check` is a headless load that exits and the lint rules become load errors; `gen` is build tooling (RC00) |
| Glyph and icon atlases, stage strip | 1,297 | `VulpenGlyphAtlas.*`, `VulpenIconAtlas.*`, `VulpenStageStrip.*` | a font is an image a recipe loads; the icon atlas serves only the menubar recipe; the stage strip is an env-var debug overlay and is dropped |
| Source relations | 1,259 | `RuntimeGraphDeps.cpp`, `RuntimeGraphPaths.cpp`, `Capabilities/Relations.h` | recipe on the file port, as the POC's `ide-relations` already half is |
| Manifest extras | 1,003 | `VulpenViewInstances.*`, `VulpenViewProject.cpp` | `[include]` (~145 lines) has no user and is dropped; `instances = N` (used twice, both in `ide-full`) is a recipe command that expands to N `add node` in one log group, so it needs no manifest word (V04); the dock and mode projection goes to the IDE recipe; the naming lint moves into the loader |
| Screenshot, video, memory audit, trace | 997 | `vulkan_base/output/` | screenshot and video are output recipes on an asynchronous readback (V07, VK03), and only Generations records video; the memory audit is the A03 soak test, not runtime code |
| Config file | 218 | `VulpenConfig.*` | settings are node params, and only a save writes them (RA04, C02) |

With the commands, 48,371 of the 84,027 lines (58%) are not ported as engine code: most become recipes, some become build tooling, and about 4,700 are deleted outright (the control struct, the graph mirror, `check` and the lint, `[include]`, the two unused operators, the stage strip, the config file). A few rows leave a small port behind, as noted, and the command floor (6,499 of the `cli/` lines in the POC) is rewritten small as `CommandPort` instead of ported. [Code accounting](#code-accounting) places every file. The ~35,700 lines left are the core: device, resources, pipelines and frame loop, window, log and platform, lowering, shader compile and reflection, the manifest and operator core, and the runtime loop. Most of the POC's 13,958 lines of Python in `vulpen/tools/` guard IDE features (`panel_smoke.py`, `panel_cycle.py`, `export_smoke.py`, `cli_sequence_audit.py`), and each moves with the recipe it guards.

Child views (`RuntimeChild.cpp`, `ChildViewOperator`, 570 lines) stay: V03 makes a view hosting a view part of the core.

### Open decisions

- **D1 Graph reads.** Operators get a read-only `const View&` as a fourth port. Writes stay commands (V06) and reads change no state, so the log is unaffected (V08). It unlocks the node canvas, the relations, the dock and mode projection and `list` as recipes, and deletes the mirror. Approving it adds one edge to the [include map](include-map.md).
- **D2 Recipe C++, linked or loaded.** The POC `dlopen`s each view's `.so` against a separately staged `libVulpen.so`. The ABI stamp, the host marker and the fused-or-not player split exist only to survive that boundary, and the POC's ODR segfaults come from it. Linking recipe C++ into the binary that runs it removes all of that, and costs live C++ reload; keeping `dlopen` for a dev build alone keeps live reload with a smaller seam. Linked, two views that carry different copies of one recipe (V03, one edited by hand under V02) define the same class twice in one binary, which the ODR forbids, so each deployed copy would need a namespace of its own.
- **D3 Engine as nodes.** The POC declares `Device`, `Swapchain`, `FrameLoop`, `FrameDraw`, `Surface`, `FramebufferTargets`, `HostLoader` and `Fullscreen` as builtin operators in its bootstrap `.vlp` (1,044 lines), which V01 favours. This tree's skeleton makes them `Engine` members (`baseclasses/Engine.h`, `Swapchain.h`). One of the two is picked before these are ported.
- **D4 Streams.** `vulpen_stream.h` is one C header that foreign processes copy whole, and it calls `shm_open`. RA01 keeps OS APIs in the platform files, so it lands either as a platform shared-memory port with the protocol in the stream recipe, or as a library beside the recipes that RA01 exempts by name.
- **D5 Operator ends of a connection.** Parts hand Rects, Items and Labels from operator to operator as often as to a shader ([recipe map](recipe-map.md)). A contract is laid out once, in GLSL (RV05); open is how an operator reads and writes one. The engine knows both ends of every connection, so it can place the transfers as it places barriers (V10): data stays on the GPU between shaders, stays on the CPU between operators, and crosses once, asynchronously, where one feeds the other (VK03). A part can then move between C++ and GLSL without its neighbors noticing (Goal-01). The other way, a typed publish port per kind like the POC's `RelationPublishCapability`, grows the engine by one port per feature (V05).

## Code accounting

Every tracked code file under the POC's `vulpen/`, grouped by function and by where it lands in this tree. Measured 2026-09-27 at `4f299714`: 746 files, 133,697 lines, 85,166 lines of code. The vendored `vulpen/external_libraries/` (stb, VMA, tinyobj: 5 files, 37,948 lines) is left out.

- **Lines** counts every line, as `wc -l` does and as the section above does. **Code** counts lines that hold any code, by the rule of [`fetch-statistics.py`](../../src/tools/fetch-statistics.py), so it compares with [LOC.md](../statistics/LOC.md).
- **Classes** counts the distinct named C++ `class` and `struct` definitions with a body in each group, nested types and per-OS `Impl`s included. They were found by a pattern over comment-free source, not by a compiler. GLSL structs and Python classes are not counted.
- **From** paths start at `vulpen/src/vulkan_base/` in the baseclasses table, at `vulpen/src/operator_base/` or `vulpen/runtime/src/` in the runtime table (`cli/` is `runtime/src/operators/cli/`), and at `vulpen/` beside the engine.
- **Rework home** is where the code lands in this tree, or what happens to it (see [Porting](#porting)).
- **Rating** is a proposal; the project lead settles it (A00). It rates the code against the narrative of a graph of C++ and GLSL nodes (V00, Goal-01):
  - **Essential**: without it, such a graph cannot be loaded, changed by command, scheduled and run on the GPU, headless or windowed.
  - **Useful**: serves or proves the narrative (measurement, tests, data in and out, examples), but the graph runs without it.
  - **Optional**: a feature beside the narrative, mostly the IDE, or code that nothing uses.

### Baseclasses

| Group | Rework home | From (POC) | Functionality | Rating | Files | Lines | Code | Classes |
| --- | --- | --- | --- | --- | --: | --: | --: | --: |
| Log | `baseclasses/Log.h` | `output/VulpenLog`, `VulpenTrace` | log lines, levels, sinks, ANSI, `out/log.txt` | Essential | 4 | 951 | 677 | 1 |
| Platform | `baseclasses/Platform.h` | `platform/` (process, console, dynlib, time, clipboard), `presentation/VulpenWindow` | window (GLFW), terminal, processes, dynamic libraries, time, clipboard | Essential | 15 | 2,938 | 2,000 | 15 |
| Mechanics | `baseclasses/Mechanics.h` | `device/`, `mechanics/VulpenMechanics`, `resources/VulpenSync`, `VulpenCommands` | instance, device, queues, validation, feature tiers, command buffers, sync | Essential | 14 | 2,365 | 1,719 | 18 |
| Resources | `baseclasses/Resources.h` | `resources/` (buffers, images, bindless registry, descriptors, VMA, arena), `presentation/VulpenGeometry` | buffers, images, bindless registry, descriptors, offscreen targets, meshes | Essential | 17 | 3,420 | 2,620 | 21 |
| Pipelines | `baseclasses/Pipelines.h` | `pipelines/`, `resources/VulpenBindless*` | pipelines, presets, bindless graphics and compute passes, the pass chain | Essential | 15 | 3,096 | 2,103 | 20 |
| Swapchain | `baseclasses/Swapchain.h` | `resources/VulpenSwapchain` | swapchain create, acquire, present, resize | Essential | 2 | 519 | 386 | 2 |
| Engine | `baseclasses/Engine.h` | `VulpenEngine*`, `VulpenLibrary`, `precompiled.h`, `mechanics/VulpenFrameLoop`, `VulpenTimer` | owns the GPU, frame loop, engine resources, library entry, PCH | Essential | 13 | 3,038 | 1,946 | 15 |
| Profiling | timestamps in `baseclasses/Engine.h`, display a recipe | `mechanics/VulpenFrameStats`, `VulpenGpuTimestamps`, `VulpenStatsChannel`, `resources/VulpenPassTimer`, `platform/VulpenProcMetrics*` | per-pass GPU timestamps, frame stats, process metrics | Useful | 9 | 1,260 | 777 | 16 |
| Memory and allocation audits | soak test, debug build | `output/VulpenMemoryVerification`, `CookAudit`, `resources/VulpenObjectCensus` | Vulkan object census, memory verification (A03), cook-phase heap audit (CPP10) | Useful | 6 | 805 | 557 | 8 |
| Screenshot | output recipe | `output/VulpenScreenshot` | readback of a frame to PNG; the goldens use it | Useful | 2 | 202 | 160 | 1 |
| Video recorder | output recipe | `output/VulpenVideoRecorder` | frames to a video file; only Generations uses it | Optional | 2 | 213 | 156 | 1 |
| Glyph atlas | font recipe | `presentation/VulpenGlyphAtlas` | ASCII SDF font atlas that every text shader samples | Useful | 2 | 438 | 313 | 3 |
| Icon atlas, stage strip | menubar recipe; strip dropped | `presentation/VulpenIconAtlas`, `VulpenStageStrip` | menubar symbol atlas; env-var debug strip of pass outputs | Optional | 4 | 859 | 633 | 5 |
| Foreign-process streams | D4 | `platform/vulpen_stream.h`, `VulpenStream*` | shared-memory ring a foreign process publishes into | Useful | 5 | 2,280 | 1,520 | 9 |
| **Total** |  |  |  |  | 110 | 22,384 | 15,567 | 135 |

### Runtime

| Group | Rework home | From (POC) | Functionality | Rating | Files | Lines | Code | Classes |
| --- | --- | --- | --- | --- | --: | --: | --: | --: |
| View | `runtime/View.h` | `Manifest/VulpenViewModel.h`, `VulpenViewAccess.h`, `VulpenPayloadLayout.h`, `Chain/VulpenChannelKind.h`, `VulpenChannelEnums` | the graph model: `View`, `Node`, payloads, channel kinds | Essential | 7 | 2,006 | 939 | 17 |
| Manifest | `runtime/Manifest.h` | `Manifest/VulpenViewManifest`, `Serialize`, `Validate`, `Derive` | read, write, validate and derive a `.vlp` | Essential | 7 | 2,025 | 1,384 | 1 |
| Manifest extras | dropped, recipe command, loader | `Manifest/VulpenViewInstances`, `VulpenViewProject.cpp` | `instances = N`, `[include]`, dock and mode projection, naming lint, channel-kind inference | Optional | 3 | 1,003 | 666 | 4 |
| Command floor | `runtime/Commands.h` | `cli/command_spec`, `command_line`, `commands`, `command_completion`, `undo`; `cmds/node`, `edges`, `history`, `view`, `shared_ops`, `shared_topology` | registry with usage, help and completion; dispatch; undo; primitive edits (node, param, connect); load and save | Essential | 18 | 6,499 | 4,429 | 13 |
| Recipes | `runtime/Recipes.h`; `cpp_import` dropped (RV00) | `cli/recipe_library`, `cmds/recipe`, `recipe_drop`, `shared_imports`; `Manifest/VulpenRecipePrebuilt` | library lookup, drop, capture and publish, prebuilt modules, `cpp_import` | Essential | 9 | 2,593 | 1,610 | 5 |
| Child views and nesting | not placed yet (V03) | `RuntimeChild.cpp`, `builtins/ChildViewOperator`, `cmds/nest`, `shared_nest` | a view hosting a view (V03), nest folders, `cd`, promote, merge | Essential | 4 | 1,736 | 1,270 | 4 |
| Operator and ports | `runtime/Operator.h` | `operator.h`, `Operator/` (base, registry, contexts, capabilities), `RuntimeBindPorts`, `RuntimeAuthoredPublish`, `RuntimeHit` | `Operator` base, registry, contexts, param schema, the ports a recipe reads (pointer, pick, control, file, graph edit) | Essential | 18 | 3,111 | 1,544 | 49 |
| Module loading | D2 | `Module/` | `dlopen` of a view's `.so` (D2) | Useful | 3 | 381 | 176 | 2 |
| Host builtin operators | D3 | `builtins/` Device, Swapchain, FrameLoop, FrameDraw, Surface, FramebufferTargets, HostLoader, Fullscreen, InputAdaptor, InputPublisher | device, swapchain, frame loop and draw, surface, targets, input, host loader, fullscreen (D3) | Essential | 10 | 1,477 | 906 | 10 |
| Data operators | recipes | `builtins/` ImageLoader, MeshLoader, StagingFiller, ReadbackProbe | image and mesh loading, CPU-to-GPU fill, readback | Useful | 4 | 934 | 613 | 8 |
| Small operators | recipes; unused dropped | `builtins/` Ramp, KeyRoute, TriggerLogger, Census, Noop, FrameIndexAdapter | ramp, key route, trigger logger; `Census` and `FrameIndexAdapter` unused, `Noop` a test fixture | Optional | 6 | 367 | 222 | 7 |
| Schedule | `runtime/Schedule.h` | `Chain/` (flatten, lowering, shader compile, reflection, bindless wiring, reload, emitter), `VulpenOperatorSchedule`, `assets/presets/` | flatten, lower to a chain spec, shader compile, SPIR-V reflection, bindless wiring, reload with operators, C++ mirror emitter | Essential | 44 | 7,140 | 4,252 | 38 |
| Runtime core | `runtime/Runtime.h` | `VulpenRuntime`, `RuntimeFrame`, `RuntimeInternal.h`, `RuntimeInput`, `RuntimePaths`, `WindowNode`, `VulpenBootstrap`, `VulpenRunUntilIdle` | construction, frame loop, input driver, host paths, window node, bootstrap `main()` | Essential | 13 | 4,476 | 2,174 | 16 |
| Host GPU layout and graph | `baseclasses/GpuLayout.glsl`, host `.vlp` | `runtime/shaders/shared/`, `Grid.*`, `runtime/view.vlp`, `bootstrap.vlp` | shared GLSL (bindless, push constants, input), host grid shaders, `view.vlp`, `bootstrap.vlp` | Essential | 10 | 881 | 443 | — |
| Builtin shaders | recipes | `runtime/shaders/builtins/` | shaders the basic recipes and builtins run (triangle, cube, vector add, histogram, composite) | Useful | 16 | 577 | 293 | — |
| Hot reload | file-watch port; rebuild is tooling | `RuntimeWatch.cpp` | shader, theme and keymap watches; view-module rebuild | Useful | 1 | 913 | 586 | — |
| Command vocabulary and CLI front end | recipes (`recipes/apps/cli/`, `recipes/parts/command-line/`) | the rest of `cli/` and `cli/cmds/` | every other verb: read, list, select, layout, operator scaffold, shell verbs, lifecycle hooks, jobs, probe, the terminal operator | Optional | 15 | 5,770 | 4,066 | 9 |
| Engine-control backdoor | dropped (V06) | `RuntimeBindEngineControl`, `Capabilities/EngineControl.h`, `Verb.h` | 47 `std::function` fields lending runtime-private operations to verbs | Optional | 3 | 1,238 | 797 | 2 |
| Bundle, export, player, ABI | recipe and CMake (D2) | `VulpenPlayer`, `ViewFusedBuild`, `VulpenFused*`, `VulpenHostMarker`, `RuntimeAbiManifest`, `AuthoringRequirements`, `cmds/export*`, `bundle_stamp` | `view export`, bundle stamp, fused player, host marker, ABI stamp with its own SHA-256 | Optional | 20 | 3,170 | 1,832 | 9 |
| check, gen, lint, dump | loader errors (A02) and tooling | `VulpenCheck*`, `VulpenGen`, `Chain/VulpenAuditLint`, `VulpenChainDump` | `vulpen check`, `vulpen gen`, audit lint, chain dump | Optional | 6 | 1,543 | 1,028 | 1 |
| IDE dock and panels | recipes (`recipes/apps/ide/`; [recipe map](recipe-map.md)) | `RuntimeDock`, `Dock*` | dock tree, splits, tabs, tab drag, perform panels | Optional | 5 | 3,648 | 2,228 | 7 |
| IDE text widget | recipe | `TextWidget*`, `FindBar.h`, `TerminalLog.h`, `Capabilities/Text.h` | editable text surface, tabs, find bar, terminal log mirror | Optional | 8 | 2,543 | 1,551 | 11 |
| IDE chrome input, keys, theme | recipe | `RuntimeInputChrome`, `FocusRouter`, `KeyMap`, `KeyNames`, `GestureParams`, `PerformGestures.h`, `IdeTheme`, `IdePalette`, `*.schema` | chrome input, focus router, keymap, key names, gestures, theme and palette | Optional | 15 | 2,146 | 1,105 | 7 |
| IDE popup, hover, modes | recipe | `RuntimePopup`, `RuntimeHover`, `RuntimeModes`, `ModeState.h` | context popup, hover and tooltip, modes and auto composite | Optional | 4 | 2,064 | 1,245 | 1 |
| IDE node canvas and graph mirror | recipe; mirror dropped (D1) | `RuntimeGraph`, `RuntimeInputCanvas`, `RuntimeCanvasCamera`, `NodeCanvas.h`, `Projection.h`, `GraphSnapshot.h`, `VulpenNodeLayout.h` | canvas camera and input, node layout, the flattened graph snapshot (D1) | Optional | 7 | 3,511 | 2,003 | 7 |
| IDE source relations | recipe | `RuntimeGraphDeps`, `RuntimeGraphPaths`, `Capabilities/Relations.h` | includes, `uses_cap`, path relations drawn on the canvas | Optional | 3 | 1,259 | 820 | 11 |
| IDE stats mirror | recipe | `RuntimeStatsMirror`, `Capabilities/FrameStats.h` | per-frame cost mirror for display | Optional | 2 | 559 | 289 | 2 |
| Config file | dropped (RA04) | `VulpenConfig` | ini with device preference and window state | Optional | 2 | 218 | 151 | 1 |
| **Total** |  |  |  |  | 263 | 63,788 | 38,622 | 242 |

### Beside the engine

| Group | From (POC) | Functionality | Rating | Files | Lines | Code | Classes |
| --- | --- | --- | --- | --: | --: | --: | --: |
| Tests | `src/tests/` | C++ unit tests, leak test, shell scenarios | Useful | 47 | 7,386 | 5,173 | 6 |
| Build | `CMakeLists.txt`, `cmake/`, `run.sh` | CMake, view-module and bundle CMake, stamps, `run.sh` | Essential | 6 | 1,284 | 621 | — |
| Recipes: rendering basics | 22 recipes | triangle, cubes, instancing, indexed and indirect draw, mesh shader, textures, blend, composite, view render, camera, light | Useful | 79 | 3,411 | 2,192 | — |
| Recipes: compute | 5 recipes | vector add, particles, histogram, image difference, SVG export | Useful | 20 | 887 | 588 | 1 |
| Recipes: terrain and post effects | `fx_*`, `generator_terrain-*`, `geometry_terrain-*` | terrain generator, grid, markers, scene; bloom, edge, fog, tonemap, vignette (Goal-04) | Useful | 45 | 1,650 | 1,258 | — |
| Recipes: input, text, layout | 11 recipes | button, mouse, control value, hit rect, pick, layout, text, font chart | Useful | 41 | 2,452 | 1,645 | 4 |
| Recipes: external data | `extern-stream`, `extern-video`, `hopfield`, `recipe-src/hopfield`, `recipe-src/compose-showcase` | stream and video from a foreign process, Hopfield network (Goal-03) | Useful | 15 | 3,204 | 1,955 | 2 |
| Recipes: IDE | `ide-*`, `dock-participant`, `widget-participant`, `recipe-src/ide-full` | node grid, perform, text panel, terminal, chrome, menubar, context menu, tooltip, relations, theme, dock and widget participants | Optional | 67 | 11,224 | 6,921 | 12 |
| Recipes: apps | `turtle-os`, `slideshow` | Turtle-OS, slideshow | Optional | 9 | 615 | 396 | 1 |
| Examples | `examples/` | foreign producers publishing into Vulpen | Useful | 6 | 1,454 | 921 | — |
| Tools: feature gates | 15 scripts in `tools/` | gates and generators that guard the IDE, CLI and export | Optional | 15 | 5,477 | 3,655 | — |
| Tools: general gates | 23 scripts in `tools/` | check entry point, goldens, ratchet, structure and folder gates, soak, recipe smoke, harness | Useful | 23 | 8,481 | 5,652 | — |
| **Total** |  |  |  | 373 | 47,525 | 30,977 | 26 |

### By rating

| Rating | Engine code | Engine classes | All `vulpen/` code | All `vulpen/` classes |
| --- | --: | --: | --: | --: |
| Essential | 30,402 (56%) | 245 | 31,023 (36%) | 245 |
| Useful | 4,995 (9%) | 47 | 24,379 (29%) | 60 |
| Optional | 18,792 (35%) | 85 | 29,764 (35%) | 98 |
| **Total** | 54,189 | 377 | 85,166 | 403 |

The engine is the baseclasses and runtime tables together. Three Essential rows still carry weight this tree does not keep:

- **Command floor** is the POC's floor, at 4,429 lines of code.
- **Runtime core** holds IDE state in `VulpenRuntime.h` and `RuntimeInternal.h` (`GraphSnapshot`, `HoverState`, `PopupState`, `PerformTab`, `StatsMirrorStore`, `SurfacePanels`).
- **Operator and ports** has 26 `*Capability` structs where V05 asks for a few general ports, and many of them serve only the IDE (`DockCapability`, `EditorCapability`, `FocusCapability`, `CameraRestoreCapability`, `GlyphMetricsCapability`).

Recipes also copy code into each other, since RV00 forbids sharing it: `ChromeOperator` is defined in 4 IDE recipes and `StreamSourceOperator` in 3. In this tree a recipe deploys the part that holds such code instead of copying it (V11), and recipes share only contracts (RV05); the [recipe map](recipe-map.md) shows where each IDE job lands.

### Classes per group

- **Log** (1): `Style`
- **Platform** (15): `Button`, `ChildProcess`, `ChildProcess::Impl`, `DisplayConfiguration`, `DynLib`, `FileLock`, `FileLock::Impl`, `KeyEvent`, `Mouse`, `ProcessResult`, `RawTerminalMode`, `RawTerminalMode::Impl`, `Window`, `WriteStream`, `WriteStream::Impl`
- **Mechanics** (18): `BaseCommandBuffers`, `BaseCommandInterface`, `BaseDevice`, `BaseInitializeVulkan`, `BaseMechanics`, `BaseQueues`, `BaseSingleUseCommands`, `BaseSynchronizationObjects`, `BaseValidationLayers`, `BorrowedDevice`, `BorrowedFramebuffers`, `BorrowedRenderTargets`, `BorrowedSurface`, `BorrowedSwapchain`, `FamilyIndices`, `FeatureTiers`, `GpuLogSettings`, `Instance`
- **Resources** (21): `BaseBuffer`, `BaseDescriptor`, `BaseDescriptorInterface`, `BaseGridImage`, `BaseImage`, `BaseSampledGridImage`, `BaseSwapchainStorageImage`, `BaseTextureSampler`, `BindlessHandle`, `BindlessRegistry`, `Geometry`, `KindState`, `Mesh`, `OffscreenTarget`, `Recipe`, `ResourceArena`, `Shape`, `SsboReader`, `std::hash<…>`, `TransitionEntry`, `Vertex`
- **Pipelines** (20): `BaseRenderPass`, `BindlessChain`, `BindlessComputePass`, `BindlessDispatchExtent`, `BindlessFieldOffset`, `BindlessGraphicsPass`, `BindlessGraphicsState`, `BindlessOptionalStages`, `BindlessPass`, `BindlessVertexInput`, `BlendFactors`, `ChainPreset`, `Entry`, `GraphicsDraw`, `GraphicsEntry`, `PassReport`, `ResourceBinding`, `ShaderPass`, `TileGroup`, `VertexLayout`
- **Swapchain** (2): `BaseSwapchain`, `SupportDetails`
- **Engine** (15): `ChainReload`, `DepthDesc`, `EngineAccess`, `EngineCommands`, `EngineConfig`, `FeedbackPair`, `FrameHooks`, `FrameLoop`, `FrameProfiler`, `FrameSample`, `FrameState`, `HostDesc`, `Timer`, `Vec2UintFast16`, `VulkanCore`
- **Profiling** (16): `FrameState`, `FrameStats`, `GpuStatRow`, `GpuStatsHeader`, `GpuTimestamps`, `PassInfo`, `PassTimer`, `Percentiles`, `PipelineStats`, `Recorded`, `RegionResult`, `ResourceCensus`, `Scope`, `ScopeAccum`, `SpikeContext`, `StatRow`
- **Memory and allocation audits** (8): `ExitReporter`, `GpuAllocation`, `MemoryVerification`, `ObjectCensus`, `Pause`, `Scope`, `Stats`, `VulkanObject`
- **Screenshot** (1): `Screenshot`
- **Video recorder** (1): `VideoRecorder`
- **Glyph atlas** (3): `GlyphAtlas`, `Metric`, `PositionedGlyph`
- **Icon atlas, stage strip** (5): `IconAtlas`, `P2`, `StageStripCache`, `StageStripConfig`, `StageStripTile`
- **Foreign-process streams** (9): `StreamClient`, `StreamClient::Impl`, `StreamField`, `StreamPort`, `StreamPorts`, `StreamProbe`, `StreamSample`, `StreamStats`, `vp_stream`
- **View** (17): `Appetite`, `DockSpec`, `DockSpecNode`, `Entry`, `LayoutSpec`, `ModeSpec`, `ModeSpecEntry`, `Node`, `NodeEnum`, `NodeOutput`, `PayloadSize`, `PayloadSpec`, `RecipePort`, `RefusedShrink`, `Row`, `View`, `ViewportPx`
- **Manifest** (1): `StageBag`
- **Manifest extras** (4): `Candidate`, `InstanceSet`, `InstanceWritePlan`, `Rewrite`
- **Command floor** (13): `CommandLine`, `CommandNode`, `CommandSpec`, `DetachedNode`, `Entry`, `GeneratedFragment`, `Queued`, `RuntimeHooks`, `Snap`, `TabResult`, `TrashEntry`, `UndoStack`, `ViewState`
- **Recipes** (5): `DropState`, `DropStepClass`, `PrebuiltRecord`, `RecipeRef`, `RecipeWriteResult`
- **Child views and nesting** (4): `ChildViewOperator`, `DemoteRecord`, `PromoteRecord`, `Suffix`
- **Operator and ports** (49): `CameraRestoreCapability`, `CapabilityRegistry`, `caps_to_vector<…>`, `CliCapability`, `ContextAccess`, `ControlCapability`, `CookContext`, `DeviceCapability`, `DockCapability`, `EditorCapability`, `EngineCapability`, `Entry`, `EventContext`, `FileFacts`, `FileReadCapability`, `FocusCapability`, `FramebufferCapability`, `GlyphMetricsCapability`, `GraphEditCapability`, `GridPushCapability`, `HostChainCapability`, `HostFrameLoopControllerCapability`, `InitContext`, `InputGateCapability`, `InputRoute`, `list_contains<…>`, `NodeFacts`, `Operator`, `OperatorContextBacking`, `OperatorContextBase`, `OperatorRegistry`, `ParamSchema`, `ParamSpec`, `PickCapability`, `PickResult`, `PointerCapability`, `RuntimeEventsCapability`, `RuntimeStateCapability`, `ScopedLookup`, `ScopedOwner`, `SurfaceCapability`, `SurfaceRecord`, `SwapchainCapability`, `TriggerPayload`, `UserViewChainCapability`, `ViewModelCapability`, `ViewModuleCapability`, `WidgetSlots`, `WindowCapability`
- **Module loading** (2): `ViewModule`, `ViewModuleLoader`
- **Host builtin operators** (10): `DeviceOperator`, `FramebufferTargetsOperator`, `FrameDrawOperator`, `FrameLoopOperator`, `FullscreenOperator`, `HostLoaderOperator`, `InputAdaptorOperator`, `InputPublisherOperator`, `SurfaceOperator`, `SwapchainOperator`
- **Data operators** (8): `DrawIndexedIndirect`, `ImageLoaderOperator`, `IndexKey`, `IndexKeyHash`, `MeshLoaderOperator`, `MeshVertex`, `ReadbackProbeOperator`, `StagingFillerOperator`
- **Small operators** (7): `CensusOperator`, `FrameIndexAdapter`, `KeyRouteOperator`, `NoopOperator`, `RampOperator`, `RampState`, `TriggerLoggerOperator`
- **Schedule** (38): `BindlessRenderPassSpec`, `BlockEmit`, `ChainCompatibility`, `ChainContext`, `ChainSpec`, `ChannelPreset`, `ChannelSpec`, `EmitResult`, `FeatureTiers`, `Field`, `FlattenedView`, `HostBroadcast`, `LintFinding`, `LoadedOperators`, `MeshPreset`, `NodeShaderTokens`, `OperatorSchedule`, `OperatorScheduleEntry`, `ParsedSpirv`, `PassSpec`, `Pending`, `PortRef`, `ProcResult`, `ReloadAccess`, `ReloadOptions`, `ShaderBlock`, `ShaderBlockField`, `ShaderCompileError`, `ShaderCompileResult`, `ShaderStage`, `StageRow`, `StagingChannel`, `TransparentStringEq`, `TransparentStringHash`, `TypeNode`, `TypeSpec`, `ViewRenderRequest`, `ViewRenderTarget`
- **Runtime core** (16): `ChildView`, `GraphSnapshot`, `HoverState`, `HoverTarget`, `InputFrame`, `ModuleBuild`, `PathRel`, `PerformTab`, `PerformView`, `PointerClaims`, `PointerPort`, `PopupRow`, `PopupState`, `RuntimeState`, `StatsMirrorStore`, `SurfacePanels`
- **Command vocabulary and CLI front end** (9): `CliOperator`, `Group`, `K`, `OperatorHome`, `P`, `Pred`, `ProjectedCalls`, `Row`, `SpanSummary`
- **Engine-control backdoor** (2): `EngineControlCapability`, `VerbCapability`
- **Bundle, export, player, ABI** (9): `AbiManifest`, `AbiManifestEntry`, `BundleFile`, `BundleStamp`, `DigestCache`, `LiveChain`, `SdkFile`, `SdkPart`, `Sha256`
- **check, gen, lint, dump** (1): `BroadcastArray`
- **IDE dock and panels** (7): `DockHighlight`, `DockLayout`, `DockNode`, `Drop`, `LeafRect`, `Rect`, `Seam`
- **IDE text widget** (11): `Appetite`, `DetachedTab`, `Entry`, `FindBar`, `PointerInput`, `Snapshot`, `Tab`, `TerminalLog`, `TextBufferCapability`, `TextSurfaceRegistry`, `TextWidget`
- **IDE chrome input, keys, theme** (7): `Entry`, `FocusRouter`, `IdeMetrics`, `KeyBinding`, `KeyMap`, `Named`, `PerformGestures`
- **IDE popup, hover, modes** (1): `ModeState`
- **IDE node canvas and graph mirror** (7): `GraphSnapshotCapability`, `NodeCanvas`, `PortalArm`, `Projection`, `SnapshotEdge`, `SnapshotNode`, `SnapshotPort`
- **IDE source relations** (11): `CacheEntry`, `DepGraph`, `NodeScan`, `Place`, `Published`, `RawRel`, `Relation`, `RelationPublishCapability`, `RelDetail`, `Row`, `SourceFacts`
- **IDE stats mirror** (2): `FrameStatsCapability`, `StatMirrorRow`
- **Config file** (1): `RuntimeConfig`
- **Tests** (6): `AllocationRecord`, `DeviceMemoryRecord`, `HostBdaBuffer`, `LeakReport`, `Rec`, `WindowGuard`
- **Recipes: compute** (1): `SvgExportOperator`
- **Recipes: input, text, layout** (4): `ButtonOperator`, `FontEyeChartOperator`, `TextOperator`, `TextStaticOperator`
- **Recipes: external data** (2): `Field`, `StreamSourceOperator`
- **Recipes: IDE** (12): `ChromeOperator`, `DockProbeOperator`, `Found`, `NodeGridOperator`, `PortalGeom`, `PreviewOperator`, `RecordSink`, `RelationScanOperator`, `StageOperator`, `TextOperator`, `Tree`, `WidgetProbeOperator`
- **Recipes: apps** (1): `SlideshowOperator`
