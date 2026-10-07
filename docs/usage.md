# Using Vulpen

How each part of Vulpen works today, shown the way you use it. Each section has a session, a manifest or a node's code, with comments that say what happens, where it holds and what comes out. Every session and output here ran on 2026-10-03, on the tree after `9656e19` that makes every node a folder (RV08); the log's time column is left out, and long paths are shortened with `…`. Code marked as a sketch shows a call's shape, not a file in the tree. What is planned but not built yet is in the [IDE port](plans/ide-port.md) and the [CLI examples](plans/cli-examples.md); how nodes, folders and recipes came to be this way is in [nodes and folders](plans/nodes-and-folders.md).

1. [Words](#1-words)
2. [Running a view](#2-running-a-view)
3. [The manifest](#3-the-manifest)
4. [Nodes and folders: C++ and GLSL](#4-nodes-and-folders-c-and-glsl)
5. [The GPU layout](#5-the-gpu-layout)
6. [Live code](#6-live-code)
7. [Commands](#7-commands)
8. [Graph edits](#8-graph-edits)
9. [Scripts and the log](#9-scripts-and-the-log)
10. [Saving a view](#10-saving-a-view)
11. [Recipes: drop and sync](#11-recipes-drop-and-sync)
12. [The CLI](#12-the-cli)
13. [Child views](#13-child-views)
14. [The library](#14-the-library)
15. [What a node's C++ can reach](#15-what-a-nodes-c-can-reach)
16. [When something is wrong](#16-when-something-is-wrong)

## 1. Words

| Word | What it is | Where |
| --- | --- | --- |
| view | a project: a folder with a `view.vlp`, a folder for each node, and the views it hosts. It loads, runs and moves as a whole (V03) | `src/examples/<name>/` |
| manifest | a view's `view.vlp`: its nodes and their files, its connections, and the views it hosts (V00) | `<view>/view.vlp` |
| node | one `[node]` of a manifest and the folder its name names: a C++ class, shaders, or both, and the nodes inside it (RV08) | `<view>/<node>/` |
| connection | one buffer: one node's port writes it, other nodes' ports read it | the manifest |
| recipe | a node of the library with the nodes inside it, which a drop copies into a view | `src/recipes/<kind>/<name>/` |
| library | the recipes views copy from: parts hold code; components and apps only use other recipes (RV06) | `src/recipes/{parts,components,apps}/` |
| child view | a view another view's manifest names, which runs on its own schedule; commands reach it by name | `[view "<name>"]` |
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
{run} vulpen 9656e19                                                # the commit it was built from (RC07)
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

[node "wave"]               # a node, by a name unique in the view; its folder is wave/ (RV08)
operator    = Wave          # the C++ class wave/Wave.cpp registers; leave it out for shaders alone
file        = Wave.comp     # the files in wave/: a .comp makes a dispatch, a .vert and a .frag a draw
file        = Wave.cpp
invocations = 1024          # a dispatch's threads
param       = amplitude=1.0 # a value the shader's pass block or the C++ reads, by name
param       = speed=0.02

[node "probe"]
operator    = Probe
file        = Probe.comp
file        = Probe.cpp
invocations = 8
param       = step=128
param       = every=60
log         = info          # this node's own log level (V09)

[connection "values"]       # one buffer
from = wave.values          # the node.port that writes it
to   = probe.values         # the node.port that reads it; repeat `to` for more readers
```

- **The words.** `[node]` takes `recipe`, `operator`, `file`, `invocations`, `vertex_count`, `instance_count`, `param`, `image` and `log`; `[connection]` takes `from` and `to`; `[view]` takes `file` (section 13). A dispatch counts its `invocations`; a draw counts its `vertex_count`, and `instance_count = 64` draws 64 instances, one when left out (VK04: `vkCmdDraw(vertexCount, instanceCount, …)`). `recipe` names where a dropped node came from (section 11).
- **The folder decides a node's files.** The `file` lines list what the node's folder holds, and vulpen keeps them so: a file put in `wave/` is one of the node's at the next load, edit or live scan, one taken out is no longer, and `view save` writes the list. A command never names a file. A folder that no node names is a warning at load, since every folder is a node.
- **Loading runs edits.** Each section goes through the same edits a command makes (section 8), so a loaded view and a typed one pass the same checks.
- **Order.** Nodes run writers before readers, as the connections order them, then in the manifest's order.
- **A mistake names its line** and stops the load (A02):

  ```text
  {!!!} …/wave/view.vlp:11: unknown word invocatons in a node; its words are recipe, operator, file, invocations, vertex_count, instance_count, param, image and log
  ```

- **Comments stay.** `view save` writes the manifest back with the comments a person wrote (section 10).
- **Where:** `src/runtime/Manifest.cpp` reads and writes it, and keeps each node's files; `src/runtime/Edits.cpp` checks each word.

## 4. Nodes and folders: C++ and GLSL

A node is the folder its name names (RV08), and its files are what that folder holds. The triangle, whose C++ moves the corners and whose shaders draw them:

```text
src/examples/triangle/
  view.vlp                # [node "triangle"]: operator = Triangle, its four files, vertex_count = 3
  triangle/               # the node's folder
    Triangle.cpp          # the operator: C++ that runs every frame
    Triangle.glsl         # the pass block both shaders include
    Triangle.vert         # the vertex shader
    Triangle.frag         # the fragment shader
```

- **Nodes inside nodes.** `[node "ui.panel"]` is the folder `ui/panel/`, beside the files of `ui` itself in `ui/`, and its port `rects` is `ui.panel.rects`. Every node has a section, a group too: `[node "ui"]` with no words runs nothing and holds the nodes inside it, and its log level reaches them (V09).
- **No two nodes share a folder**, so two nodes running one class each hold a copy of it, as two drops of a recipe do (section 11).

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
    const std::span<glm::vec2> corners = frame.write(_corners); // one per vertex
    // … fill the corners, then:
    frame.set(_tint, glm::vec4(1, 0, 0, 1));
  }
};

VP_OPERATORS(registry) {                       // once in the folder: its module's operators
  registry.add<Triangle>("Triangle");          // the name `operator = Triangle` uses
}
```

Data moves by different routes, told apart by who writes it and how much there is, not by how often they fire:

| From, to | How | In the examples |
| --- | --- | --- |
| you, a shader | a param named like a pass-block field: the engine writes it at load and after each edit, with no C++ | wave's `amplitude` |
| you, C++ | `node.param<T>`, parsed in `bind`, which runs again after an edit: a number, or a word as `std::string`, such as a file's name | the triangle's `speed`; the font's `face` |
| C++, a shader, one value | `node.value<T>`, set with `frame.set` in `cook`, each frame | the triangle's `tint` |
| C++, a shader, one per invocation | `node.upload<T>`, filled through `frame.write` in `cook`; it holds until written again | the triangle's `corners` |
| C++, a shader, an image | `node.texture<T>`, filled once through `frame.upload` in `cook`, in the format its `image` word gives; it keeps its pixels through rebuilds | the fail-loud fixture's `picture` |
| C++, another node's shader | `node.upload<T>(port, count)` at a port no shader of the node holds, through a connection; the reader's struct is checked against C++'s | the fail-loud fixture's `maker` to `shape` |
| a shader, C++ | `node.readback<T>`, read a frame after the GPU wrote it | the probe's `samples` |
| a shader, a shader | a connection, which never leaves the GPU (VK03) | `wave.values` to `probe.values` |
| C++, C++ | a connection: `node.output<T>` in the writer, `node.input<T>` in each reader, one object both hold by reference (section 15) | the fail-loud fixture's `give.count` to `take.count` |
| the engine, both | the frame block for shaders (section 5); `frame.index()` and `frame.resolution()`, the window's size, zero without one, for C++ | `frame.resolution` |

- **The names meet at load, not at compile time.** The loader reads each shader's SPIR-V (reflection, RA03) and checks every request of `bind` against it, and every param and pass-block field against the node. Nothing is looked up during a frame.
- **A mismatch leaves the node out**, naming why; the rest of the view runs:

  ```text
  {!!!} triangle/view.vlp:10 node triangle: the operator reads param speed, which the node does not set
  ```

- **The handles.** `T` is `float`, `int`, `uint` or a glm vector of them, compared with the GLSL type. A buffer's element may also be a struct that names its members once, which the loader checks against the shader's struct by name, type and offset; a member put elsewhere is refused (`shapes: member size is a float at byte 12 in C++, but a float at byte 8 in the shader`). The fail-loud fixture's `shape`:

  ```cpp
  struct Shape { // the shader's: struct Shape { vec2 at; float size; uint sides; };
    glm::vec2 at;
    float size = 0;
    std::uint32_t sides = 0;

    static constexpr auto members() {
      return std::array{VP_MEMBER(Shape, at), VP_MEMBER(Shape, size), VP_MEMBER(Shape, sides)};
    }
  };
  // in bind: _shapes = node.upload<Shape>("shapes");
  ```
- **The build** compiles each node's folder: its shaders to SPIR-V, and its `.cpp` files to one module in debug builds, or into `vulpen` in release. They land under `out/build/<preset>/views/<view>/<the node's folder>/`. One file of the folder registers its operators with `VP_OPERATORS`.
- **Where:** `src/runtime/Operator.h` is all a node's C++ includes (section 15); `src/runtime/Schedule.cpp` loads, `Binder.cpp` binds and `Checks.cpp` checks; `src/runtime/cmake/nodes.cmake` builds.

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
  vec2 cursor;      // the pointer, in pixels; zero until it moves
  float time;       // seconds, from the frame index at the run's rate
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
vertex_count   = 3      # vertices of one triangle
instance_count = 4      # four triangles in one draw
```

`instance_count` may also name a port: the draw then runs as many instances as that buffer's used length, each frame. C++ that writes the buffer sets it, and asks for room when it writes more than one element per vertex:

```cpp
_items = node.upload<glm::vec2>("items", 256);  // room for 256, never fewer than one per vertex
// each frame:
const std::span<glm::vec2> items = frame.write(_items, count); // count instances this frame
```

```ini
instance_count = items  # one triangle per element C++ wrote this frame
```

Writing more than the room is refused, and the node's operator stops (`the operator writes 8 elements of shapes, which holds 4`). A buffer a shader writes counts all its elements, one per invocation of its writer.

Images live in set 0 (RV02). A pass block names one by a `Texture`, its slot in `textures[]`, and `sample_linear` samples it at level 0, so every stage samples alike (GLSL02). A node's C++ fills an image once. The loader gives its slot to the node's own `Texture` of that name, or, through a connection, to another node's:

```glsl
layout(set = 1, binding = 0) uniform Pass {
  Texture atlas; // C++: node.texture<T>("atlas"), or a connection from a port C++ fills
} pass;
// in main: color = sample_linear(pass.atlas, uv);
```

```ini
[node "font"]
operator = Font
image    = atlas=R8_UNORM   # one 8-bit channel, which shaders read as 0 to 1
```

```cpp
_atlas = node.texture<std::uint8_t>("atlas");    // in bind: the pixel R8_UNORM takes
if (frame.empty(_atlas))                         // in cook: at first, and after image clear
  frame.upload(_atlas, pixels, {width, height});
```

The `image` word gives an image its format, by Vulkan's name (VK04), and C++ fills it with that format's pixel, byte for byte; the loader refuses any other (`the operator fills picture with uint8 pixels, but its format R32_SFLOAT takes float`). An image no word names is `R8G8B8A8_UNORM`.

| Format | C++ pixel |
| --- | --- |
| `R8_UNORM`, `R8G8_UNORM`, `R8G8B8A8_UNORM`, `R8G8B8A8_SRGB` | `std::uint8_t`, `glm::u8vec2`, `glm::u8vec4`, `glm::u8vec4` |
| `R16_SFLOAT`, `R16G16_SFLOAT`, `R16G16B16A16_SFLOAT` | `std::uint16_t`, `glm::u16vec2`, `glm::u16vec4`, each a 16-bit float's bits, as `glm::packHalf1x16` gives them |
| `R32_SFLOAT`, `R32G32_SFLOAT`, `R32G32B32A32_SFLOAT` | `float`, `glm::vec2`, `glm::vec4` |

The upload makes the image the size and format given, and the frame copies the pixels in before its passes. The image keeps them through rebuilds while the node, the port and the format stay, so a node uploads once. They stay with the node that fills them, so a connection removed and made again finds them, until `image clear <port>` drops them, named by the port that fills them or one that samples them. Until a node fills its image, and after a clear, its `Texture` holds 0, which means unbound and samples as nothing, and `frame.empty` says so to the node's C++. A `Texture` nothing fills, an image nothing samples, a format there is not or one this GPU cannot sample filtered, and pixels that do not make the size given are refused.

- **Barriers** follow from the qualifiers: a pass waits for what an earlier pass wrote, with no barrier placed by hand.
- **Memory** follows from who uses a buffer: one C++ writes or reads back lives where the CPU maps it; any other stays on the GPU (VK03). A buffer holds one element per invocation of its writer, a dispatch's thread or a draw's vertex, and starts zeroed.
- **Draws** run after the dispatches, in graph order, into the window, each blended premultiplied over what came before: an opaque color covers, and alpha lets what is behind show.
- **The frame block** is written once a frame, after the window's image is acquired, so a resized window's size shows at once. Its layout comes from reflection, and a shader whose push constant is anything else is refused.
- **Shared GLSL** (GLSL01): `sample_nearest` samples the texel nearest `uv`, for an image drawn a texel a pixel, as a glyph is; `quad_corner(gl_VertexIndex)` gives the corners of a quad's two triangles, so a draw of `vertex_count = 6` places a rectangle an instance; `pixel_clip` puts a point in pixels from the window's top left into clip space.
- **Where:** `src/baseclasses/GpuLayout.glsl`; `src/baseclasses/Shader.cpp` reflects; `src/baseclasses/Pipelines.cpp` owns the frame block and set 0; `src/baseclasses/Engine.cpp` records the frame, the copies into images first.

## 6. Live code

In a debug build, saving any file of a running view swaps the change in, and the view keeps running.

```text
$ ./run.sh src/examples/wave/view.vlp --log info
{mod} live code: a save under …/src/examples/wave swaps in
{out} probe: frame    120: +0.7930336 +0.7200354 +0.637223 …
# Save wave/Wave.comp in any editor:
{nod} wave: pipeline from Wave.comp                       # a new pipeline; the buffers stay
{mod} built in 0.32 s and swapped
{out} probe: frame    180: -0.2531812 -0.36209437 …       # the wave goes on where it was
# Save wave/Wave.cpp:
{nod} wave: new operator Wave                             # the folder's module is reloaded
{mod} built in 0.65 s and swapped; new modules: wave/wave
{out} probe: frame    240: -0.97651756 -0.98245066 …      # its phase follows the frame index
```

- **How.** The runtime scans the views' folders every 100 ms; a change seen twice runs the build on a thread, so the frame never waits. Between frames, it unloads the modules the build rewrote, reads the manifest again if it changed on disk, takes each node's files from its folder, and rebuilds.
- **What stays:** pipelines of unchanged SPIR-V, buffers of unchanged shape with their contents, operators whose folder's module stayed, and the edits typed since the manifest was read.
- **What resets:** a swapped module's operators start over, so state that must survive belongs in params or GPU buffers.
- **A failed build swaps nothing**, and its compiler output is logged; a manifest that does not load keeps the running graph.
- **Where:** debug preset only; `src/runtime/Runtime.cpp` (`Live`, `swap`); the plan is [live code](plans/live-code.md).

## 7. Commands

Every action is a command: a line of text through one port (V06). A command registers with its usage and help, or not at all (RV04). The CLI app lists them with `help`:

```text
quit                               ends the run before its next frame
source <file>                      runs a file's commands, one a line, and stops at the first that fails
log save <file>                    writes the session's edits to a file, which source replays
input key down <value>             presses a key, named by the character it prints, or as space, enter or f1, …
input key up <value>               lets a key go
input text <value>...              types the words, joined by single blanks
input pointer <value> <value>      moves the pointer to x and y, in pixels from the window's top left
input button down <value>          presses a pointer button: left, right or middle
input button up <value>            lets a pointer button go
input wheel <value> <value>        turns the wheel by x and y
input focus on                     gives the window focus
input focus off                    takes focus from the window
help                               lists every command, with its usage and what it does
complete <value>...                lists the words that may come next, the last word given being the start of one, or "" for one not begun
clear                              clears the terminal
ls                                 lists the views the view hosts, its nodes and its connections
info <node>                        shows a node's words, and the connections it writes and reads
recipe list                        lists the library's recipes, by kind
node new draw <name>               adds a draw node, and writes its first files from the template into its folder: …
node new dispatch <name>           adds a dispatch node, and writes its first files from the template into its folder: …
recipe drop <recipe> <name>        copies a library recipe into the view as a node of that name, with the nodes inside it …
recipe sync <node>                 brings a dropped recipe up to the library's, while what it copied is unchanged; …
view new <file>                    hosts a new, empty view in a folder, and saves its view.vlp
view load <file>                   hosts the view in a folder's view.vlp
view save                          writes the view over its manifest, keeping the comments in it
child list                         lists the hosted views, with their files, in the order hosted
image clear <port>                 drops the pixels of an image a node's C++ fills, named by that port or one that …
node add <name> [<word=value>...]  adds a node, given the manifest's node words; its files are what its folder holds
node remove <node>                 removes a node that no connection names and no node is inside; its folder stays
node set <node> <word=value>...    gives a node the words named, clearing those given empty; param and image words …
connect <name> <port> <port>...    joins the port that writes a buffer to the ports that read it
disconnect <connection>            removes a connection
param set <node> <key> <value>     sets a param of a node
param unset <node> <key>           removes a param of a node
child add <name> <file>            hosts the view a view.vlp holds, or an empty one that view save writes; …
child remove <name>                stops hosting a view; its files stay
```

- **Who registers them.** The engine registers the edits, the log, save, `child list`, `image clear`, `input` and `quit`; every other command is a node's: `help`, `complete` and `clear` are the command-line part's, `ls` and `info` the inspect part's, and `recipe …`, `node new …` and `view new`/`view load` the library part's (V05).
- **A usage is its completion.** Its placeholders say what can come there, so `complete` knows. The last may end in `...`, one argument or more, and stand in brackets, which a line may leave out: `node add ui` adds a group.

  ```text
  > complete node new d       # the word after `node new` that starts with d
  dispatch
  draw
  > complete param set s      # a <node> comes next: the view's nodes starting with s
  spinner
  ```

- **A line may name the view it addresses** as `<name>: <command>`, and `:` alone names the one vulpen started with (section 13).
- **A `<file>` argument arrives resolved:** beside the script that runs the line, or from where vulpen started for a typed line (section 12).
- **Where:** `src/runtime/Commands.cpp`.

## 8. Graph edits

The primitives that change a view are the manifest's own words as commands. Each edit changes a copy of the view; the schedule is rebuilt once before the next frame, for all the edits since the last, so a script is checked as a whole.

```text
# edits.txt, run on a copy of the wave example: ./run.sh wave/view.vlp --source edits.txt
disconnect values                                         # values had one reader, probe
node remove probe                                         # out of the graph; probe/ stays
node add probe operator=Probe invocations=8 param=step=128 param=every=60   # its files: probe/'s
connect values wave.values probe.values                   # one buffer; more ports may read it
param set probe every 30                                  # a param of a node
node set probe log=info                                   # the words named; empty clears one
view save                                                 # section 10
log save session.log                                      # section 9
node remove wave                                          # refused: values still names it
```

```text
{!!!} …/edits.txt:9: node wave is connected through values; disconnect it first
```

The first eight lines applied, and the saved view now holds `param = every=30`, the probe's files listed as they were; line 9 stopped the script, and the run with it.

- **What happens.** An edit that breaks the view's shape is refused at once, naming why, and changes nothing: a name used twice, a port in two connections, a connection that would close a cycle, a node inside no node, a file named by a command. A node whose shaders or C++ disagree is left out at the rebuild, naming its line; the rest runs.
- **Files stay.** No edit writes or deletes a file: `node remove` leaves the folder, and `node add` takes the files its folder holds.
- **The window follows.** An edit that adds the first draw opens the window before the rebuild, and one that removes the last closes it:

  ```text
  {run} a node draws, so the window opens
  {run} no node draws, so the window closes
  ```

- **Errors name the line that last changed the node**, the manifest's or the script's, as `param set wave spare 1` does:

  ```text
  {!!!} spare.txt:1 node wave: param spare: nothing reads it
  ```

- **Other edits:** `param unset <node> <key>` removes a param, and `child add` and `child remove` change the views a view hosts (section 13).

- **Where:** `src/runtime/Edits.cpp`; `src/runtime/Views.cpp` keeps the edited view until the rebuild.

## 9. Scripts and the log

A script is a file of commands, one a line. The session's log is the primitives it ran, so replaying the log rebuilds the graph and the files saved from it (V08).

```bash
./run.sh src/examples/wave/view.vlp --source edits.txt   # before the first frame
```

The script of section 8 saved this log beside itself, since a relative path in a script names a file beside the script:

```text
# session.log: the primitives only, one a line, as typed
disconnect values
node remove probe
node add probe operator=Probe invocations=8 param=step=128 param=every=60
connect values wave.values probe.values
param set probe every 30
node set probe log=info
view save
```

```bash
./run.sh copy-of-wave/view.vlp --source session.log   # replays it: the same graph, saved again
```

- **`source <file>`** runs a script from inside another, or from a typed line.
- **Files are not commands.** A log rebuilds the graph; the files its nodes hold must be in their folders, as a copy of the view brings them (V03).
- **Groups.** Each typed or sourced line is a group with the primitives it ran, so a future undo steps back one line at a time. The saved log is flat.
- **Files in a log are named from the log's own folder**, where `source` resolves them, so a log moves with the files it names:

  ```text
  # notes.log, saved at the repo's root while a project was hosted
  child add zz-demo src/examples/zz-demo/view.vlp
  zz-demo: view save
  zz-demo: node add spinner operator=Spinner vertex_count=3
  ```

- **A failing line stops the script**, naming the file and line, and nothing after it runs:

  ```text
  {!!!} usage.txt:1: node remove does not fit the usage `node remove <node>`
  ```

- **Where:** `src/runtime/Commands.cpp` (`source`, `CommandLog`).

## 10. Saving a view

`view save` writes the view over its `view.vlp`, through a temp file and a rename, so a killed run leaves the old file or the new one (RA04).

- **What it writes:** `[manifest]`, then the views it hosts, the nodes with the files their folders hold, and the connections, in the view's order, keys aligned. A comment a person wrote stays over the line it was written over, and goes with a node that is removed.
- **It refuses a file that changed on disk** since vulpen read or saved it, naming the file, since saving would lose that change.
- **Hosted views** save their own file: `zz-demo: view save`.
- **Where:** `src/runtime/Manifest.cpp` (`Manifest::save`), `src/baseclasses/Platform.cpp` (`Files::save`).

## 11. Recipes: drop and sync

A recipe is a node of the library, with the nodes inside it. A drop copies it into the view as a node under the name you give, and the copy is yours: every edit works, as if you had typed it (V02).

```text
src/examples/zz-demo> recipe drop inspect look
copied inspect into …/src/examples/zz-demo/look, as node look of recipe inspect@abb3239c
src/examples/zz-demo> info look
recipe = inspect@abb3239c          # where it came from, and a fingerprint of what was copied
operator = Inspect
file = Inspect.cpp                 # copied into look/, which the live build compiles
```

- **What a drop copies:** the recipe's files into the node's folder, the recipes it uses into the folders of the nodes inside it, and the contracts their files include into the view's `contracts/` (RV05). A contract the view has already stays as it is.
- **Nothing links back.** A later edit of the library reaches the copy only through `recipe sync` (V03).

`recipe sync <node>` brings a drop up to the library's version while what it copied is unchanged:

| | Counts as a change | A sync keeps it |
| --- | --- | --- |
| the files of the node's folder, and of the nodes inside it | yes | — |
| the nodes' operators, counts, and the connections between them | yes | — |
| a param's value | no | yes, while the library's version has the param; one it let go goes, and the sync names it |
| `log`, and connections to the rest of the view | no | yes |

```text
# a project holding d, a drop of a component c of two parts, after the library changed a part:
recipe sync d
synced node d with the library's c: a8fbfa6e is now b6454ca3
recipe sync d
node d is as the library's c is
# after an edit of d/a/a.ini, and another change in the library:
recipe sync d
{!!!} node d changed since it was dropped from c, and a sync would lose that; drop c beside it to take the library's, and make the change there again
```

- **Refused, too:** a sync the library's version would take a connection from: one from the rest of the view to a node the library let go.
- **The fingerprint** hashes what was copied, as FNV-1a over 64 bits folded to 32, so it is the same on every machine (C01).

In the library, a recipe uses others as they are, which keeps one copy of each part's code (V11). The CLI app is three parts used that way:

```ini
# src/recipes/apps/cli/view.vlp
[node "cli"]                 # the recipe's own node: apps/cli/

[node "cli.command-line"]
recipe = command-line        # no fingerprint: the library's own part, unfolded at load
```

- **A recipe is one node named like its folder**, and the nodes inside it; a node that uses it becomes that node, under its own name, and takes `param` and `log` only. Uses never form a cycle (RV06).
- **Only the library uses recipes so.** A view holds its own copies (V03), so a view's node that names a recipe without a fingerprint is refused, naming the drop that copies it.
- **Where:** `src/recipes/parts/library/Library.cpp` (drop and sync); `src/runtime/Manifest.cpp` (`Manifest::flatten` unfolds a recipe the library uses).

## 12. The CLI

The `cli` app is three library parts used together: `command-line` (a line in, an answer out), `inspect` (`ls`, `info`) and `library` (recipes, nodes and views). It runs headless on the terminal.

```text
$ ./run.sh src/recipes/apps/cli/view.vlp
> view new src/examples/zz-demo          # hosts an empty project and saves its view.vlp
src/examples/zz-demo> node new draw spinner
wrote Spinner.cpp, Spinner.glsl, Spinner.vert, Spinner.frag in …/src/examples/zz-demo/spinner, and added node spinner
src/examples/zz-demo> ls                 # this line went to zz-demo: the prompt says where you are
node spinner
src/examples/zz-demo> : ls               # `:` sends a line to the CLI itself
view zz-demo from …/src/examples/zz-demo/view.vlp
node cli
node cli.command-line of recipe command-line
node cli.inspect of recipe inspect
node cli.library of recipe library
src/examples/zz-demo> view save
src/examples/zz-demo> child remove zz-demo
> ls                                     # no project now: lines go to the CLI itself
node cli
…
>                                        # ctrl-D ends the input, and the run
```

- **The prompt** shows the folder of the project your lines go to, from where you started vulpen, as a shell shows its folder; `>` alone means the CLI itself. It shows only when you type at a terminal, so a piped script's output stays clean.
- **Lines go to the newest project.** `view new` and `view load` host a project and make it the one lines go to; `child remove` falls back to the one before, or the CLI. The CLI sends `zz-demo: ls` for your `ls`, so every log line names its view.
- **`name: …`** sends one line to another hosted project, and **`: …`** to the CLI itself.
- **A new node builds at once.** In a debug build, the live build compiles `spinner/` a moment after `node new` wrote it, and swaps it in; until then the node is left out, naming the module it waits for (section 16).

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

**The terminal in a window.** The `ide` app is, so far, the theme, the font, a dock that seats the `terminal` component, in a panel, in the window's bottom row, and `inspect` and `library` as in the CLI; the room above waits for the editor and graph panels. The terminal is the same `command-line` part with `param = on=window`: it reads the keyboard instead of standard input, and shows above the line typed what lines answer and the whole log, errors in red and warnings in amber, so a refused line names its cause there, a frame later.

```text
$ ./run.sh src/recipes/apps/ide/view.vlp
```

- **Keys:** typing inserts at the caret, which Left, Right, Home and End move; Backspace and Delete erase, Enter sends the line, and Up and Down step through the lines sent.
- **Tab** completes as a shell does: the word at the caret takes what every candidate shares, a word only one candidate fits is finished with a blank, and several are listed above the line until the next key. A placeholder such as `<name>` is listed but never typed in. Tab sends `complete`, which takes `""` for a word not begun.
- **A press on a completion** types the rest of its word, by sending `input text <rest>`, as if you typed it; no blank follows, unlike Tab.
- **The seam** above the terminal lights up under the pointer, and a drag moves it, the terminal following. Letting go sends one `param set ide.dock.split ratio 0.416`, so the log keeps one line a drag, and a replay or a save puts the seam back.
- **The tab** over the terminal names it and the view its lines go to: `terminal`, and `terminal - demo` after `view new demo`. It is the command-line's, from its `title` param, and has no command, so a press on it sends nothing.
- **`clear`** empties the scrollback, which keeps the last 1,000 rows. A row wider than the terminal is cut, and a long line scrolls to keep the caret in view.

- **Where:** `src/recipes/apps/cli/view.vlp`, `src/recipes/apps/ide/view.vlp`, `src/recipes/components/terminal/view.vlp`, `src/recipes/components/dock/view.vlp`, `src/recipes/components/panel/view.vlp`; `src/recipes/parts/command-line/CommandLine.cpp`.

## 13. Child views

A view's manifest names the views it hosts (V03). Each runs with its own schedule, and commands reach it by name:

```ini
# host/view.vlp
[manifest]
version = 1

[view "w"]
file = ../wave/view.vlp      # left out, w/view.vlp beside this manifest
```

```text
$ ./run.sh host/view.vlp --source host/lines.txt --frames 61 --log info
# lines.txt: w: param set wave amplitude 0.5, then child list
w …/wave/view.vlp                          # child list: the hosted views, in the order hosted
{run} child w: hosted from …/wave/view.vlp
{out} probe: frame     60: +0.4141257 +0.442029 …   # half the wave of section 2
```

- **Hosting follows the manifest.** Loading a view hosts the views it names; `child add` and `child remove` are edits of the view a line addresses, which `view save` writes, and the log keeps. A child's own `[view]` sections host views in turn, and removing a child takes them along.
- **`child remove <name>` reaches the view that hosts the child** from wherever the line goes, so `child remove zz-demo` typed in zz-demo closes it.
- **Each frame** cooks the host, then the hosted views in the order hosted, and runs all their passes. The window opens while a node of any view draws.
- **A view whose `view.vlp` does not exist yet** starts empty, and its `view save` writes it, folder included: so a log can rebuild a project from nothing.
- **No two hosted views share a name**, since a line addresses one by it; nor two views' folders, since the build tree mirrors each view by its folder's name.
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

**`node new draw <name>` and `node new dispatch <name>`** write a node's first files into its folder in the current project, from the template in `src/recipes/parts/library/template/`, and add the node: a draw of `vertex_count = 3`, or a dispatch of `invocations = 64`. The class takes the name in CamelCase, and a name may hold the nodes it is inside: `node new draw ui.spinner` writes `ui/spinner/`. The files build at once and do nothing yet; every line the engine offers waits in them, commented. The template's own files end in `.in`, so the library never builds them.

```text
src/examples/zz-demo> node new draw spinner
wrote Spinner.cpp, Spinner.glsl, Spinner.vert, Spinner.frag in …/zz-demo/spinner, and added node spinner
src/examples/zz-demo> node new dispatch fill
wrote Fill.cpp, Fill.comp in …/zz-demo/fill, and added node fill
```

**Decision 1 of the 2026-10-03 handoff: the kind is a word of the command.** Built as two commands, the kind is part of the command, so the CLI knows it:

```text
# Built: two commands, `node new draw <name>` and `node new dispatch <name>`
> complete node new d
dispatch                      # the CLI offers both kinds
draw
> help                        # and help explains each on its own line
node new draw <name>          adds a draw node, and writes its first files …
node new dispatch <name>      adds a dispatch node, and writes its first files …

# One command, `node new <name> <value>`, would take any word:
> complete node new spinner d
                              # nothing: <value> is any word, so the CLI cannot offer draw or dispatch
> node new spinner drow
{!!!} …                       # a typo is found only when the command runs
```

The engine's placeholder kinds are a closed list (`name`, `node`, `port`, `file`, …) that completion reads; a `kind` placeholder would add a library-only word to the engine (V05). As two commands, nothing in the engine changes.

**`recipe drop <recipe> <name>`** and **`recipe sync <node>`** copy a library recipe into the project and bring the copy up to date (section 11). The library's recipes are its own, so neither works on a view of the library.

**`view new <folder>` and `view load <folder>`** host a project in a folder, named for the folder, and make it the current one. `view new` also saves its empty `view.vlp`, and refuses a folder that holds one; `view load` refuses a folder that holds none. In the log, both are a `child add` of the CLI's view, and `view new` a `view save` too.

- **Where:** `src/recipes/parts/library/Library.cpp`.

**Text on screen.** Four parts draw what other parts write as Rects and Labels, and `list` lays out Items as they do:

| Part | Reads | Gives |
| --- | --- | --- |
| `palette` | `theme.ini` in its folder, again once it changes | `palette`: a color a role |
| `font` | the font file its `face` param names in its folder, at its `height` param in pixels | `font`, its cell and atlas layout; `atlas`, its glyphs; `metrics`, the same Font for C++ that lays text out |
| `rects` | `rects`, `palette` | a quad a Rect, in its role's color |
| `glyphs` | `labels`, `characters`, `font`, `atlas`, `palette` | a quad a character |
| `list` | `items`, `place` (a Rect) and `font` (the font's `metrics`), from C++; its param `axis`, `y` stacked or `x` side by side | a column of rows in the place over a framed fill, or a row of tabs over a ground: `rects`, `labels`, `characters`; and `shown`, where each Item shows, for `hit` |

A part that shows text writes them as a node's C++ fills any buffer for another node's shader, with the types in `contracts/`:

```cpp
#include "contracts/Label.h"
#include "contracts/Palette.h"
#include "contracts/Rect.h"

_rects = node.upload<VP_VIEW::Rect>("rects", 16); // in bind: room for 16
frame.write(_rects, 1)[0] = {.offset = {24, 24}, .extent = {592, 88},
                             .role = VP_VIEW::role("panel")}; // a name no role has does not compile
```

- **A Label** shows `count` characters of the one list of characters, from `first`, one cell a character from its `offset`; a character past its `extent` is not drawn. Each character names its Label, and `glyphs` draws as many as the list's used length.
- **A theme** gives each role red, green and blue from 0 to 1, linear, with alpha after when it is not opaque. A key that is no role, a role left out or a color that is no such numbers stops the palette, naming the line (`theme.ini:5: pannel is no role; the roles are background, panel, border, text, accent, error, warning`).
- **The font** is the file its `face` param names, printable ASCII baked a cell a glyph; the library's is Roboto Mono. Another face is a file put in the font node's folder and named, as `param set font face RobotoMono-Bold.ttf` does, and a font node bakes again after the edit. A face named with a folder is refused, so the font moves with its view (V03), and so is a font that is not monospace. Two faces or heights at once are two font nodes.
- **Draws stack in graph order**, so `rects` listed before `glyphs` puts text over its panel. The fail-loud test's `text` case wires all four to the fixture's `sign`.

**Layout and the pointer.** Four parts place what the drawing parts draw, and find what a press is on:

| Part | Reads | Gives |
| --- | --- | --- |
| `viewport` | the window's size | `area`: the whole window as a Rect; empty without one |
| `split` | `area`, a Rect; its params `axis`, `x` side by side or `y` stacked, and `ratio`, the first's share; the pointer | `first` and `second`, the two Rects; `rects`, the ground under both and the seam between them; once a dragged seam is let go, `param set <its name> ratio <share>` |
| `bar` | `area`, a Rect, and `font`, the font's `metrics`; its params `edge`, `top`, `bottom`, `left` or `right`, and `size`, in text cells across | `strip`, the bar: `size` rows or columns of text, and 4 pixels either side; `rest`, the room left |
| `hit` | `items`, and `rects`, where each shows, as a list's `shown`; the pointer | on a press, the command of the Item under the pointer |

- **A dock** is a `split` and the `rects` node that draws its ground and seam. A dock in a Rect another dock gave lays out more panels, each seam with its own ratio. The `ide` seats its terminal in `second` of a dock whose split has `axis=y` and `ratio=0.65`.
- **A panel** is a `bar`, a `list` with `axis=x`, `rects`, `glyphs` and `hit`. The bar's strip, one row of text at the top of the Rect its host connects to `bar.area`, shows as tabs the Items the content hands `tabs.items` and `hit.items`, and the content fills `bar.rest`. A press on a tab sends its Item's command, which the content set, so a panel registers no command. The `ide` seats its terminal in one.
- **Another shape is another composition**, and no part changes: a panel with no tabs is its content alone; `bar.size=0` leaves the strip no room; a bar at a side, as `edge=left` with `size=30`, seats a panel 30 columns wide, with no seam to drag, beside its `rest`; and a dock in a panel's `rest`, with a panel on each side, gives it two groups of tabs.
- **A panel closes** by its seam: dragged to the edge, or with `param set ide.dock.split ratio 1`, it gets no room and draws nothing, and the seam stays at the edge to drag back.
- **The seam** is 4 pixels, drawn in the theme's border color, and in its accent color under the pointer and while dragged. A drag keeps the ratio to three decimals, as `param set` writes it, so letting go moves nothing. An axis but `x` or `y`, or a ratio outside 0 to 1, stops the split, naming the param.
- **What is drawn is what is pressed**: `hit` tests the Rects the list drew its rows in, last first, so the one drawn on top answers. The command a press sends is the Item's, as the part that wrote the Items set it.

## 15. What a node's C++ can reach

A node's C++ includes `runtime/Operator.h`, and `runtime/View.h` when it reads the graph. The template shows every call, commented; these are the ports besides the pass block of section 4.

**Its own commands.** A node registers commands in `bind`; they last while the node runs, and a node that goes takes them along. The inspect part's `ls`, shortened:

```cpp
void bind(VP::Bind &node) override {
  _ls = node.command("ls", "lists the views the view hosts, its nodes and its connections");
}
void command(VP::Call &call) override {
  if (!call.is(_ls))
    return;
  for (const VP::Node &node : call.view().nodes) // the view the line addresses, as edited
    call.reply(std::format("node {}", node.name)); // the answer
  // call.commands().send("w: param set wave amplitude 0.5"); // a line inside this one:
  //   it joins this command's log group, and a refusal fails this command too
}
```

A command name is one command: a second node registering it is refused, as the drop in section 11 shows.

**Its own name.** `node.name()` in `bind` is the node's name as a line names it, so a node can send a command about itself. The split part sends where its seam was dragged to this way, so the log keeps it and a replay puts the seam back:

```cpp
_name = node.name();                                                    // in bind
frame.commands().send(std::format("param set {} ratio {:.3f}", _name, ratio)); // in cook
```

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

**The log**: what it printed since the frame before, a line each as the console shows it after the time, with the level it was written at, so a node shows the log as the terminal component does. A line a node sent and a command refused comes back this way, a frame later:

```cpp
for (const VP::Logged &line : frame.commands().log())  // {!!!} node x: …, {nod} …
  show(line.text, line.level == VP::Level::error);     // the node's own way of showing it
```

**Input**: keys, text, the pointer, its buttons, the wheel and focus, as the window got them since the frame before. Every node reads the same events, in order:

```cpp
void cook(VP::Cook &frame) override {
  for (const VP::Event &event : frame.input().events())
    if (event.kind == VP::Event::Kind::key && event.down && event.name == "enter")
      submit();                                         // a key by the character it prints, or its name
  const glm::vec2 at = frame.input().pointer();        // where the pointer is, as the cursor says
}
```

Without a window, the `input` command makes the same events, for the next frame, so a headless test types and points as a person does (V07):

```text
input key down a
input text hello world
input pointer 12.5 40
input button down left
input wheel 0 -1
input focus off
```

- **Neither goes through the log.** The log keeps what input caused, an edit or a save, so a replay rebuilds the graph but not the hands that drove it.
- **Key names**: the character a key prints in the keyboard's layout, or `space`, `enter`, `escape`, `tab`, `backspace`, `insert`, `delete`, `left`, `right`, `up`, `down`, `page_up`, `page_down`, `home`, `end`, `shift`, `control`, `alt`, `super`, `f1` to `f12`. Buttons are `left`, `right` and `middle`. A held key repeats as more `down` events.
- **The pointer** counts pixels from the window's top left, so it matches the frame block's `cursor`; moves within a frame keep the last.

**Files.** A file a node holds is opened in `bind` and watched; any other goes by absolute path. A sketch, as the palette part would read its theme:

```cpp
void bind(VP::Bind &node) override {
  _theme = node.file("theme.ini");        // in the node's folder: a relative path
  _folder = node.folder();                // that folder, absolute
}
void cook(VP::Cook &frame) override {
  const std::string_view theme = frame.files().text(_theme); // read again once it changed
  // frame.files().save(_theme, "text");                     // through a temp file (RA04)
  // frame.files().list(_folder);                           // names, folders ending in /
  // frame.files().read(path);                              // any file, by absolute path
  // frame.files().remove(path);                            // and its folder once empty
  // frame.files().manifest(path);                          // a manifest's graph, as a view runs it
}
```

**Objects between C++ nodes.** A connection between two C++ nodes carries one C++ object, which the engine owns; the writer gets it to change and each reader gets it `const`, once, in `bind`. The type they share is a contract, a header in the view's `contracts/` whose types live in `namespace VP_VIEW`, one namespace per view (RV05). The fail-loud fixture's pair, shortened:

```cpp
// contracts/Count.h
namespace VP_VIEW {
struct Count {
  std::uint64_t frame = 0;
};
} // namespace VP_VIEW

// give/Give.cpp: the writer, which binds first since the connection orders it first
void bind(VP::Bind &node) override {
  _count = &node.output<VP_VIEW::Count>("count"); // the engine's object, by reference
}
void cook(VP::Cook &frame) override {
  _count->frame = frame.index();                   // plain C++
}

// take/Take.cpp: a reader, which cooks after the writer each frame
void bind(VP::Bind &node) override {
  _count = &node.input<VP_VIEW::Count>("count");  // const VP_VIEW::Count *
}
```

```ini
[connection "count"]
from = give.count
to   = take.count
```

- **Checked at load:** a reader of another type is left out, naming both types (`input count is a std::vector<int, …> of 24 bytes, but give writes a vp_mistakes::Count of 8 bytes`), and so is a writer whose type sits in an unnamed namespace, which no reader could name.
- **Kept** across rebuilds, contents included, while the writer's module stays; a swap of that module empties it (`{mod} mistakes/give swapped; connection count starts empty`), since its destructor is that module's code, and the writer makes it again as it binds.
- **A reference lasts until the next `bind`**: every rebuild binds every node again, so none is kept anywhere else.

**The frame index and the log.** `frame.index()` is the frame, as the frame block's `index` is; `frame.log(VP::Level::info, "text")` logs at the node's level (V09).

**The engine by hand.** A node may include any engine header and reach the engine through `node.engine()`, within the [runtime boundary](plans/native-cpp.md#the-runtime-boundary): what it makes, it destroys, and nothing it borrows outlives the next `bind`. The fail-loud fixture's `scratch`, shortened:

```cpp
#include "baseclasses/Engine.h"
#include "baseclasses/Resources.h"

std::optional<VP::Buffer> _buffer; // the operator's own, so it goes before its module

void bind(VP::Bind &node) override {
  _buffer.emplace(node.engine().resources().buffer(
      4096, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT, VP::Memory::upload));
}
```

- **The executable exports its symbols**, so a dev build's module calls the one engine there; a release build links the modules in. On Windows nothing links a dev module against the executable yet, so there a module that calls the engine builds only in a release build.
- **What a node's C++ cannot reach:** another node's code (RV00), since nodes share headers in `contracts/` and never a `.cpp`, and the OS, whose calls live in the platform files (RA01).
- **Where:** `src/runtime/Operator.h`; the template in `src/recipes/parts/library/template/`.

## 16. When something is wrong

Every mistake surfaces at load or at its line, naming its cause (A02):

| Mistake | What you see | What runs |
| --- | --- | --- |
| an unknown word or section, a word given twice | `view.vlp:11: unknown word invocatons in a node; its words are …` | nothing: the load stops |
| a node inside no node | `node a.b is inside a, which is no node` | nothing: the load stops |
| a view's node that uses a recipe as the library does | `node x uses recipe fill as it is, as only the library's own recipes do (V11); …` | nothing: the load stops |
| C++ and a shader disagree on a name or a type | `node fill: the operator sets amount as a uint, but the shader declares a float` | the rest of the view; that node is left out |
| a C++ struct and a shader's put a member at different offsets | `node shape: shapes: member size is a float at byte 12 in C++, but a float at byte 8 in the shader` | the rest of the view; that node is left out |
| two C++ nodes disagree on what a connection carries | `node take: input count is a std::vector<int, …> of 24 bytes, but give writes a vp_mistakes::Count of 8 bytes` | the rest of the view; that node is left out |
| C++ writes more elements than its buffer holds | `node shape: the operator writes 8 elements of shapes, which holds 4; …; its operator stops` | the rest of the view; that node stops |
| a `Texture` nothing fills, or an image C++ fills that nothing samples | `node picture: picture samples nothing: connect it to an image another node's C++ fills, or fill it from the operator` | the rest of the view; that node is left out |
| C++ uploads pixels that do not make the size it gives | `node picture: the operator uploads 7 pixels to picture, which is 4 by 2; its operator stops` | the rest of the view; that node stops |
| C++ fills an image with another pixel than its format takes, or the format is none there is | `node picture: the operator fills picture with uint8 pixels, but its format R32_SFLOAT takes float` | the rest of the view; that node is left out |
| shaders that make neither a draw nor a dispatch, or the wrong count | `node fill: it runs a .comp, so it counts its invocations, and no vertex_count` | the rest of the view |
| a param nothing reads, a field nothing sets | `node wave: param spare: nothing reads it` | the rest of the view |
| an edit that cannot apply | `refuse.txt:2: node wave is connected through values; disconnect it first` | nothing changes; a script stops there |
| a command that does not exist, or does not fit its usage | `unknown command frobnicate; the commands are …` | nothing changes |
| a sync of a copy the view changed | `node d changed since it was dropped from c, and a sync would lose that; …` | nothing changes |
| a C++ module whose build failed | the compiler's output | the running code stays |
| two nodes registering one command | `node look: command \`ls\` registers twice` | the second node is left out |
| a node whose C++ is not built yet | `zz-demo/look has no C++ built: …/module.so is missing` | until the live build compiles it |
| a folder no node names | `{ ! } …: folder extra/ is no node's, but every folder is a node: …` | everything: it is a warning |
