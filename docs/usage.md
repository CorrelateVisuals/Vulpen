# Using Vulpen

How each part of Vulpen works today, shown the way you use it. Each section has a session, a manifest or a node's code, with comments that say what happens, where it holds and what comes out. Every session and output here ran on 2026-10-03, on the tree after `9656e19` that makes every node a folder (RV08), with paths as the [folder move](plans/terminal-app.md#folders) of 2026-10-16 left them; the log's time column is left out, and long paths are shortened with `…`. Code marked as a sketch shows a call's shape, not a file in the tree. What is planned but not built yet is in the [IDE port](plans/ide-port.md) and the [CLI examples](plans/cli-examples.md); how nodes, folders and recipes came to be this way is in [nodes and folders](plans/nodes-and-folders.md).

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
| view | a project: a folder with a `view.vlp`, a folder for each node, and the views it hosts. It loads, runs and moves as a whole (V03) | `projects/<name>/`, as the example projects `wave` and `triangle` |
| manifest | a view's `view.vlp`: its nodes and their files, its connections, and the views it hosts (V00) | `<view>/view.vlp` |
| node | one `[node]` of a manifest and the folder its name names: a C++ class, shaders, or both, and the nodes inside it (RV08) | `<view>/<node>/` |
| connection | one buffer: one node's port writes it, other nodes' ports read it | the manifest |
| recipe | a node of the library with the nodes inside it, which a drop copies into a view | `src/runtime/recipes/library/<kind>/<name>/` |
| library | the recipes views copy from: parts hold code, and `parts/shared/` the code parts share; components only use other recipes (RV06, RV09) | `src/runtime/recipes/library/{parts,components}/` |
| app | a view beside the library that uses its recipes as they are: the `cli`, and the `ide` (section 12) | `src/runtime/recipes/{cli,ide}/` |
| child view | a view another view's manifest names, which runs on its own schedule; commands reach it by name | `[view "<name>"]` |
| command | a line of text that does one thing; every action is one (V06) | the command port |
| primitive | a command that edits or saves a view; the log keeps exactly these (V08) | the log |

## 2. Running a view

`run.sh` (or `run.ps1` on Windows) builds the debug preset in parallel and runs `out/build/debug/vulpen` with your arguments; `--release` takes the release preset. `--cli` and `--ide` are the scripts' own and stand for an app's `view.vlp` (section 12), and with no view named the scripts run the app the line `default=--ide` at their top names. `--<name>` stands for a project's, `projects/<name>/view.vlp`, which then runs alone (section 12); vulpen's options come first, so a project named like one, as `log`, runs by its path.

```bash
./run.sh projects/wave/view.vlp   # the view to run: one view.vlp, or --wave for a project
         --log info               # how much it says: error, warn (the default), info, debug
         --frames 61              # stop after 61 frames; 0, the default, runs until quit
         --fps 0                  # unpaced; the default is 60 frames a second
#        --first-frame N          # start counting frames at N, so a test reaches years in a moment
#        --source script.txt      # run a file of commands before the first frame
#        --size 1280x720          # with no window: the size of images draws render into
```

What it prints:

```text
{run} vulpen 9656e19                                                # the commit it was built from (RC07)
{run} view wave from …/projects/wave/view.vlp: no node draws into the window, so it runs headless   # V07
{gpu} Vulkan validation is on
{gpu} runs on NVIDIA GeForce RTX 5070 Laptop GPU: discrete, Vulkan 1.4.312
{nod} wave: pipeline from Wave.comp                                # each node's pipeline and operator
{nod} wave: new operator Wave
{nod} probe: pipeline from Probe.comp
{nod} probe: new operator Probe
{mem} wave: values of 4096 bytes on the GPU                        # the connection's buffer
{mem} probe: probe.samples of 32 bytes read back by the CPU
{mod} live code: a save under …/projects/wave swaps in              # debug builds only (section 6)
{out} probe: frame     60: +0.8282516 +0.88405794 +0.9278144 …     # what a node's C++ logs
{run} ran 61 frames in 0.01 s
```

- **Headless or a window.** A view runs headless unless a node draws into the window; `./run.sh --triangle` says `a node draws into the window, so it opens one`. A draw whose output a connection takes on renders into an image instead (section 5), so it needs no window: a run with none renders it at the size `--size` gives, and without one records nothing. The window also opens and closes while running, as draws into it come and go (section 8). It is titled after the view, as `triangle - vulpen`, and the desktop knows it as `vulpen`, Wayland's app id and X11's class, which a dock finds it by; its close button sends `quit` (section 7).
- **Quiet by default.** At `warn` a run that goes as meant prints only the frame around the log; `{!!!}` marks an error and `{ ! }` a warning (C09).
- **Exit code.** 0, or 1 when a node is in error when the run ends, or the view cannot load.
- **The baseclasses alone.** `out/build/debug/baseclasses/examples/triangle/triangle` draws a triangle with the GPU layer and nothing above it: a window, the engine, two shaders and their pipeline, in one `main.cpp` in `src/baseclasses/examples/triangle/`, whose loop waits for each frame before it runs the next, as the runtime's does. The baseclasses build as a library of their own, which vulpen links too, so they never need the runtime (RA00).
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

- **The words.** `[node]` takes `recipe`, `operator`, `file`, `invocations`, `vertex_count`, `instance_count`, `param`, `image` and `log`; `[connection]` takes `from` and `to`, each a `node.port`, and `from` also `<view>:`, the window of a view the manifest hosts; `[view]` takes `file` (section 13). A dispatch counts its `invocations`; a draw counts its `vertex_count`, and `instance_count = 64` draws 64 instances, one when left out (VK04: `vkCmdDraw(vertexCount, instanceCount, …)`). `recipe` names where a dropped node came from (section 11).
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
projects/triangle/
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
| the engine, both | the frame block for shaders (section 5); `frame.index()` and `frame.resolution()`, the window's size, or `--size`'s without one, zero without either, for C++ | `frame.resolution` |

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
  uvec2 resolution; // of the window in pixels, or without one --size's; else zero
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

The upload makes the image the size and format given, and the frame copies the pixels in before its passes. The image keeps them through rebuilds while the node, the port and the format stay, so a node uploads once. They stay with the node that fills them, so a connection removed and made again finds them, until `image clear <port>` drops them, named by the port that fills them or one that samples them. Until a node fills its image, and after a clear, its `Texture` holds 0, which means unbound and samples as nothing, and `frame.empty` says so to the node's C++; so does a `Texture` no connection reaches yet, as the `ide`'s waits for the view it presents. An image nothing samples, a format there is not or one this GPU cannot sample filtered, and pixels that do not make the size given are refused.

**An image a draw renders into.** A draw's fragment shader writes one color, a port by its name. Unconnected, it goes into the window; taken by a connection to `Texture`s, the draw renders into an image instead, for those nodes to sample, so which draw reaches the screen stays a fact of the manifest (A7). The image starts each frame cleared to nothing at all, so what the draw leaves uncovered samples as transparent:

```ini
[node "paint"]                               # Paint.frag: layout(location = 0) out vec4 color;
vertex_count = 3
image        = color=R16G16B16A16_SFLOAT     # its format; R8G8B8A8_UNORM without the word

[connection "painted"]
from = paint.color
to   = image.picture                         # a Texture
```

- **Its size** is the window's, or with no window the size `--size` gives, and the image is made again when the size changes. With no size there is no image: its draw records nothing, and its readers sample nothing, as with a 0 slot.
- **Who reads it**: a draw, or a dispatch, after the draw in graph order, samples what it drew that frame.
- **Refused, naming why**: a reader that is no `Texture`, a fragment shader that writes other than one color, and a format there is not, or one this GPU cannot sample and blend into.

- **Barriers** follow from the qualifiers: a pass waits for what an earlier pass wrote, with no barrier placed by hand.
- **Memory** follows from who uses a buffer: one C++ writes or reads back lives where the CPU maps it; any other stays on the GPU (VK03). A buffer holds one element per invocation of its writer, a dispatch's thread or a draw's vertex, and starts zeroed.
- **Passes run in graph order**: the dispatches and the draws into images as the connections order them, then the draws into the window, in one render pass. Each draw blends premultiplied over what came before: an opaque color covers, and alpha lets what is behind show. A hosted view's passes run before those of the view hosting it, so a host samples what a view it hosts drew that frame (section 13).
- **No instances, no draw.** A draw whose port-counted instances are 0 this frame is not recorded, while its pipeline and buffers stay. So a part with nothing to show writes nothing, and what is hidden costs no GPU work and shows again the next frame it is given something: no room, no work, everything kept.
- **The frame block** is written once a frame, after the window's image is acquired, so a resized window's size shows at once; with no window, its resolution is `--size`'s. Its layout comes from reflection, and a shader whose push constant is anything else is refused.
- **Shared GLSL** (GLSL01): `sample_nearest` samples the texel nearest `uv`, for an image drawn a texel a pixel, as a glyph is; `quad_corner(gl_VertexIndex)` gives the corners of a quad's two triangles, so a draw of `vertex_count = 6` places a rectangle an instance; `pixel_clip` puts a point in pixels from the window's top left into clip space.
- **Where:** `src/baseclasses/GpuLayout.glsl`; `src/baseclasses/Shader.cpp` reflects; `src/baseclasses/Pipelines.cpp` owns the frame block, set 0 and the render passes of images draws render into; `src/baseclasses/Engine.cpp` records the frame, the copies into images first; `src/runtime/Schedule.cpp` makes each image a draw renders into, at the frame's size.

## 6. Live code

In a debug build, saving any file of a running view swaps the change in, and the view keeps running.

```text
$ ./run.sh --wave --log info
{mod} live code: a save under …/projects/wave swaps in
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
- **A name that starts with `.`** is no node's: the loader, the build and the scan all skip it, so what a part keeps beside a view, as the graph its `.vulpen/graph.ini` (section 14), starts no build.
- **What stays:** pipelines of unchanged SPIR-V, buffers of unchanged shape with their contents, operators whose folder's module stayed, and the edits typed since the manifest was read.
- **What resets:** a swapped module's operators start over, so state that must survive belongs in params or GPU buffers.
- **A failed build swaps nothing**, and its compiler output is logged; a manifest that does not load keeps the running graph.
- **Where:** debug preset only; `src/runtime/Runtime.cpp` (`Live`, `swap`); the plan is [live code](plans/live-code.md).

## 7. Commands

Every action is a command: a line of text through one port (V06). A command registers with its usage and help, or not at all (RV04). The CLI app lists them with `help`, as a tree of the words they start with: a word more than one command starts with shows once, with their usages under it, indented, and so again for a second word, as `input key`; a command named as its group, as `quit` beside `quit force`, is the group's row; groups come in the order of their words, and what each command does reads down one column:

```text
clear                            clears the terminal
complete <value>...              lists the words that may come next, the last word given being the start of one, …
connect <name> <port> <port>...  joins the port that writes a buffer to the ports that read it
disconnect <connection>          removes a connection
help                             lists every command, under the words it shares, and what each does
image clear <port>               drops the pixels of an image a node's C++ fills, named by that port or one that …
info [<node>]                    shows a node's words, or a hosted view's, and the connections it writes and …
input
  button
    down <value> [<value>]       presses a pointer button: left, right or middle; a count after it, as 2, makes …
    up <value>                   lets a pointer button go
  focus
    off                          takes focus from the window
    on                           gives the window focus
  key
    down <value>                 presses a key, named by the character it prints, or as space, enter or f1, for …
    up <value>                   lets a key go
  pointer <value> <value>        moves the pointer to x and y, in pixels from the window's top left
  text <value>...                types the words, joined by single blanks
  wheel <value> <value>          turns the wheel by x and y
log save <file>                  writes the session's edits to a file, which source replays
ls                               lists the view as folders: each node with its files, and each view it hosts
node
  add <name> [<word=value>...]   adds a node, given the manifest's node words; its files are what its folder holds
  new
    dispatch <name>              adds a dispatch node, and writes its first files from the template into its …
    draw <name>                  adds a draw node, and writes its first files from the template into its folder: …
  remove <node>                  removes a node with the nodes inside it, the connections they write, and them …
  set <node> <word=value>...     gives a node the words named, clearing those given empty; param and image words …
open [<file>]                    opens a file in the editor in a window while one runs, else a file or a folder …
param
  set <node> <key> <value>       sets a param of a node
  unset <node> <key>             removes a param of a node
quit                             ends the run before its next frame, unless a node holds work the end would lose, …
  force                          ends the run before its next frame, even while a node holds work the end would lose
recipe
  drop <recipe> <name>           copies a library recipe into the view as a node of that name, with the nodes …
  list                           lists the library's recipes, by kind
  sync <node>                    brings a dropped recipe up to the library's, while what it copied is unchanged; …
source <file>                    runs a file's commands, one a line, and stops at the first that fails
view
  add <name> <file>              hosts the view a view.vlp holds, or an empty one that view save writes; a line …
  close <name>                   stops hosting a view, letting go of the connections that take its window; its …
  list                           lists the views this view hosts, with their files, in the order hosted
  load <view>                    hosts the view of a project by name, or of a folder by path
  new <view>                     hosts a new, empty view, a project's by name or a folder's by path, and saves …
  rename <name>                  saves the view, renames its folder where it is, and hosts it under the new name
  save                           writes the view over its manifest, keeping the comments in it
    as <name>                    copies the view, its edits too, into a new project and works on the copy; the …
  switch [<name>]                makes a hosted view the current one, where the lines typed go and which every …
```

- **Who registers them.** The engine registers the edits, the log, save, `view list`, `view switch`, `image clear`, `input` and `quit`; every other command is a node's: `help`, `complete` and `clear` are the command-line part's, `ls` and `info` the inspect part's, `recipe …` and `node new …` the library part's, `view new`, `view load`, `view save as` and `view rename` the projects part's, `open` the editor part's, `focus` the keys part's, `write`, `close`, `close force` and `find` the text part's, `mode edit`, `mode perform` and `mode toggle` the modes part's, `zoom in`, `zoom out` and `zoom reset` the font part's, and `select` and `selected` the graph part's; the `ide` has these last five parts and the CLI has not (V05).
- **A usage is its completion.** Its placeholders say what can come there, so `complete` knows. The last may end in `...`, one argument or more, and stand in brackets, which a line may leave out: `node add ui` adds a group. Such a placeholder completes each word after it too, as `<node>` does for every node `[<node>...]` takes.

  ```text
  > complete node new d                       # the word after `node new` that starts with d
  dispatch
  draw
  > complete param set s                      # a <node> comes next: the view's nodes starting with s
  spinner
  > complete view load projects/              # a <file>: what the folder holds, from where vulpen started
  projects/triangle/
  projects/wave/
  ```

- **A `<file>` completes from its folder**, which the path begun names, counted as a typed line's path is (section 12). A name with a blank is left out, since it would split into two words, and a hidden one shows once the name begun starts with `.`, as in a shell. A path that names no folder that can be listed, as a file followed by `/`, is refused, naming why.

- **`quit` refuses while a node holds work** the end would lose, as the editor a file's changes, naming the node and what (section 15). The window's close button sends `quit`, so a refusal keeps the window open; `quit force` ends the run anyway.

  ```text
  {!!!} quit would lose what nodes hold, so the run goes on; quit force ends it anyway:
        ide.editor.text: keys/Keys.cpp* has changes: write saves the file shown, and close force drops its changes
  ```

- **A line may name the view it addresses** as `<name>: <command>`, and `:` alone names the one vulpen started with (section 13).
- **A `<file>` argument arrives resolved:** beside the script that runs the line, or from where vulpen started for a typed line (section 12). A `<view>` is a project's name while it holds no `/`, and arrives as typed; with a `/` it is a folder's path, and arrives resolved as a `<file>` does. It completes as a `<file>` does.
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
node remove nothing                                       # refused: there is none
```

```text
{!!!} …/edits.txt:9: no node is named nothing
```

The first eight lines applied, and the saved view now holds `param = every=30`, the probe's files listed as they were; line 9 stopped the script, and the run with it.

- **What happens.** An edit that breaks the view's shape is refused at once, naming why, and changes nothing: a name used twice, a port in two connections, a connection that would close a cycle, a node inside no node, a file named by a command. A node whose shaders or C++ disagree is left out at the rebuild, naming its line; the rest runs.
- **`node remove` takes a node whole**: the nodes inside it, the connections it or they write, and them from the connections they read, which stay with the readers that stay. It is one line, which a replay expands alike (V08), so a dropped recipe goes as it came.
- **Files stay.** No edit writes or deletes a file: `node remove` leaves the folder, and `node add` takes the files its folder holds.
- **The window follows.** An edit that adds the first draw into the window opens it before the rebuild, and one that removes the last closes it; connecting a draw's output takes its draw out of the window, and connecting a hosted view's window takes its draws out of it:

  ```text
  {run} a node draws into the window, so it opens
  {run} no node draws into the window, so it closes
  ```

- **Errors name the line that last changed the node**, the manifest's or the script's, as `param set wave spare 1` does:

  ```text
  {!!!} spare.txt:1 node wave: param spare: nothing reads it
  ```

- **Other edits:** `param unset <node> <key>` removes a param, and `view add` and `view close` change the views a view hosts (section 13).

- **Where:** `src/runtime/Edits.cpp`; `src/runtime/Views.cpp` keeps the edited view until the rebuild.

## 9. Scripts and the log

A script is a file of commands, one a line. The session's log is the primitives it ran, so replaying the log rebuilds the graph and the files saved from it (V08).

```bash
./run.sh --wave --source edits.txt   # before the first frame
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
  view add zz-demo projects/zz-demo/view.vlp
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
projects/zz-demo> recipe drop inspect look
copied inspect into …/projects/zz-demo/look, as node look of recipe inspect@abb3239c
projects/zz-demo> info look
recipe = inspect@abb3239c          # where it came from, and a fingerprint of what was copied
operator = Inspect
file = Inspect.cpp                 # copied into look/, which the live build compiles
```

- **What a drop copies:** the recipe's files into the node's folder, the recipes it uses into the folders of the nodes inside it, the contracts their files include into the view's `contracts/` (RV05), and the shared code they include into the view's `shared/`, a header with its `.cpp` (RV09). A contract or shared file the view has already stays as it is.
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

In the library, and in the apps beside it, a recipe uses others as they are, which keeps one copy of each part's code (V11). The CLI app is four parts used that way:

```ini
# src/runtime/recipes/cli/view.vlp
[node "cli"]                 # the app's own node: cli/

[node "cli.command-line"]
recipe = command-line        # no fingerprint: the library's own part, unfolded at load
```

- **A recipe is one node named like its folder**, and the nodes inside it; a node that uses it becomes that node, under its own name, and takes `param` and `log` only. Uses never form a cycle (RV06).
- **Only the library and its apps use recipes so.** A view holds its own copies (V03), so a view's node that names a recipe without a fingerprint is refused, naming the drop that copies it.
- **Where:** `src/runtime/recipes/library/parts/library/Library.cpp` (drop and sync); `src/runtime/Manifest.cpp` (`Manifest::flatten` unfolds a recipe the library uses).

## 12. The CLI

The `cli` app is four library parts used together: `command-line` (a line in, an answer out), `inspect` (`ls`, `info`), `library` (recipes and nodes) and `projects` (views of their own). It runs headless on the terminal.

```text
$ ./run.sh --cli
> view new zz-demo                       # hosts an empty project in projects/zz-demo/, and saves its view.vlp
projects/zz-demo> node new draw spinner
wrote Spinner.cpp, Spinner.glsl, Spinner.vert, Spinner.frag in …/projects/zz-demo/spinner, and added node spinner
projects/zz-demo> ls                     # this line went to zz-demo: the prompt says where you are
spinner/                                 # a node is a folder, its files under it
  Spinner.cpp
  Spinner.frag
  Spinner.glsl
  Spinner.vert
projects/zz-demo> : ls                   # `:` sends a line to the CLI itself
zz-demo/         view, from …/projects/zz-demo/view.vlp
cli/
  command-line/  recipe command-line     # inside cli, using a library recipe as it is
  inspect/       recipe inspect
  library/       recipe library
  projects/      recipe projects
projects/zz-demo> view save
projects/zz-demo> view close zz-demo
> ls                                     # no project now: lines go to the CLI itself
cli/
…
>                                        # ctrl-D ends the input, and the run
```

- **The prompt** shows the folder of the project your lines go to, from where you started vulpen, as a shell shows its folder; `>` alone means the CLI itself. It shows only when you type at a terminal, so a piped script's output stays clean.
- **A run in the background** (`./run.sh … &`) reads nothing from the terminal until it is brought to the foreground, since a read would stop it; what is typed meanwhile is the shell's.
- **Keys edit the line**, as in the `ide`'s terminal: Tab completes, up and down recall the lines sent, left, right, home and end move the caret, and backspace and delete erase. control+c ends the run, as its signal did, unless a node holds work, as `quit` does, and so does control+d on an empty line, as the end of input does. The terminal takes each key as it comes, with no echo, from the first frame until the run ends, which gives it back as it found it; a run killed outright (`kill -9`) cannot, and `reset` then does. Each line the log prints takes the line's place, and the line comes back the frame after. A script piped in reads lines as before, and Windows' console types nothing yet.
- **`ls` shows the view as folders**, as its nodes are (RV08): each node a folder under the node it is inside, its files under it, and beside it the recipe it uses, or was dropped from as `recipe probe@3f2a9c1e`; each view it hosts is a folder too, beside it the file it is hosted from. A node's connections are `info <node>`'s.
- **Lines go to the current view**, which the command port keeps for every face: `view switch <name>` moves it to a hosted view, a view a project hosts too, and `view switch` alone to the CLI itself; `view new`, `view load`, `view save as` and `view rename` move it to the project they host. Once the current view is no longer hosted, as after `view close` of it or of the project holding it, it is the nearest view that held it, or the CLI. The CLI sends `zz-demo: ls` for your `ls`, so every log line names its view; a line from a script, or one a node sends, that names no view still goes to the CLI itself, so a log replays the same from any current view (V08). `view switch` changes no view, so the log keeps none of it. The log keeps to the current view too: another project's node lines stay off the console, its errors and warnings excepted.
- **`name: …`** sends one line to another hosted project, and **`: …`** to the CLI itself.
- **A new node builds at once.** In a debug build, the live build compiles `spinner/` a moment after `node new` wrote it, and swaps it in; until then the node is left out, naming the module it waits for (section 16).

**`open [<file>]`** starts the person's editor on a file or a folder, and with none on the folder of the project lines go to, which VS Code opens as a workspace. The `editor` part owns it: its `program` param names the editor, `code` unless set otherwise, as `: param set cli.editor program vim` for a session or that line in the CLI's manifest; a program the PATH lacks is refused, naming it. Its `window` param names an editor in a window as `<view>:<node>`, which takes each file instead while that view runs: the CLI's names the `ide`'s, `ide:ide.editor.text`, for when the CLI hosts the `ide`, and the `ide`'s own `opener` node names its editor, `:ide.editor.text`. The program starts through the shell, in the background, so nothing waits for it; the graph's press on a file and the editor's tabs send `open` too.

**Projects** are views of their own (V03) in `projects/` at the top of the repository, which git ignores but for the example projects, so each person's stay theirs, and a project can be a git repository of its own. The `projects` part keeps them:

- **A name or a path.** `view new <view>` and `view load <view>` take a project's name, a word without a `/`, which means `projects/<name>/`, or a folder's path, from where vulpen started. A name is letters, digits, `_` and `-`, so it also addresses the view, as `zz-demo: ls`, and runs it, as `./run.sh --zz-demo`. `view new` saves the new view's empty `view.vlp`, and refuses a folder that holds one; `view load` refuses a folder that holds none. In the log, both are a `view add` of the CLI's view, and `view new` a `view save` too.
- **`view save as <name>`** copies the view's folder into `projects/<name>/`, whole or not at all, its look in `.vulpen/` too, and writes the manifest there as the edits left it; then it closes the view and hosts the copy, which lines go to from then on. So the view's own files stay as last saved, and a view from elsewhere comes into `projects/`. It refuses a view that hosts views from its folder, since their copies would share their names.
- **`view rename <name>`** saves the view, renames its folder where it is, and hosts it under the new name. Nothing inside a project names it: the build names its contracts and modules from the folder.
- **Both work on a view the CLI hosts**, not on the CLI itself, nor on a view a project hosts, which goes with that project. A name a hosted view has already, or a folder in its place, is refused.
- **`./run.sh --<name>`** runs a project alone, as the view vulpen starts with: no command line, its window or none.
- **The build compiles every project**, and a project made while vulpen runs at the next live build. It holds one view of each name, so a project named `library`, as the recipes build, fails the build, naming both folders: `view rename` it.
- **The gates read every project** as they read `src/`, the include map's node rule and the code rules among them, since a release build links its code into vulpen.
- **The example projects**, `wave`, which computes headless, and `triangle`, which draws in a window, are everyone's: git tracks them, `./run.sh --wave` and `./run.sh --triangle` run them, and the tests use them, so trying things in them moves the tests until `git checkout projects/wave` puts them back. A project worth sharing becomes another, with a line for it in `.gitignore`, or recipes in the library.
- **Elsewhere:** the folder is the `projects` part's `folder` param, from the part's own folder (RP02); working elsewhere is `param = folder=…` on `cli.projects` in the CLI's manifest, and the build compiles projects only in `projects/`.
- **Where:** `src/runtime/recipes/library/parts/projects/Projects.cpp`; the file port's `copy`, `rename` and `save` of a view at another file in `src/runtime/Ports.cpp`.

**Decision 2 of the 2026-10-03 handoff: where a path you type points.** A relative path in a typed line counts from where you started vulpen, which is also what the prompt's path counts from:

```text
$ cd ~/Documents/GitHub/Vulpen && ./run.sh --cli
> view new zz-demo
projects/zz-demo> log save notes.log
# writes ~/Documents/GitHub/Vulpen/notes.log, beside where you started,
# not ~/Documents/GitHub/Vulpen/projects/zz-demo/notes.log
projects/zz-demo> view load projects/wave
# hosts ~/Documents/GitHub/Vulpen/projects/wave: the same text works from any project
```

The other way is a shell's `cd`: paths count from the current project.

```text
# If paths counted from the project, as after `cd projects/zz-demo`:
projects/zz-demo> log save notes.log         # would write projects/zz-demo/notes.log
projects/zz-demo> view load ../wave          # would be needed for projects/wave;
projects/zz-demo> view load projects/wave    # this would mean projects/zz-demo/projects/wave
```

The first keeps every typed path meaning one thing wherever you are, and matches the prompt; the second keeps a project's files next to it. Lines from a script always count from the script's folder either way.

**The `ide`.** The `ide` app is the CLI's parts in a window, with four panels in a dock: the theme, the font, the keys, the modes part, `inspect`, `library` and `projects` as in the CLI, and in the dock the editor, the graph, the Perform panel and the terminal. Each panel is a `panel` around a content, and each seam is a `split`'s:

```text
+-------------------------+--------------------------+
| triangle/Triangle.cpp   | graph - triangle         |
|  the editor (section 14)|  a box a node (14)       |
|                         +--------------------------+
|                         | triangle                 |
|                         |  the Perform panel (14)  |
+-------------------------+--------------------------+
| terminal - triangle                                |
|  the log, and the line typed                       |
+----------------------------------------------------+
```

`control` with `=` or `+` makes all of it larger, `control` with `-` smaller, and `control+0` puts it back (section 14). A session, from the terminal: `view load triangle` hosts the triangle, the graph shows its node, and the Perform panel shows its window under a tab of its own; a press on its box opens `triangle/Triangle.cpp` in the editor, and after an edit `control+s` writes it, which the live build swaps in, so the Perform panel shows the change a moment later (section 6). `f1` gives the triangle the whole window, and `f1` or `escape` gives it back. `escape` in edit mode lets go of what is selected, then of the focus, and then sends `quit`, which, as the window's close button, ends the run unless a file has changes (section 7).

**The terminal in a window.** The terminal is the same `command-line` part with `param = on=window`: it reads the keyboard instead of standard input, and shows above the line typed what lines answer and the log, the lines typed in the accent color, what they answer and what nodes write (`{out}`) in the text color, the rest of the log muted, errors in red and warnings in amber, so a refused line names its cause there, a frame later. With views running side by side, the log keeps to the view the terminal's lines go to: a line about a node of another view stays out as it comes, on the console as in these rows, so one project's output never drowns another's. The lines about that view, the views it hosts and the `ide`'s own parts show, and so do every view's errors and warnings, which never hide (A02). The terminal vulpen was started from takes lines too, as the CLI's does: they go where the window's lines go, and what they answer prints there, while the log shows in both. The end of its input, as a launcher that gives none or a script piped in has, ends no run; the window does.

```text
$ ./run.sh --ide      # the default, so ./run.sh alone does the same
```

- **Keys:** typing inserts at the caret, which Left, Right, Home and End move; Backspace and Delete erase, Enter sends the line, and Up and Down step through the lines sent. Escape closes the completions, with none open lets go of the selection, and with none of the focus.
- **Selecting:** a drag over the rows above the line selects their text, a double click a word, and a third press the row, and `control+c` copies it; `control+v` types what the clipboard holds at the caret, as a shell takes a paste: each line break sends the line typed so far. A selection stays on its text as new rows push it up, and goes with its rows once the scrollback drops them, or `clear` does.
- **Tab** completes as a shell does: the word at the caret takes what every candidate shares, a word only one candidate fits is finished with a blank, and several are printed among the rows, as a shell lists them: the line as typed, then the candidates in columns, down then across, as many columns as the rows hold. A placeholder such as `<name>` is listed but never typed in. Tab sends `complete`, which takes `""` for a word not begun. A folder is finished with no blank, so Tab again lists what it holds, each name from after the path's last `/`: `view load proj` and Tab gives `projects/`, and Tab again lists `triangle/` and `wave/`. What Tab lists is text, as any row is: a drag selects it, and a press picks nothing.
- **The seam** above the terminal lights up under the pointer, and a drag moves it, the terminal following. Letting go sends one `param set ide.dock.split ratio 0.416`, so the log keeps one line a drag, and a replay or a save puts the seam back.
- **The tab** over the terminal names it and the current view: `terminal`, and `terminal - demo` after `view new demo`. It is the command-line's, from its `title` param, and has no command, so a press on it sends nothing, and no close mark, as the graph's has none: a view closes from its Perform tab, so no two marks close the same view.
- **Scrolling:** the wheel scrolls the rows above the line, 3 rows a step, keeping a step's fraction, so a touchpad's small turns add up; a drag on the bar at the right edge does too, and a press beside its thumb brings the thumb there first. While the newest rows show, new ones scroll into view; scrolled back, the rows shown stay, until a line sent shows the newest again.
- **Across:** a row wider than the terminal is cut at its right edge, and the wheel turned sideways, as a touchpad turns it, or with `shift` held, scrolls the rows across, 3 columns a step, as does a drag on the bar along their bottom edge, just above the rule, which shows while a column is out of view. The line typed does not scroll with them: it scrolls on its own, to keep the caret in view.
- **`clear`** empties the scrollback, which keeps the last 1,000 rows.

- **Where:** `src/runtime/recipes/cli/view.vlp`, `src/runtime/recipes/ide/view.vlp`, `src/runtime/recipes/library/components/terminal/view.vlp`, `src/runtime/recipes/library/components/dock/view.vlp`, `src/runtime/recipes/library/components/panel/view.vlp`; `src/runtime/recipes/library/parts/command-line/CommandLine.cpp`.

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
# lines.txt: w: param set wave amplitude 0.5, then view list
w …/wave/view.vlp                          # view list: the views this view hosts, in the order hosted
{run} child w: hosted from …/wave/view.vlp
{out} probe: frame     60: +0.4141257 +0.442029 …   # half the wave of section 2
```

- **Hosting follows the manifest.** Loading a view hosts the views it names; `view add` and `view close` are edits of the view a line addresses, which `view save` writes, and the log keeps. A child's own `[view]` sections host views in turn, and removing a child takes them along.
- **`view close <name>` reaches the view that hosts it** from wherever the line goes, so `view close zz-demo` typed in zz-demo closes it. It lets go first of every connection that takes the view's window (below), in the same edit, so the log keeps one line, and nothing takes the view's place: the `ide`'s Perform panel shows nothing until another view is presented. Its files stay.
- **Each frame** cooks the host, then the hosted views in the order hosted, and runs all their passes. The window opens while a node of any view draws.
- **A view whose `view.vlp` does not exist yet** starts empty, and its `view save` writes it, folder included: so a log can rebuild a project from nothing.
- **No two hosted views share a name**, since a line addresses one by it; nor two views' folders, since the build tree mirrors each view by its folder's name.
- **The live scan** watches every hosted view's folder, so saving a project's file swaps it in.
- **A line a command sends** goes to the same view as the line that ran it, unless it names one; a script's lines each stand alone.
- **Where:** `src/runtime/Views.cpp`.

**A hosted view's window.** A hosted view draws into the window over a host that draws nothing there, as the CLI. A host that draws into the window itself has it to itself (A01), so a view it hosts draws nowhere: it keeps running, its C++ cooking and its dispatches running, and its buffers and images kept, but it records no draw, as the `ide` keeps a project it does not present (`child triangle: draws nowhere, since its host draws there and no connection takes its window`, at `--log info`). A connection from `<view>:` takes what it draws into its window as one image, which the host's `Texture`s sample. The view runs as it would on its own and knows nothing of who takes it (V03):

```ini
# host/view.vlp
[view "demo"]

[node "show"]                # samples the Texture picture, as the image part does
…

[connection "present"]
from = demo:                 # the window of the view demo, as one image
to   = show.picture          # repeat to for more readers
```

- **What it holds**: the view's draws into its window, stacked in graph order, and those of the views it hosts that draw into its window, over them, as in a window of its own. It starts each frame cleared to nothing at all, so what they leave uncovered samples as transparent.
- **Its format** is `R8G8B8A8_SRGB`, as the window takes it, so the view's draws blend and band as they would in its own window. **Its size** is the window's, or `--size`'s with none, as a draw's image is.
- **Order**: the view's passes run first, its draws into the image in one render pass, then the host's, so a host's dispatch or draw samples what the view drew that frame.
- **Connecting or disconnecting** rebuilds the hosted view too, so its draws' pipelines fit what they draw into now; drawing nowhere, its draws have none.
- **Refused**, naming why: `<view>:` naming no view this view hosts (`connection stage is from inner:, but this view hosts no view inner`), a connection to a view's window, and a reader that is no `Texture`. `view close` of a view a connection takes lets go of the connection first.

## 14. The library

The `library` part works on recipe and view folders through the file port. It runs in the CLI.

**`recipe list`** lists the library's recipes as `kind/name`; the apps are beside the library, not in it, so it lists neither:

```text
> recipe list
components/dock
components/find-bar
…
parts/command-line
parts/inspect
…
```

**`node new draw <name>` and `node new dispatch <name>`** write a node's first files into its folder in the current project, from the template in `src/runtime/recipes/library/parts/library/template/`, and add the node: a draw of `vertex_count = 3`, or a dispatch of `invocations = 64`. The class takes the name in CamelCase, and a name may hold the nodes it is inside: `node new draw ui.spinner` writes `ui/spinner/`. The files build at once and do nothing yet; every line the engine offers waits in them, commented. The template's own files end in `.in`, so the library never builds them.

```text
projects/zz-demo> node new draw spinner
wrote Spinner.cpp, Spinner.glsl, Spinner.vert, Spinner.frag in …/zz-demo/spinner, and added node spinner
projects/zz-demo> node new dispatch fill
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

**`recipe drop <recipe> <name>`** and **`recipe sync <node>`** copy a library recipe into the project and bring the copy up to date (section 11). The library's recipes are its own, so neither works on a view of the library, or on an app beside it.

`view new` and `view load` are the projects part's (section 12).

- **Where:** `src/runtime/recipes/library/parts/library/Library.cpp`.

**Text on screen.** Four parts draw what other parts write as Rects and Labels, and `list` lays out Items as they do:

| Part | Reads | Gives |
| --- | --- | --- |
| `palette` | `theme.ini` in its folder, again once it changes | `palette`: a color a role |
| `font` | the font file its `face` param names in its folder, at its `height` param in pixels, times the zoom it keeps | `font`, its cell and atlas layout; `atlas`, its glyphs; `metrics`, the same Font for C++ that lays text out |
| `rects` | `rects`, `palette` | a quad a Rect, in its role's color |
| `glyphs` | `labels`, `characters`, `font`, `atlas`, `palette` | a quad a character |
| `image` | `area`, a Rect, and `picture`, an image: one a draw renders into, the window of a view its view hosts, or one C++ fills | the image in the Rect, letterboxed: as large as fits at its own aspect, centered; nothing while the Rect is empty or nothing reaches `picture` |
| `curves` | `curves`, `area` (a Rect, its first), `palette` | a strip of quads along each Curve, a cubic that leaves its start heading right and reaches its end from the left, 2 pixels wide, in its role's color, cut at the area's edge |
| `list` | `items`, `place` (a Rect) and `font` (the font's `metrics`), from C++; its param `axis`, `y` stacked or `x` side by side | a column of rows in the place over a framed fill, or a row of tabs over a ground, a close mark at the end of each tab whose Item has a `close` command: `rects`, `labels`, `characters`; and `shown`, where each Item shows, and `marks`, where its close mark shows, for `hit` |

A part that shows text writes them as a node's C++ fills any buffer for another node's shader, with the types in `contracts/`:

```cpp
#include "contracts/Label.h"
#include "contracts/Palette.h"
#include "contracts/Rect.h"

_rects = node.upload<VP_VIEW::Rect>("rects", 16); // in bind: room for 16
frame.write(_rects, 1)[0] = {.offset = {24, 24}, .extent = {592, 88},
                             .role = VP_VIEW::role("panel")}; // a name no role has does not compile
```

- **A Label** shows `count` characters of the one list of characters, from `first`, one cell a character from its `offset`, in the room its `extent` gives from there; a character past the room's edge is cut at it. Its `cut` moves the room's top left in from `offset`, so a run that starts above or left of what shows, as a graph's row half out of view, is cut there too; 0, as a Label written without one has, cuts nothing. Each character names its Label, and `glyphs` draws as many as the list's used length.
- **Its `scale`** sizes its cells, the font's times it, as the graph's zoom does; 0, as a Label written without one has, draws at the font's size. `glyphs` samples the atlas smoothly, which at the font's size changes no pixel, since each cell lies on whole pixels.
- **A theme** gives each role red, green and blue from 0 to 1, linear, with alpha after when it is not opaque. A key that is no role, a role left out or a color that is no such numbers stops the palette, naming the line (`theme.ini:5: pannel is no role; the roles are background, panel, border, text, accent, error, warning, muted, keyword, type, string, number, comment, directive`). `muted` is for text that matters less, as the terminal's log lines, and the last six for code, as the editor shows it.
- **The font** is the file its `face` param names, printable ASCII baked a cell a glyph, the range `contracts/Font.h` names, so the parts that take typed text keep to it; the library's is Roboto Mono. Another face is a file put in the font node's folder and named, as `param set font face RobotoMono-Bold.ttf` does, and a font node bakes again after the edit. A face named with a folder is refused, so the font moves with its view (V03), and so is a font that is not monospace.
- **Zoom.** `zoom in` draws the text a step larger, a tenth more, up to 4 times the `height` param; `zoom out` a step smaller, down to half of it; and `zoom reset` at the `height` again. Every part lays text out in cells, so all of them grow with the font, and in the `ide` that is the whole window but its seams. The zoom is a look at the text, not the view, so it stays out of the manifest (V04) and the log (V08), in `.vulpen/font.ini` in the font node's folder, which the font reads again once it changes and a restart reads; the `ide`'s font is the library's part, so its zoom is kept in `src/runtime/recipes/library/parts/font/.vulpen/`. A mistake in the file is named, the font draws at its `height`, and a zoom is refused rather than writing over it (`.vulpen/font.ini:1: zoom = 9 is no zoom = z from 0.5 to 4`).
- **One font a run.** Its zoom commands have one owner (B5), so a second font node is left out, naming them (``command `zoom in` registers twice``), as a second text part is; two faces or heights at once wait until something needs them (C00).
- **Draws stack in graph order**, so `rects` listed before `glyphs` puts text over its panel. The fail-loud test's `text` case wires all four to the fixture's `sign`.

**Layout and the pointer.** Four parts place what the drawing parts draw, and find what a press is on:

| Part | Reads | Gives |
| --- | --- | --- |
| `viewport` | the window's size | `area`: the whole window as a Rect; empty without one |
| `split` | `area`, a Rect; its params `axis`, `x` side by side or `y` stacked, and `ratio`, the first's share; the pointer | `first` and `second`, the two Rects; `rects`, the ground under both and the seam between them; once a dragged seam is let go, `param set <its name> ratio <share>` |
| `bar` | `area`, a Rect, and `font`, the font's `metrics`; its params `edge`, `top`, `bottom`, `left` or `right`, and `size`, in text cells across | `strip`, the bar: `size` rows or columns of text, and 4 pixels either side; `rest`, the room left |
| `hit` | `items`, and `rects`, where each shows, as a list's `shown`, and `marks`, where its close mark shows; the pointer | on a press, the command of the Item under the pointer, or on its close mark its `close` command |

- **A dock** is a `split` and the `rects` node that draws its ground and seam. A dock in a Rect another dock gave lays out more panels, each seam with its own ratio. The `ide` seats its terminal in `second` of a dock whose split has `axis=y` and `ratio=0.65`.
- **A panel** is a `bar`, a `list` with `axis=x`, `rects`, `glyphs` and `hit`. The bar's strip, one row of text at the top of the Rect its host connects to `bar.area`, shows as tabs the Items the content hands `tabs.items` and `hit.items`, and the content fills `bar.rest`. A press on a tab sends its Item's command, which the content set, so a panel registers no command; a tab whose Item has a `close` command shows a close mark, `x`, at its end, whose press sends that instead, so the content still says what closing means. The mark shows only where the tab fits whole. The `ide` seats its terminal in one.
- **Another shape is another composition**, and no part changes: a panel with no tabs is its content alone, which costs nothing; `bar.size=0` leaves the strip no room, so it draws nothing (section 5) and comes back the frame after `size 1`, its parts kept: their buffers and pipelines, and three operators that find nothing to do; a bar at a side, as `edge=left` with `size=30`, seats a panel 30 columns wide, with no seam to drag, beside its `rest`; and a dock in a panel's `rest`, with a panel on each side, gives it two groups of tabs.
- **A panel closes** by its seam: dragged to the edge, or with `param set ide.dock.split ratio 1`, it gets no room and draws nothing, and the seam stays at the edge to drag back.
- **The seam** is 4 pixels, drawn in the theme's border color, and in its accent color under the pointer and while dragged. A drag keeps the ratio to three decimals, as `param set` writes it, so letting go moves nothing. An axis but `x` or `y`, or a ratio outside 0 to 1, stops the split, naming the param.
- **What is drawn is what is pressed**: `hit` tests the Rects the list drew its rows in, last first, so the one drawn on top answers. The command a press sends is the Item's, as the part that wrote the Items set it.

**Keys and the focus.** The `keys` part decides where a key goes, so a key does nothing a typed command cannot (V06):

| Part | Reads | Gives |
| --- | --- | --- |
| `keys` | `keymap.ini` in its folder, again once it changes; the keys and text the window got; its param `focus`, the node in focus at the start | the line each chord sends; `typed`, every other key and text, with the name of the node in focus |

```ini
# keymap.ini: a chord, = and the line it sends, as a line typed at the command port
control+l       = clear
control+s       = write
control+=       = zoom in        # the = key, which types + with shift
shift+control+= = zoom in
control++       = zoom in        # a + key, as the keypad's
control+-       = zoom out
control+0       = zoom reset
f1              = mode toggle
```

- **A chord** holds `shift`, `control`, `alt` or `super`, joined by `+`, then presses a key named as section 15 names keys, `+` and `=` too: the `=` after the chord sends. A modifier or a key that is not there, a chord given twice, or a line no command takes stops the keys part, naming the line (`keymap.ini:1: no command takes clera`), and a fixed file brings it back. A key that types, as `a`, still types when a chord takes it, since the window sends text apart from keys; `control` on a letter, as `control+l`, types nothing on any system.
- **The focus** is a node's name: `focus <node>` moves it, `focus` alone leaves no node in focus, and the keys part's `focus` param says where it starts, as the `ide` starts it in its terminal. A node that does not read `typed` is refused, naming those that do (`ide.dock reads no keys from ide.keys; these do: ide.terminal.command-line`). A press in the area of the terminal, the editor, the graph or the Perform panel gives it the focus, and only the part in focus shows its caret. Like input, a focus change stays out of the log.
- **With no node in focus** the keys go to the empty name, `typed.to("")`, which the `ide`'s modes part reads.
- **Escape** is no chord: the part in focus takes it, letting go of its selection, or of its completions, and with none of the focus. With no node in focus, modes takes it: in perform mode it sends `mode edit`, and in edit mode `quit`. So `escape` steps back one thing at a time: perform mode, the selection, the focus, the run.
- **A part that takes keys** reads `typed`, not the input port, and acts only when the name in it is its own; it still reads the pointer and its buttons from the input port. Given no room it takes none, as it draws none, so a terminal whose panel is closed runs no line typed unseen; the keys go nowhere until a press gives a part that shows the focus.
- **In the order they came**: a focus line takes effect as the keys part next hands keys on, so every part reading them in one frame finds the same node in focus. A key after a press, a chord or `escape` waits for the next frame, since each may move the focus, and so does a chord after keys, so its line runs once the part in focus took them. A script that types, presses and types again in one frame puts each key where a person's would go.
- **Where:** `src/runtime/recipes/library/parts/keys/`.

**The editor.** The `text` part holds the files open and shows one. The `text-area` component is it with `rects` and `glyphs`, and the `ide` seats one in a panel in the dock's top row, a tab a file open. Its `file` param names the file it shows: a change opens it as a tab and shows it, or shows it if it is open. `open` sets it (section 12), so the log keeps each file opened, a replay opens it again, and a save of the `ide` keeps the last one; the file opens as the editor next binds, the frame after, so a script's line after `open` that needs the file, as `find`, waits a frame. Closing the file the param names unsets it, so `open` of it again is a change.

| Command | What it does |
| --- | --- |
| `open [<file>]` | the editor part's: names the file in the editor's `file` param; with none, the `view.vlp` of the view the line goes to. A folder, or a file it cannot read, is refused, naming `: param unset ide.opener window`, which sends each `open` to the person's editor |
| `write` | saves the file shown; `control+s` sends it |
| `close [<file>]` | closes a file the editor holds, or with none the file shown, unless it has changes; the close mark on a file's tab sends it with the file |
| `close force [<file>]` | the same, dropping its changes, as `quit force` ends the run however `quit` was refused |
| `find <value>...` | selects the next place the file shown holds the words, joined by single blanks, after the caret, and from its start past its end |

- **Keys**, while it has the focus: typed text goes in at the caret. `left` and `right` step a character, `up` and `down` a row, `page_up` and `page_down` a page, and `home` and `end` go to the row's start and end, selecting while `shift` is held. `enter` breaks the row, keeping its indent; `tab` types spaces to the next even column; `backspace` and `delete` take out a character, or the selection; `escape` lets go of the selection, and with none of the focus. `control` with `c` copies the selection, with `x` cuts it, and with `v` pastes in its place, each line break of the paste starting a row. A press puts the caret where it was and gives the editor the focus, a drag selects, a double click selects a word, of letters, digits and `_`, of blanks, or of other characters, and a third press the row with its break, and the wheel scrolls, as the terminal's does, until the last row shows alone; so does a drag on the bar at the right edge, which shows while a row is out of view. Turned sideways, or with `shift` held, the wheel scrolls across, until the end of the widest row shows, as does a drag on the bar along the bottom edge, which shows while a column is out of view; as the caret moves, the rows and columns shown follow it.
- **A tab** reads the file's folder and name, as `keys/Keys.cpp`, with `*` while it has changes, and a press on it shows the file, and one on its close mark closes it; the tab shown is in the content's color. `close` on a file with changes is refused, naming `write` and `close force` (`keys/Keys.cpp* has changes: write saves them, and close force drops them`); `quit` is refused too, naming every file with changes (section 7).
- **Colors**: C++ and GLSL files, by their ending (`.cpp`, `.h`, `.glsl`, `.vert`, `.frag`, `.comp`, and the template's `.cpp.in` by its name), show comments, `#` directives, quotes, numbers, keywords and types in the theme's roles for them; a type is one of the languages' own, a name in CamelCase, as types are named here (RC01), one ending in `_t`, or a GLSL vector or matrix, as `uvec2`. A manifest and an `.ini` show their `#` comments, quotes, numbers, and each row's first word, its key or its section, so the manifest's words need no list of their own in the editor. Python (`.py`), CMake (`CMakeLists.txt`, `.cmake`) and shell (`.sh`), which the tooling is written in (RC00), show their `#` comments, quotes, numbers and keywords: Python its built-in types and names in CamelCase as types, a decorator as a directive, and a string in three quotes over every row it spans; CMake a command, a name before its `(`, as a keyword, a name in capitals as a type, and `${…}`, `$ENV{…}` and `$<…>` as directives; shell `$name`, `${…}`, `$1` and the like as directives. Markdown and any other file show in one color. A block comment's rows, and a Python string's, are found again from the file's top after an edit, not each frame.
- **Files** are read once, as they open, so what was typed is never read over; a change on disk to a file open does not show. `write` saves through the file port (RA04), and a file not open is never written.
- **The font draws ASCII**, so a character past it shows as a blank cell a byte, `—` as three. The caret steps over it whole and no edit splits it, so the rest of a file stays as it was.
- **One editor a run**: its commands have one owner (B5), so a second text part is left out, as a second command-line is.
- **Where:** `src/runtime/recipes/library/parts/text/`.

**Modes and the Perform panel.** The `modes` part chooses what reaches the window, and shows a view the `ide` hosts without it drawing over the `ide` (section 13):

| Command | What it does |
| --- | --- |
| `mode edit` | gives the window to the dock; what is presented shows in the Perform panel. `escape` in perform mode sends it |
| `mode perform` | gives the window to what is presented, and the dock no room, so the dock draws and takes nothing; refused while nothing is presented, naming `view switch` |
| `mode toggle` | switches to the other mode; `f1` sends it, so one key goes there and back, while a script names the mode it wants |

```text
> view load triangle              # the triangle, in the Perform panel, letterboxed
projects/triangle> mode perform   # the triangle, over the whole window
```

- **The Perform panel** sits beside the editor in the dock's top row. Its tabs name the views the `ide` hosts, the one presented marked; a press on one sends `view switch` for it, and a press on its close mark sends `view close` for it, so nothing is presented in its place. Its image shows the view's window letterboxed, at the window's size, so a view draws as on its own; in perform mode it matches the view's own window pixel for pixel. A drag moves the window in the panel and the wheel zooms it about the pointer, as in the graph, cut at the panel's edge; the image part does so only with `param = moves=yes`, which the `ide` gives the panel and not the one that fills the window in perform mode. A press in the panel gives it the focus, and `h` then brings the window home, letterboxed as at first; `escape` lets go of the focus. Neither is kept: a restart, and another view presented, show the window letterboxed again.
- **The Perform panel presents the current view** while it is a project, a view the `ide` hosts, each in a tab of its own, so the views loaded before keep theirs; a view a project hosts shows through its project, as the project composes its window (V03). While the current view is the `ide` itself, the panel keeps the one it presents, and when the `ide` starts, it presents the project hosted last. A view not presented draws nowhere and keeps running (section 13), so no project ever draws over the `ide`. `view load` of a view hosted already is refused, naming it and the file it is hosted from (`a view named wave is hosted already, from …/wave/view.vlp`), so a press on its tab shows it.
- **The terminals, the graph and the Perform panel follow one view**, the current one, so a press on a Perform tab, or `view switch b` typed, switches all of them, and the terminal's and the graph's tabs name it (`terminal - b`, `graph - b`). Once the project presented closes, the panel shows nothing.
- **Perform mode needs something presented.** `mode perform`, and `f1` with it, is refused while nothing is (`nothing is presented to perform; view switch <name> shows a view this view hosts`), rather than giving the window to nothing; the current view's project is presented first, as the frame would, so `f1` just after `view load` finds it. Perform mode gives way to edit mode once the view presented goes, as `view close` takes it, and once the log shows an error, as a refused line's or that of `quit` refused while a file has changes, so the terminal shows why: perform mode hides the log.
- **The mode is a param** of the modes node, `param set ide.modes mode perform`, which `mode` sends, so a save keeps it and a replay restores it; each switch rebuilds the `ide`. Perform mode leaves the panels no room, so it takes the focus from the part that had it, and edit mode gives it back.
- **Presenting** makes one connection, `present`, from `<view>:` to the images the modes node's `to` param names, ports joined by commas, from the `ide`'s connections as the lines so far left them, so the next frame shows it; the log keeps it, so a replay presents the same view (V08). `view close` of the view presented lets go of `present` first, so the Perform panel shows nothing, and nothing else is presented in its place.
- **Where:** `src/runtime/recipes/library/parts/modes/`, `src/runtime/recipes/library/parts/image/`.

**The graph.** The `graph` part lays out the graph of the current view, the one the terminals' lines go to, and its tab names it, as `graph - wave`, or `graph` for the `ide` itself, whose own nodes it shows after `view switch`, as the CLI's will once the CLI hosts the `ide`. A second press on a view's box goes into that view, as `view switch` does. The `graph-editor` component is it with `curves`, `rects` and `glyphs`, and the `ide` seats one in a panel over the Perform panel.

```text
> view load wave       # the graph panel shows wave and probe, a curve between

+-----------+     +------------+
| wave      |~~~~~| probe      |    # the node's name, in the accent color
| Wave.comp |     | Probe.comp |    # under it its files, as its folder sorts them
| Wave.cpp  |     | Probe.cpp  |
+-----------+     +------------+
```

```ini
# projects/wave/.vulpen/graph.ini, as the graph writes it once a box is dragged
color dispatch = number # written by a person: a kind of node's boxes in a theme role
place probe = 0 200     # each box, from the graph's top left at zoom 1
place wave = -10 -5
```

- **The layout follows from the graph** until a box is dragged, since the manifest holds no places (V04): a box a node, with its name and a row a file under it, a column past every node it reads from, and down each column in the manifest's order. A node's C++ and GLSL are its files, not nodes of their own, so the triangle is one box of five rows. A connection is a curve from the middle of its writer's name row on the right to its reader's on the left; one that passes a column runs along a lane of its own below that column's boxes, not behind them. A view the view shown hosts is a box of its own, after the nodes, its name over the file it is hosted from, as `view.vlp`, and a connection that takes its window is a curve from it, so the project shows whole (V01). Boxes past 256 are cut.
- **A press on a box selects it**, and draws its border wider, in the accent color; `shift` with a press adds a box to the selection or takes it out. A press on a box selected keeps the selection, so a drag moves every box selected, and letting go where it was leaves only that box selected. A press on the ground lets go of the selection; with `shift` held it starts a select box instead, drawn in the accent color from there to the pointer, and letting go adds every box it touches to the selection; it is cut at the graph's edge, so only what shows of a box can be touched. A press on one of a box's files opens it in the editor, as `open <file>` typed would, on letting go, so a box pressed and moved is dragged instead; its name opens none. A view's box opens the manifest it is hosted from. A press in the graph gives it the focus.
- **The selection is the graph's, through three commands**, so a script or an agent selects as a press does (V06). They act on the view the graph shows, the one lines go to, and refuse any other:

  | Command | What it does |
  | --- | --- |
  | `select [<node>...]` | selects those nodes, or views the view hosts, in place of those selected; with none, none |
  | `selected` | lists the nodes and views selected, a line each |

  ```text
  projects/wave> select probe
  projects/wave> info              # with no node: each selected, under its header
  [node "probe"]
  operator = Probe
  …
  reads values from wave.values
  projects/wave> node remove probe  # values goes with it, which only probe read
  ```

  A view selected shows under its own header, `[view "inner"]`, with the file it is hosted from whole, `file = …/inner/view.vlp`, and each connection that takes its window, `writes its window as stage`.

- **The graph starts at home**, whenever it shows a view, as `view load` and a switch to another view do: unmoved at zoom 1 while every box shows whole there; otherwise the boxes from their top left, as far in as the graph puts its first box, and as near as lets them all show, never nearer than zoom 1, so text keeps its size where it can. Where even a quarter cannot hold them, they show from their top left at a quarter.
- **Keys**, while the graph has the focus: `delete` or `backspace` removes what is selected, a line each, `node remove` for a node, which takes the nodes inside it along, and `view close` for a view, and their boxes leave the `.ini`; `escape` lets go of the selection, and with none of the focus, and `h` brings the graph home.
- **A dragged box stays where it is put.** The first drag places every box where it shows, so moving one moves no other, and a node added later takes its column's place; lanes are left out from then on, as the boxes no longer keep to columns.
- **A drag on the ground moves the graph, and the wheel zooms it**, about the pointer, which stays over the same place of the graph, from a quarter to four times as near, and the text scales with the boxes; the wheel's turn sideways does nothing yet. GLFW gives no pinch event, so a pinch does nothing. The Perform panel moves alike, through the same shared `PanZoom` (RV09).
- **A restart shows the boxes where they were left.** Where the boxes are is a look at the graph, not its plumbing, so it stays out of the manifest (V04) and the log (V08), in `.vulpen/graph.ini` beside the view's manifest, which moves with the project (V03). A drag saves as it ends, through the file port (RA04); the build and the live scan skip the folder (section 6), and this repository's `.gitignore` leaves it out. How far the graph moved and how near it is are not kept, since it starts at home. Deleting the file lays the graph out from the graph again; a node the Delete key removes leaves it, and one gone otherwise keeps its place there.
- **Colors per kind of node**: a line `color <kind> = <role>` draws the border and the name of each box of that kind in a role `theme.ini` names, so the theme keeps every color and the `.ini` picks among them. A node's files make its kind, the first that fits: `draw` (a `.vert` or a `.frag`), `dispatch` (a `.comp`), `code` (C++ or GLSL and no shader), `data` (files and no code) and `group` (no files of its own); a box of a view the view hosts is a `view`. A kind with no line, and a box selected, are drawn as before. The graph writes the lines back with every save, and a kind or a role there is not, or a kind given twice, is a mistake in the file.
- **A mistake in the file** is named, and the graph lays the view out from the graph alone and never writes over it, so nothing typed there is lost:

  ```text
  {!!!} ide.graph.graph: graph.ini:5: zoom = 1.388: the graph keeps no pan or zoom, as it starts at home; delete the line; the graph lays the view out from the graph alone, and keeps the file as it is
  ```

- **Only what lies in its area is drawn**: a box cut at its edge, a row of text cut at its edge too, so it shows until it goes out of view, and a curve cut at its edge, which the graph hands `curves` as its ground, so a curve to a box dragged out of view runs to the edge. Its ground is a line a pixel wide every 24 pixels at zoom 1 and a stronger one every tenth, which the graph's own shaders draw under the rest, over the whole area at any zoom; zoomed out, the lesser lines fade, so it never crowds.
- **Given no room it draws and takes nothing**, as every part does.
- **Where:** `src/runtime/recipes/library/parts/graph/`, `src/runtime/recipes/library/parts/curves/`, `src/runtime/recipes/library/components/graph-editor/`.

## 15. What a node's C++ can reach

A node's C++ includes `runtime/Operator.h`, and `runtime/View.h` when it reads the graph. The template shows every call, commented; these are the ports besides the pass block of section 4.

**Its own commands.** A node registers commands in `bind`; they last while the node runs, and a node that goes takes them along. The inspect part's `ls`, shortened:

```cpp
void bind(VP::Bind &node) override {
  _ls = node.command("ls", "lists the view as folders: each node with its files, and …");
}
void command(VP::Call &call) override {
  if (!call.is(_ls))
    return;
  for (const VP::Node &node : call.view().nodes) // the view the line addresses, as edited
    call.reply(node.name + '/'); // the answer, a row a reply
  // call.commands().send("w: param set wave amplitude 0.5"); // a line inside this one:
  //   it joins this command's log group, and a refusal fails this command too
}
```

A command name is one command: a second node registering it is refused, as the drop in section 11 shows.

**The graph, during a frame.** `frame.view()` in `cook` is the view the node runs in, as this frame runs it, with each recipe a node of the library uses unfolded; a rebuild replaces it, so a node reads it while it cooks and never keeps it. The modes part lists the views its view hosts this way, a tab each:

```cpp
for (const VP::Child &child : frame.view().children)   // the views its view hosts
  tabs.push_back({.label = child.name, .command = "view switch " + child.name});
```

Any view it reads by name, `frame.view(name)`, empty for the one vulpen started with, as the changes so far left it, which is what a command addressed to that view reads; the graph part lays out the current view so:

```cpp
const VP::View *const shown = frame.view(frame.commands().current());
```

A command reads a view by the name a line addresses it with, `call.view(name)`, empty for the one vulpen started with, as `:` addresses it, so it acts at once on another view than the one its line addressed. `mode perform` typed in the `ide`'s terminal addresses the project, and reads the `ide`'s own connections so:

```cpp
const VP::View &own = *call.view("");   // the ide's, as the lines so far left it
```

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

**Input**: keys, text, the pointer, its buttons, the wheel and focus, as the window got them since the frame before. Every node reads the same events, in order; a part that takes keys reads them from the keys part instead, so only the part in focus gets them (section 14):

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
input button down left 2     # the second press of a double click
input wheel 0 -1
input focus off
```

- **Neither goes through the log.** The log keeps what input caused, an edit or a save, so a replay rebuilds the graph but not the hands that drove it.
- **Key names**: the character a key prints in the keyboard's layout without shift, so `l` and never `L`, or `space`, `enter`, `escape`, `tab`, `backspace`, `insert`, `delete`, `left`, `right`, `up`, `down`, `page_up`, `page_down`, `home`, `end`, `shift`, `control`, `alt`, `super`, `f1` to `f12`; `frame.input().names_key(name)` says whether a name is one, as the keys part checks its keymap. Buttons are `left`, `right` and `middle`. A held key repeats as more `down` events.
- **Presses in a row**: a button pressed carries `event.presses`, its place in a row of presses of that button, each within 500 ms of the one before and 6 pixels of where it was, so a double click's second press is 2 and a third 3. The engine counts them for the window, one interval and one distance for every part; `input button down <button> <count>` gives the count, 1 when left out, so a script's double click does not depend on the clock.
- **The pointer** counts pixels from the window's top left, so it matches the frame block's `cursor`; moves within a frame keep the last.
- **The clipboard**: `frame.input().clipboard()` is the text the system's clipboard holds, and `frame.input().copy(text)` puts text there, as the editor and the terminal do for `control` with `c`, `x` and `v`. Without a window it is a text of the run's own, so a headless run pastes what it copied (V07). Its owner may take a moment to hand it over, which a paste waits for.
- **A script's events all reach the first frame**: the parts read the pointer and its buttons that frame, and only keys wait, as the keys part holds them back. So a script cannot press what shows only later, as a completion Tab lists once `complete` answers, and two presses in different parts both act in the first frame, the second taking the focus before the keys between them reach the first.

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

**Work the end would lose.** A node that holds such work, as the editor a file's changes, calls `frame.hold(what)` in each frame it does. `quit` then refuses, naming the node and what, while the node held this frame or the one before, whichever node sends it and wherever it cooks; the window's close button sends `quit`, and `quit force` ends the run anyway (section 7). A node that stops, or goes, holds nothing, so no hold outlives its node:

```cpp
// _unsaved: "keys/Keys.cpp* has changes: write saves the file shown, …", named when the
// tabs are, so a frame allocates nothing for it (CPP10)
if (!_unsaved.empty())
  frame.hold(_unsaved);
```

A `write` and a `quit` sent in one frame, as a pasted pair of lines, may still be refused once, since the frame before held.

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
- **Where:** `src/runtime/Operator.h`; the template in `src/runtime/recipes/library/parts/library/template/`.

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
| an image C++ fills that nothing samples | `node picture: the operator fills image stray, which nothing samples: no Texture of its shaders, and no connection` | the rest of the view; that node is left out |
| C++ uploads pixels that do not make the size it gives | `node picture: the operator uploads 7 pixels to picture, which is 4 by 2; its operator stops` | the rest of the view; that node stops |
| C++ fills an image with another pixel than its format takes, or the format is none there is | `node picture: the operator fills picture with uint8 pixels, but its format R32_SFLOAT takes float` | the rest of the view; that node is left out |
| shaders that make neither a draw nor a dispatch, or the wrong count | `node fill: it runs a .comp, so it counts its invocations, and no vertex_count` | the rest of the view |
| a draw's image taken to a port that is no `Texture`, or a draw of more colors than one | `node sum: connection painted: paint draws an image, which values is no Texture to sample` | the rest of the view; that node is left out |
| a view's window taken from a view the view does not host, or taken to a view | `view.vlp:8: connection stage is from inner:, but this view hosts no view inner` | nothing: the load stops |
| a view's window taken to a port that is no `Texture` | `node sum: connection stage: view inner draws an image, which values is no Texture to sample` | the rest of the view; that node is left out |
| a param nothing reads, a field nothing sets | `node wave: param spare: nothing reads it` | the rest of the view |
| an edit that cannot apply | `refuse.txt:2: node wave is connected through values; disconnect it first` | nothing changes; a script stops there |
| a command that does not exist, or does not fit its usage | `unknown command frobnicate; the commands are …` | nothing changes |
| a sync of a copy the view changed | `node d changed since it was dropped from c, and a sync would lose that; …` | nothing changes |
| a C++ module whose build failed | the compiler's output | the running code stays |
| two nodes registering one command | `node look: command \`ls\` registers twice` | the second node is left out |
| a keymap line the keys part cannot take | `node ide.keys: keymap.ini:1: no command takes clera; its operator stops` | the rest of the view; keys go nowhere until the file is fixed |
| a node whose C++ is not built yet | `zz-demo/look has no C++ built: …/module.so is missing` | until the live build compiles it |
| a folder no node names | `{ ! } …: folder extra/ is no node's, but every folder is a node: …` | everything: it is a warning |
