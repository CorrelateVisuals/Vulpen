# Using Vulpen

How each part of Vulpen works today, shown the way you use it. Each section has a session, a manifest or a recipe's code, with comments that say what happens, where it holds and what comes out. Every session and output here ran at `79c92d1` on 2026-10-03; the log's time column is left out, and long paths are shortened with `…`. Code marked as a sketch shows a call's shape, not a file in the tree. What is planned but not built yet is in the [IDE port](plans/ide-port.md) and the [CLI examples](plans/cli-examples.md).

1. [Words](#1-words)
2. [Running a view](#2-running-a-view)
3. [The manifest](#3-the-manifest)
4. [Recipes: C++ and GLSL](#4-recipes-c-and-glsl)
5. [The GPU layout](#5-the-gpu-layout)
6. [Live code](#6-live-code)
7. [Commands](#7-commands)
8. [Graph edits](#8-graph-edits)
9. [Scripts and the log](#9-scripts-and-the-log)
10. [Saving a view](#10-saving-a-view)
11. [Deploys](#11-deploys)
12. [The CLI](#12-the-cli)
13. [Hosted views](#13-hosted-views)
14. [The library](#14-the-library)
15. [What a recipe's C++ can reach](#15-what-a-recipes-c-can-reach)
16. [When something is wrong](#16-when-something-is-wrong)

## 1. Words

| Word | What it is | Where |
| --- | --- | --- |
| view | a project: a folder with a `view.vlp` and its own `recipes/`. It loads, runs and moves as a whole (V03) | `src/examples/<name>/` |
| manifest | a view's `view.vlp`: its nodes, connections and deploys (V00) | `<view>/view.vlp` |
| recipe | a folder of C++ and GLSL that nodes run | `<view>/recipes/<recipe>/` |
| node | one `[node]` of a manifest: a recipe's C++ class, its shaders, or both | the manifest |
| connection | one buffer: one node's port writes it, other nodes' ports read it | the manifest |
| deploy | a recipe brought into a view under a name; its nodes become `<deploy>.<node>` | the manifest |
| library | the recipes every view copies from: parts hold code; components and apps only deploy (RV06) | `src/recipes/{parts,components,apps}/` |
| host, hosted view | the view vulpen started with hosts other views, which commands reach by name | `child add` |
| command | a line of text that does one thing; every action is one (V06) | the command port |
| primitive | a command that edits or saves a view; the log keeps exactly these (V08) | the log |

## 2. Running a view

`run.sh` (or `run.ps1` on Windows) builds the debug preset in parallel and runs `out/build/debug/vulpen` with your arguments; `--release` takes the release preset.

```bash
./run.sh src/examples/wave/view.vlp   # the view to run: one view.vlp
         --log info                   # how much it says: error, warn (the default), info, debug
         --frames 61                  # stop after 61 frames; 0, the default, runs until quit
         --fps 0                      # unpaced; the default is 60 frames a second
#        --first-frame N              # start counting frames at N, so a test reaches years in a moment
#        --source script.txt          # run a file of commands before the first frame
```

What it prints:

```text
{run} vulpen 79c92d1                                                # the commit it was built from (RC07)
{run} view wave from …/src/examples/wave/view.vlp: no node draws, so it runs headless   # V07
{gpu} Vulkan validation is on
{gpu} runs on NVIDIA GeForce RTX 5070 Laptop GPU: discrete, Vulkan 1.4.312
{nod} wave: pipeline from Wave.comp                                # each node's pipeline and operator
{nod} wave: new operator Wave
{nod} probe: pipeline from Probe.comp
{nod} probe: new operator Probe
{mem} wave: values of 4096 bytes on the GPU                        # the connection's buffer
{mem} probe: probe.samples of 32 bytes read back by the CPU
{mod} live code: a save under …/src/examples/wave swaps in         # debug builds only (section 6)
{out} probe: frame     60: +0.8282516 +0.88405794 +0.9278144 …     # what a node's C++ logs
{run} ran 61 frames in 0.01 s
```

- **Headless or a window.** A view runs headless unless a node draws; `./run.sh src/examples/triangle/view.vlp` says `a node draws, so it opens a window`. The window also opens and closes while running, as draws come and go (section 8).
- **Quiet by default.** At `warn` a run that goes as meant prints only the frame around the log; `{!!!}` marks an error and `{ ! }` a warning (C09).
- **Exit code.** 0, or 1 when a node is in error when the run ends, or the view cannot load.
- **Where:** `src/runtime/Runtime.cpp` parses the arguments and runs the loop; `src/baseclasses/Log.h` lists what each level and tag shows.

## 3. The manifest

A view's `view.vlp` is the graph, in a few words with closed sets of values (V04). The wave example, with what each line does:

```ini
[manifest]
version = 1                 # every manifest states its version; this build reads 1 (RV03)

[node "wave"]               # a node, by a name that is unique in the view
recipe      = wave          # its recipe: the folder recipes/wave/ of this view (V03)
operator    = Wave          # the C++ class that recipe registers; leave it out for a shader alone
shader      = Wave.comp     # one .comp makes a dispatch; a .vert and a .frag make a draw
invocations = 1024          # a dispatch's threads, or a draw's vertices per instance
param       = amplitude=1.0 # a value the shader's pass block or the C++ reads, by name
param       = speed=0.02

[node "probe"]
recipe      = probe
operator    = Probe
shader      = Probe.comp
invocations = 8
param       = step=128
param       = every=60
log         = info          # this node's own log level (V09)

[connection "values"]       # one buffer
from = wave.values          # the node.port that writes it
to   = probe.values         # the node.port that reads it; repeat `to` for more readers
```

- **The words.** `[node]` takes `recipe`, `operator`, `shader`, `invocations`, `instance_count`, `param` and `log`; `[connection]` takes `from` and `to`; `[deploy]` takes `recipe` and `param` (section 11). `instance_count = 64` makes a draw draw 64 instances; left out, it draws one.
- **Loading runs edits.** Each section goes through the same edits a command makes (section 8), so a loaded view and a typed one pass the same checks.
- **Order.** Nodes run writers before readers, as the connections order them, then in the manifest's order.
- **A mistake names its line** and stops the load (A02):

  ```text
  {!!!} wave/view.vlp:19: unknown word invocatons in a node; its words are recipe, operator, shader, invocations, instance_count, param and log
  ```

- **Comments stay.** `view save` writes the manifest back with the comments a person wrote (section 10).
- **Where:** `src/runtime/Manifest.cpp` reads and writes it; `src/runtime/Edits.cpp` checks each word.

## 4. Recipes: C++ and GLSL

A recipe is a folder; a node names it and picks a class and shaders from it. The triangle, whose C++ moves the corners and whose shaders draw them:

```text
src/examples/triangle/
  view.vlp                # [node "triangle"] recipe = triangle, operator = Triangle, shaders below
  recipes/triangle/
    Triangle.cpp          # the operator: C++ that runs every frame
    Triangle.glsl         # the pass block both shaders include
    Triangle.vert         # the vertex shader
    Triangle.frag         # the fragment shader
```

The pass block is what C++ and the shaders share, by name:

```glsl
// Triangle.glsl
layout(buffer_reference, std430) readonly buffer Corners { // a buffer, by its address
  vec2 at[];
};

layout(set = 1, binding = 0) uniform Pass { // every shader's pass block lives here (RV02)
  vec4 tint;       // set by C++ every frame: node.value<glm::vec4>("tint")
  Corners corners; // written by C++: node.upload<glm::vec2>("corners")
} pass;
```

The operator asks for those names once, in `bind`, and uses the handles every frame:

```cpp
// Triangle.cpp, shortened
class Triangle final : public VP::Operator {   // in an unnamed namespace, so copies never clash
  VP::Value<glm::vec4> _tint;                  // a pass-block value: its offset
  VP::Upload<glm::vec2> _corners;              // a buffer C++ writes: an index
  float _speed = 0;                            // a param, parsed once

  // At load, and again after every rebuild or swap: names in, handles out.
  void bind(VP::Bind &node) override {
    _tint = node.value<glm::vec4>("tint");       // checked: the shader holds a vec4 tint
    _corners = node.upload<glm::vec2>("corners"); // checked: readonly, 8-byte elements
    _speed = node.param<float>("speed");          // checked: the node sets param speed
  }

  // Every frame, before the node's pass.
  void cook(VP::Cook &frame) override {
    const std::span<glm::vec2> corners = frame.write(_corners); // one per invocation
    // … fill the corners, then:
    frame.set(_tint, glm::vec4(1, 0, 0, 1));
  }
};

VP_RECIPE(registry) {
  registry.add<Triangle>("Triangle");          // the name `operator = Triangle` uses
}
```

- **The names meet at load, not at compile time.** The loader reads each shader's SPIR-V (reflection, RA03) and checks every request of `bind` against it, and every param and pass-block field against the node. Nothing is looked up during a frame.
- **A mismatch leaves the node out**, naming why; the rest of the view runs:

  ```text
  {!!!} triangle/view.vlp node triangle: the operator reads param speed, which the node does not set
  ```

- **The handles.** `Value<T>` is a pass-block value C++ sets each frame; `Upload<T>` a buffer C++ writes; `Readback<T>` a buffer a shader wrote, which C++ reads a frame later; `param<T>` a param parsed to `T`. `T` is `float`, `int`, `uint` or a glm vector of them, compared with the GLSL type.
- **The build** compiles each recipe folder: shaders to SPIR-V, C++ to one module per recipe in debug builds, or into `vulpen` in release. They land under `out/build/<preset>/views/<view>/recipes/<recipe>/`.
- **Where:** `src/runtime/Operator.h` is all a recipe's C++ includes (section 15); `src/runtime/Schedule.cpp` loads and checks; `src/runtime/cmake/recipe.cmake` builds.

## 5. The GPU layout

Every shader includes `baseclasses/GpuLayout.glsl`, which declares what every pass shares (RV02):

```glsl
// A buffer is a 64-bit device address in the pass block. Its qualifier is what the node
// declares it does, and the engine places the barriers from it (V10).
layout(buffer_reference, std430) readonly buffer FloatsIn { float at[]; };   // only read
layout(buffer_reference, std430) writeonly buffer FloatsOut { float at[]; }; // only written

// What every pass may read about the frame; the push constant holds its address.
layout(buffer_reference, std430) readonly buffer FrameBlock {
  uvec2 resolution; // of the window in pixels; zero without one
  vec2 cursor;      // in pixels; zero until the input port feeds it
  float time;       // seconds, from the frame index at the run's rate (see decision 3)
  uint index;       // the frame, as the node's C++ counts it
};
layout(push_constant) uniform Push { FrameBlock frame; };
```

A sketch of a draw that uses the frame block and instances:

```glsl
// A vertex shader: gl_InstanceIndex picks the instance, frame.resolution the aspect.
void main() {
  const float aspect = float(frame.resolution.x) / float(frame.resolution.y);
  const vec2 at = pass.corners.at[gl_VertexIndex] + vec2(0.2 * gl_InstanceIndex, 0.0);
  gl_Position = vec4(at.x / aspect, at.y, 0.0, 1.0);
}
```

```ini
[node "triangle"]
invocations    = 3      # vertices of one triangle
instance_count = 4      # four triangles in one draw
```

- **Barriers** follow from the qualifiers: a pass waits for what an earlier pass wrote, with no barrier placed by hand.
- **Memory** follows from who uses a buffer: one C++ writes or reads back lives where the CPU maps it; any other stays on the GPU (VK03). A buffer holds one element per invocation of its writer, and starts zeroed.
- **Draws** run after the dispatches, in graph order, into the window, each blended premultiplied over what came before: an opaque color covers, and alpha lets what is behind show.
- **The frame block** is written once a frame, after the window's image is acquired, so a resized window's size shows at once. Its layout comes from reflection, and a shader whose push constant is anything else is refused.
- **Where:** `src/baseclasses/GpuLayout.glsl`; `src/baseclasses/Pipelines.cpp` reflects and owns the frame block; `src/baseclasses/Engine.cpp` records the frame.

## 6. Live code

In a debug build, saving any file of a running view swaps the change in, and the view keeps running.

```text
$ ./run.sh src/examples/wave/view.vlp --log info
{mod} live code: a save under …/src/examples/wave swaps in
{out} probe: frame    120: +0.7930336 +0.7200354 +0.637223 …
# Save recipes/wave/Wave.comp in any editor:
{nod} wave: pipeline from Wave.comp                       # a new pipeline; the buffers stay
{mod} built in 0.32 s and swapped
{out} probe: frame    180: -0.2531812 -0.36209437 …       # the wave goes on where it was
# Save recipes/wave/Wave.cpp:
{nod} wave: new operator Wave                             # the module is reloaded
{mod} built in 0.65 s and swapped; new modules: wave/wave
{out} probe: frame    420: +0.9234959 +0.8781266 …        # its phase follows the frame index
```

- **How.** The runtime scans the views' folders every 100 ms; a change seen twice runs the build on a thread, so the frame never waits. Between frames, it unloads the modules the build rewrote, reads the manifest again if it changed on disk, and rebuilds.
- **What stays:** pipelines of unchanged SPIR-V, buffers of unchanged shape with their contents, operators of recipes whose module stayed, and the edits typed since the manifest was read.
- **What resets:** a swapped module's operators start over, so state that must survive belongs in params or GPU buffers.
- **A failed build swaps nothing**, and its compiler output is logged; a manifest that does not load keeps the running graph.
- **Where:** debug preset only; `src/runtime/Runtime.cpp` (`Live`, `swap`); the plan is [live code](plans/live-code.md).

## 7. Commands

Every action is a command: a line of text through one port (V06). A command registers with its usage and help, or not at all (RV04). The CLI app lists them with `help`:

```text
quit                             ends the run before its next frame
source <file>                    runs a file's commands, one a line, and stops at the first that fails
log save <file>                  writes the session's edits to a file, which source replays
help                             lists every command, with its usage and what it does
complete <value>...              lists the words that may come next, the last word given being the start of one
clear                            clears the terminal
ls                               lists the view's deploys, nodes and connections
info <node>                      shows a node's words, and the connections it writes and reads
recipe list                      lists the library's recipes, by kind
recipe new draw <name>           writes a draw's first files into the view's recipes from the template: its C++, its pass block and its two shaders
recipe new dispatch <name>       writes a dispatch's first files into the view's recipes from the template: its C++ and its compute shader
recipe drop <recipe> <name>      copies a library recipe into the view, with the recipes it deploys and the contracts they include, and deploys it
view new <file>                  hosts a new, empty view in a folder, and saves its view.vlp
view load <file>                 hosts the view in a folder's view.vlp
view save                        writes the view over its manifest, keeping the comments in it
child add <name> <file>          hosts the view a view.vlp holds, or an empty one that view save writes; a line `<name>: <command>` addresses it
child remove <name>              stops hosting a view; its files stay
child list                       lists the hosted views, with their files, in the order added
node add <name> <word=value>...  adds a node, given the manifest's node words
node remove <node>               removes a node that no connection names
node set <node> <word=value>...  gives a node the words named, clearing those given empty; shader and param words replace them all
connect <name> <port> <port>...  joins the port that writes a buffer to the ports that read it
disconnect <connection>          removes a connection
param set <node> <key> <value>   sets a param of a node
param unset <node> <key>         removes a param of a node
deploy add <name> <recipe>       deploys a recipe of the view under a name: its nodes, as <name>.<node>
deploy remove <deploy>           removes a deploy that no connection names
```

- **Who registers them.** The engine registers the edits, the log, save, hosting and `quit`; every other command is a recipe's: `help`, `complete` and `clear` are the command-line part's, `ls` and `info` the inspect part's, and `recipe …` and `view new`/`view load` the library part's (V05).
- **A usage is its completion.** Its placeholders say what can come there, so `complete` knows:

  ```text
  > complete recipe new d     # the word after `recipe new` that starts with d
  dispatch
  draw
  > complete param set s      # a <node> comes next: the view's nodes starting with s
  spinner
  ```

- **A line may name the view it addresses** as `<name>: <command>`, and `:` alone names the host (section 13).
- **A `<file>` argument arrives resolved:** beside the script that runs the line, or from where vulpen started for a typed line (section 12).
- **Where:** `src/runtime/Commands.cpp`.

## 8. Graph edits

The primitives that change a view are the manifest's own words as commands. Each edit changes a copy of the view; the schedule is rebuilt once before the next frame, for all the edits since the last, so a script is checked as a whole.

```text
# edits.txt, run on a copy of the wave example: ./run.sh wave/view.vlp --source edits.txt
node add probe2 recipe=probe operator=Probe shader=Probe.comp invocations=8 param=step=128 param=every=60
disconnect values                                         # values had one reader, probe
connect values wave.values probe.values probe2.values     # now two: one buffer, read twice
param set probe2 every 30                                 # a param of a node
node set probe2 log=info                                  # the words named; empty clears one
view save                                                 # section 10
log save session.log                                      # section 9
node remove probe2                                        # refused: values still names it
```

```text
{!!!} edits.txt:8: node probe2 is connected through values; disconnect it first
```

The first seven lines applied, and the saved view now holds `[node "probe2"]` and `to = probe2.values`; line 8 stopped the script, and the run with it.

- **What happens.** An edit that breaks the view's shape is refused at once, naming why, and changes nothing: a name used twice, a port in two connections, a connection that would close a cycle. A node whose shaders or C++ disagree is left out at the rebuild, naming its line; the rest runs.
- **The window follows.** An edit that adds the first draw opens the window before the rebuild, and one that removes the last closes it:

  ```text
  {run} a node draws, so the window opens
  {run} no node draws, so the window closes
  ```

- **Errors name the line that last changed the node**, the manifest's or the script's, as `param set wave spare 1` does:

  ```text
  {!!!} spare.txt:1 node wave: param spare: nothing reads it
  ```

- **Other edits:** `param unset <node> <key>` removes a param, and `deploy add` and `deploy remove` change deploys (section 11).

- **Where:** `src/runtime/Edits.cpp`; `src/runtime/Views.cpp` keeps the edited view until the rebuild.

## 9. Scripts and the log

A script is a file of commands, one a line. The session's log is the primitives it ran, so replaying the log rebuilds the graph and the files saved from it (V08).

```bash
./run.sh src/examples/wave/view.vlp --source edits.txt   # before the first frame
```

The script of section 8 saved this log beside itself, since a relative path in a script names a file beside the script:

```text
# session.log: the primitives only, one a line, as typed
node add probe2 recipe=probe operator=Probe shader=Probe.comp invocations=8 param=step=128 param=every=60
disconnect values
connect values wave.values probe.values probe2.values
param set probe2 every 30
node set probe2 log=info
view save
```

```bash
./run.sh copy-of-wave/view.vlp --source session.log   # replays it: the same graph, saved again
```

- **`source <file>`** runs a script from inside another, or from a typed line.

- **Groups.** Each typed or sourced line is a group with the primitives it ran, so a future undo steps back one line at a time. The saved log is flat.
- **Files in a log are named from the log's own folder**, where `source` resolves them, so a log moves with the files it names:

  ```text
  # notes.log, saved at the repo's root while a project was hosted
  child add zz-demo src/examples/zz-demo/view.vlp
  zz-demo: view save
  zz-demo: node add spinner recipe=spinner operator=Spinner shader=Spinner.vert shader=Spinner.frag invocations=3
  ```

- **A failing line stops the script**, naming the file and line, and nothing after it runs:

  ```text
  {!!!} usage.txt:1: node remove does not fit the usage `node remove <node>`
  ```

- **Where:** `src/runtime/Commands.cpp` (`source`, `CommandLog`).

## 10. Saving a view

`view save` writes the view over its `view.vlp`, through a temp file and a rename, so a killed run leaves the old file or the new one (RA04).

- **What it writes:** `[manifest]`, then the deploys, nodes and connections in the view's order, keys aligned. A comment a person wrote stays over the line it was written over, and goes with a node that is removed.
- **It refuses a file that changed on disk** since vulpen read or saved it, naming the file, since saving would lose that change.
- **Hosted views** save their own file: `zz-demo: view save`.
- **Where:** `src/runtime/Manifest.cpp` (`Manifest::save`), `src/baseclasses/Platform.cpp` (`Files::save`).

## 11. Deploys

A view deploys a recipe of its own copies under a name; the loader unfolds it into nodes named `<deploy>.<node>`. Two views of the test fixture:

```ini
# recipes/fill/view.vlp: the recipe's own manifest
[manifest]
version = 1

[node "fill"]
recipe      = fill
shader      = Fill.comp
invocations = 64
param       = amount=1
```

```ini
# view.vlp: deploys fill as a, and reads what it writes
[manifest]
version = 1

[deploy "a"]
recipe = fill               # recipes/fill/, whose view.vlp gives the nodes
param  = fill.amount=2      # the recipe's node fill gets amount=2 in place of 1

[node "sum"]
recipe      = connect
shader      = Sum.comp
invocations = 64

[connection "values"]
from = a.fill.values        # a deployed node's port: <deploy>.<node>.<port>
to   = sum.values
```

Reading and changing it from the CLI:

```text
> ls
deploy a of recipe fill
node sum of recipe connect
connection values from a.fill.values to sum.values
> info a.fill
comes from deploy a, of recipe fill
param = amount=2
> param set a.fill amount 3          # changes the deploy's params; a save writes it there
> node set a.fill invocations=128
{!!!} node a.fill comes from deploy a: param set and unset change its params, and its recipe's view.vlp the rest
```

- **Recipes deploy recipes.** A component's `view.vlp` deploys parts, an app's deploys components and parts, and never in a cycle (RV06). The unfolded view is what the schedule runs; the view as written is what edits and saves change.
- **A recipe of the library runs as a view.** `vulpen src/recipes/apps/cli/view.vlp` loads it as the view `library`, finding its deploys in the library's kind folders. That is how the CLI runs.
- **Where:** `src/runtime/Manifest.cpp` (`Manifest::flatten`).

## 12. The CLI

The `cli` app is three library parts deployed together: `command-line` (a line in, an answer out), `inspect` (`ls`, `info`) and `library` (recipes and views). It runs headless on the terminal.

```text
$ ./run.sh src/recipes/apps/cli/view.vlp
> view new src/examples/zz-demo          # hosts an empty project and saves its view.vlp
src/examples/zz-demo> recipe new draw spinner
wrote Spinner.cpp, Spinner.glsl, Spinner.vert, Spinner.frag in …/src/examples/zz-demo/recipes/spinner
src/examples/zz-demo> node add spinner recipe=spinner operator=Spinner shader=Spinner.vert shader=Spinner.frag invocations=3
src/examples/zz-demo> ls                 # this line went to zz-demo: the prompt says where you are
node spinner of recipe spinner
src/examples/zz-demo> : ls               # `:` sends a line to the CLI itself
deploy command-line of recipe command-line
deploy inspect of recipe inspect
deploy library of recipe library
src/examples/zz-demo> view save
src/examples/zz-demo> child remove zz-demo
> ls                                     # no project now: lines go to the CLI itself
deploy command-line of recipe command-line
…
>                                        # ctrl-D ends the input, and the run
```

- **The prompt** shows the folder of the project your lines go to, from where you started vulpen, as a shell shows its folder; `>` alone means the CLI itself. It shows only when you type at a terminal, so a piped script's output stays clean.
- **Lines go to the newest project.** `view new` and `view load` host a project and make it the one lines go to; `child remove` falls back to the one before, or the CLI. The CLI sends `zz-demo: ls` for your `ls`, so every log line names its view.
- **`name: …`** sends one line to another hosted project, and **`: …`** to the CLI itself.

**Decision 2 of the 2026-10-03 handoff: where a path you type points.** A relative path in a typed line counts from where you started vulpen, which is also what the prompt's path counts from:

```text
$ cd ~/Documents/GitHub/Vulpen && ./run.sh src/recipes/apps/cli/view.vlp
> view new src/examples/zz-demo
src/examples/zz-demo> log save notes.log
# writes ~/Documents/GitHub/Vulpen/notes.log, beside where you started,
# not ~/Documents/GitHub/Vulpen/src/examples/zz-demo/notes.log
src/examples/zz-demo> view new src/examples/other
# hosts ~/Documents/GitHub/Vulpen/src/examples/other: the same text works from any project
```

The other way is a shell's `cd`: paths count from the current project.

```text
# If paths counted from the project, as after `cd src/examples/zz-demo`:
src/examples/zz-demo> log save notes.log           # would write src/examples/zz-demo/notes.log
src/examples/zz-demo> view new ../other            # would be needed for src/examples/other;
src/examples/zz-demo> view new src/examples/other  # this would mean src/examples/zz-demo/src/examples/other
```

The first keeps every typed path meaning one thing wherever you are, and matches the prompt; the second keeps a project's files next to it. Lines from a script always count from the script's folder either way.

- **Where:** `src/recipes/apps/cli/view.vlp`; `src/recipes/parts/command-line/CommandLine.cpp`.

## 13. Hosted views

The view vulpen started with can host other views; each runs with its own schedule, and commands reach it by name (V03). Hosting is session state: the log keeps it, the host's manifest does not.

```text
child add w src/examples/wave/view.vlp     # hosts the wave example as w
w: param set wave amplitude 0.5            # a line for w
w: info wave                               # w's view, as the edits left it
child list                                 # the hosted views, in the order added
w …/src/examples/wave/view.vlp
child remove w                             # gone before the next frame; its files stay
```

- **Each frame** cooks the host, then the hosted views in the order added, and runs all their passes. The window opens while a node of any view draws.
- **A view whose `view.vlp` does not exist yet** starts empty, and its `view save` writes it, folder included: so a log can rebuild a project from nothing.
- **Two views whose folders share a name are refused**, since the build tree mirrors each view by its folder's name.
- **The live scan** watches every hosted view's folder, so saving a project's file swaps it in.
- **A line a command sends** goes to the same view as the line that ran it, unless it names one; a script's lines each stand alone.
- **Where:** `src/runtime/Views.cpp`.

## 14. The library

The `library` part works on recipe and view folders through the file port. It runs in the CLI.

**`recipe list`** lists the library's recipes as `kind/name`:

```text
> recipe list
apps/cli
apps/ide
components/dock
…
parts/command-line
parts/inspect
…
```

**`recipe new draw <name>` and `recipe new dispatch <name>`** write a recipe's first files into the current project's `recipes/<name>/`, from the template in `src/recipes/parts/library/template/`. The class takes the name in CamelCase, and the node its own name. They build at once and do nothing yet; every line the engine offers waits in them, commented.

```text
src/examples/zz-demo> recipe new draw spinner
wrote Spinner.cpp, Spinner.glsl, Spinner.vert, Spinner.frag in …/zz-demo/recipes/spinner
src/examples/zz-demo> recipe new dispatch fill
wrote Fill.cpp, Fill.comp in …/zz-demo/recipes/fill
```

**Decision 1 of the 2026-10-03 handoff: the kind is a word of the command.** The plan wrote `recipe new <name> <kind>`. Built that way, the kind would be a free word after the name; built as two commands, it is part of the command, so the CLI knows it:

```text
# Built: two commands, `recipe new draw <name>` and `recipe new dispatch <name>`
> complete recipe new d
dispatch                      # the CLI offers both kinds
draw
> help                        # and help explains each on its own line
recipe new draw <name>       writes a draw's first files … its C++, its pass block and its two shaders
recipe new dispatch <name>   writes a dispatch's first files … its C++ and its compute shader

# The plan's form: one command, `recipe new <name> <value>`
> complete recipe new spinner d
                              # nothing: <value> is any word, so the CLI cannot offer draw or dispatch
> recipe new spinner drow
{!!!} …                       # a typo is found only when the command runs
```

The engine's placeholder kinds are a closed list (`name`, `node`, `port`, `file`, …) that completion reads; a `kind` placeholder would add a library-only word to the engine (V05). As two commands, nothing in the engine changes.

**`recipe drop <recipe> <name>`** copies a library recipe into the project, with the recipes it deploys and the contracts their shaders include, then deploys it under the name:

```text
src/examples/zz-demo> recipe drop inspect look
copied inspect into …/zz-demo/recipes                # the project's own copy now (V03)
# the view now holds [deploy "look"] recipe = inspect; the live build compiles the copy
```

- A copy the project has already stays as it is: the project owns its copies, and a later library edit never reaches them.
- A recipe's contracts (`#include "contracts/Rect.glsl"`) are copied to `recipes/contracts/` with it.

**`view new <folder>` and `view load <folder>`** host a project in a folder, named for the folder, and make it the current one. `view new` also saves its empty `view.vlp`, and refuses a folder that holds one; `view load` refuses a folder that holds none. In the log, both are a `child add`, and `view new` a `view save` too.

- **Where:** `src/recipes/parts/library/Library.cpp`.

## 15. What a recipe's C++ can reach

A recipe's C++ includes `runtime/Operator.h`, and `runtime/View.h` when it reads the graph. The template shows every call, commented; these are the ports besides the pass block of section 4.

**Its own commands.** A node registers commands in `bind`; they last while the node runs, and a node that goes takes them along. The inspect part's `ls`, shortened:

```cpp
void bind(VP::Bind &node) override {
  _ls = node.command("ls", "lists the view's deploys, nodes and connections"); // usage, help
}
void command(VP::Call &call) override {
  if (!call.is(_ls))
    return;
  for (const VP::Node &n : call.view().nodes)   // the view the line addresses, as edited
    call.reply(std::format("node {} of recipe {}", n.name, n.recipe)); // the answer
  // call.commands().send("w: param set wave amplitude 0.5"); // a line inside this one:
  //   it joins this command's log group, and a refusal fails this command too
}
```

A command name is one command: a second node registering it is refused, as in section 14's drop.

**The terminal** (the command-line part's whole job):

```cpp
void cook(VP::Cook &frame) override {
  VP::TerminalPort &terminal = frame.terminal();
  for (const std::string &line : terminal.lines())      // what was typed since last frame
    if (const std::string answer = frame.commands().send(line); !answer.empty())
      terminal.print(answer);                           // a refusal is logged, answers nothing
  if (terminal.ended())                                 // ctrl-D, or a piped script's end
    frame.commands().send("quit");
  // terminal.prompt("> ");                             // only where a person types
}
```

**Files.** A file a node holds is opened in `bind` and watched; any other goes by absolute path. A sketch, as the palette part would read its theme:

```cpp
void bind(VP::Bind &node) override {
  _theme = node.file("theme.ini");        // in this recipe's folder: a relative path
  _folder = node.folder();                // that folder, absolute
}
void cook(VP::Cook &frame) override {
  const std::string_view theme = frame.files().text(_theme); // read again once it changed
  // frame.files().save(_theme, "text");                     // through a temp file (RA04)
  // frame.files().list(_folder);                           // names, folders ending in /
  // frame.files().read(path);                              // any file, by absolute path
}
```

**The frame index and the log.** `frame.index()` is the frame, as the frame block's `index` is; `frame.log(VP::Level::info, "text")` logs at the node's level (V09).

- **What a recipe cannot reach today:** the engine's objects, the OS, or another recipe's code (RV00, live code rule 2). The [native C++ plan](plans/native-cpp.md) proposes opening that.
- **Where:** `src/runtime/Operator.h`; the template in `src/recipes/parts/library/template/`.

## 16. When something is wrong

Every mistake surfaces at load or at its line, naming its cause (A02):

| Mistake | What you see | What runs |
| --- | --- | --- |
| an unknown word or section, a word given twice | `view.vlp:19: unknown word invocatons in a node; its words are …` | nothing: the load stops |
| C++ and a shader disagree on a name or a type | `node fill: the operator sets amount as a uint, but the shader declares a float` | the rest of the view; that node is left out |
| a param nothing reads, a field nothing sets | `node wave: param spare: nothing reads it` | the rest of the view |
| an edit that cannot apply | `refuse.txt:2: node wave is connected through values; disconnect it first` | nothing changes; a script stops there |
| a command that does not exist, or does not fit its usage | `unknown command frobnicate; the commands are …` | nothing changes |
| a C++ module whose build failed | the compiler's output | the running code stays |
| two nodes registering one command | `node look.inspect: command \`ls\` registers twice` | the second node is left out |
| a view whose recipe is not built yet | `recipe zz-demo/inspect has no C++ built: …/recipe.so is missing` | until the live build compiles it |
