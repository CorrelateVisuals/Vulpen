# CLI examples

Two views built from an empty folder with the CLI that the [IDE port](ide-port.md) proposes. For each: what a person types, which files they write, and what the engine does with them. Nothing here runs yet. The commands, the `[deploy]` section and the engine additions are proposals of the rows each section names.

- Example 1's recipe code is the code `src/examples/triangle/` runs today.
- Example 2's shaders and C++ compile with the build's flags against the engine additions shown [below](#what-the-engine-adds). This was checked on 2026-09-30 with glslang 15.1 and g++ 13.

## Where the engine stops

A recipe reaches the engine through two headers and a folder layout, and nothing else:

| | Engine (the author never edits it) | Recipe (what the author writes) |
| --- | --- | --- |
| C++ | [`runtime/Operator.h`](../../src/runtime/Operator.h): `Operator` with `bind` and `cook`. `Bind` hands out `Value<T>`, `Upload<T>`, `Readback<T>` and params by name, and `Cook` writes and reads them each frame. `Registry` and `VP_RECIPE` register a class. | one class per operator, in an unnamed namespace, and one `VP_RECIPE` entry |
| GLSL | [`baseclasses/GpuLayout.glsl`](../../src/baseclasses/GpuLayout.glsl): buffers as device addresses. The pass block lives at set 1, binding 0. | the pass block's fields, and the shaders |
| Build | [`runtime/cmake/recipe.cmake`](../../src/runtime/cmake/recipe.cmake) compiles each recipe folder: shaders to SPIR-V, and C++ to a module (debug) or into `vulpen` (release). It builds the views in `src/examples/`. | files in `<view>/recipes/<recipe>/` |
| Load | `runtime/Schedule` reflects the SPIR-V, runs `bind` and checks every name (A02). It makes the buffers, blocks and pipelines, and places the barriers. | nothing |
| Graph | the command port: the primitive edits, the log and save | commands, or a `view.vlp` written by hand |
| Structure | the include-map gate | the rows it prints, added once a person approves them (A00) |

A recipe's C++ never sees Vulkan, `Engine`, a buffer object or the OS. It gets names in and handles out, so its module has no engine symbol, and a swap never leaves it holding a GPU object that is gone ([live code](live-code.md)).

### The engine stubs these examples fill

The engine already has placeholders for most of what the examples need:

| Stub today | File | Becomes | Row | Example |
| --- | --- | --- | --- | --- |
| `CommandPort`, `Command` and `CommandLog`, all empty | `runtime/Commands.h` | the command port, the primitive edits and the log | B1–B4 | 1, 2 |
| `Terminal`, not a port yet | `baseclasses/Platform.h` | the terminal port: lines in, text out | B9 | 1, 2 |
| `FilePort`, empty | `runtime/Operator.h` | read, watch and save, for parts | B7 | 1, 2 |
| `CommandLine`, an empty `Operator` | `recipes/parts/command-line/` | the command-line part | E12 | 1, 2 |
| `Image`, empty | `baseclasses/Resources.h` | images the schedule owns | A5 | 2 |
| the set 0 layout, with no arrays yet | `baseclasses/Pipelines.h` | `textures[]`, `samplers[]` and `image2D[]` | A5 | 2 |
| `InputPort`, empty | `runtime/Operator.h` | keys, text and pointer | B6 | neither |

Deploys (C1), hosted views (C3), the window on demand (B14) and the `inspect` and `library` parts have no stub yet. Everything in the table is engine or library code; neither example writes any of it.

## Example 1: a triangle

A window with one triangle, whose corners and tint C++ writes every frame. Suppose `src/examples/triangle/` did not exist yet.

It needs phase 1 of the [order](ide-port.md#order), steps 1 to 11. The triangle's own code needs nothing new.

### The session

```text
$ ./run.sh src/recipes/apps/cli/view.vlp
> view new src/examples/triangle
triangle>
```

The `cli` app runs headless, with a prompt on the terminal. `view new` hosts an empty view and saves its `view.vlp`, and from then on the prompt names the view that typed lines go to.

Next, the four files [below](#the-files) go into `src/examples/triangle/recipes/triangle/`, typed in any editor. The build refuses them until the include map lists them. For each file, the gate prints the row to add:

```text
examples/triangle/recipes/triangle/Triangle.cpp: new file; once approved, add the row
      | `examples/triangle/recipes/triangle/Triangle.cpp` | `runtime/Operator.h` |
```

Once the rows are in, the next save in the view builds, and the node can be added:

```text
triangle> node add triangle recipe=triangle operator=Triangle shader=Triangle.vert shader=Triangle.frag invocations=3
triangle: the operator reads param speed, which the node does not set
triangle> param set triangle speed 0.01
triangle> info triangle
triangle> view save
```

- `node add` takes the manifest's own words (B2), and the schedule reruns without a restart. The loader finds a mistake at once and names it (A02). The node stays out until `param set` fixes it.
- Then the window opens (B14) and the triangle turns.
- `info` shows what the loader bound: the fields, who sets each, and the params (E17).
- `view save` writes `view.vlp` to a temp file and renames it over the old one (RA04).

From here, every save of a recipe file swaps in while the triangle keeps running.

### The files

`Triangle.cpp`, the operator:

```cpp
#include "runtime/Operator.h"

#include <cmath>
#include <numbers>

namespace {

constexpr float radius = 0.8f; // of the circle the corners ride on, in clip space
constexpr float turn = 2.0f * std::numbers::pi_v<float>;
constexpr float opaque = 1.0f;

// Between 0 and 1, following the angle round.
float swing(float angle) {
  return (1.0f + std::cos(angle)) / 2.0f;
}

// The triangle lives on the CPU: C++ writes its corners and its tint, and the shaders
// only draw them.
class Triangle final : public VP::Operator {
  void bind(VP::Bind &node) override {
    _corners = node.upload<glm::vec2>("corners");
    _tint = node.value<glm::vec4>("tint");
    _speed = node.param<float>("speed");
  }
  void cook(VP::Cook &frame) override {
    _angle = std::fmod(_angle + _speed, turn);
    const std::span<glm::vec2> corners = frame.write(_corners);
    const float step = turn / static_cast<float>(corners.size());
    for (std::size_t index = 0; index < corners.size(); ++index) {
      const float at = _angle + step * static_cast<float>(index);
      corners[index] = radius * glm::vec2(std::cos(at), std::sin(at));
    }
    frame.set(
        _tint,
        glm::vec4(swing(_angle), swing(_angle + step), swing(_angle - step), opaque));
  }

  VP::Upload<glm::vec2> _corners;
  VP::Value<glm::vec4> _tint;
  float _speed = 0;
  float _angle = 0;
};

} // namespace

VP_RECIPE(registry) {
  registry.add<Triangle>("Triangle");
}
```

`Triangle.glsl`, the pass block both shaders include:

```glsl
// The pass block both of the triangle's shaders read. Each includes this one
// declaration, so the loader finds the same block in both.
#include "baseclasses/GpuLayout.glsl"

layout(buffer_reference, std430) readonly buffer Corners {
  vec2 at[];
};

layout(set = 1, binding = 0) uniform Pass {
  vec4 tint;       // set by the Triangle operator every frame
  Corners corners; // written by the Triangle operator every frame, one per vertex
} pass;
```

`Triangle.vert` and `Triangle.frag`:

```glsl
#version 460
#extension GL_GOOGLE_include_directive : require
#include "Triangle.glsl"

void main() {
  gl_Position = vec4(pass.corners.at[gl_VertexIndex], 0.0, 1.0);
}
```

```glsl
#version 460
#extension GL_GOOGLE_include_directive : require
#include "Triangle.glsl"

layout(location = 0) out vec4 color;

void main() {
  color = pass.tint;
}
```

The rows the gate asks for, as the include map holds them today:

```text
| `examples/triangle/recipes/triangle/Triangle.cpp` | `runtime/Operator.h` |
| `examples/triangle/recipes/triangle/Triangle.frag` | `examples/triangle/recipes/triangle/Triangle.glsl` |
| `examples/triangle/recipes/triangle/Triangle.glsl` | `baseclasses/GpuLayout.glsl` |
| `examples/triangle/recipes/triangle/Triangle.vert` | `examples/triangle/recipes/triangle/Triangle.glsl` |
```

### What `view save` writes

```ini
[manifest]
version = 1

[node "triangle"]
recipe      = triangle
operator    = Triangle
shader      = Triangle.vert
shader      = Triangle.frag
invocations = 3
param       = speed=0.01
```

The copy in the tree also carries comments a person wrote. The writer keeps them (B4).

### The log

```text
child add triangle src/examples/triangle/view.vlp
triangle: view save
triangle: node add triangle recipe=triangle operator=Triangle shader=Triangle.vert shader=Triangle.frag invocations=3
triangle: param set triangle speed 0.01
triangle: view save
```

The first two lines are one group: what `view new` expanded to. Every line is a primitive and names the view it edits, so `source` replays the log headless, with no recipe loaded (V08).

### What the engine does with it

1. **Build.** `recipe.cmake` compiles `Triangle.vert` and `Triangle.frag` to SPIR-V. It compiles `Triangle.cpp` to `recipe.so` (debug) or into `vulpen` (release). All of it goes under `out/build/<preset>/views/triangle/recipes/triangle/`.
2. **Load.** Reflection finds the pass block both shaders declare: `vec4 tint` at byte 0, and `readonly vec2[] corners` at byte 16. `bind` asks for each by name, and the loader checks each request against the shader:
   - `value<glm::vec4>("tint")` gets offset 0, once the shader's type is known to be a `vec4`;
   - `upload<glm::vec2>("corners")` gets a handle, once the shader is known only to read it and to hold 8-byte elements;
   - `param<float>("speed")` parses `0.01`.

   The loader then checks that every field has a writer and every param a reader. It makes a 24-byte buffer the CPU writes (3 invocations of 8 bytes), the pass block and the pipeline.
3. **Every frame.** `cook` gets a span of three `glm::vec2` over the mapped buffer and writes the corners, then sets `tint`. The engine flushes both and draws 3 vertices in the window's render pass. Nothing is looked up by name during a frame.
4. **A save.** A shader save makes a new pipeline and keeps the buffer. A C++ save unloads the module, makes a new `Triangle` and runs `bind` again.

## Example 2: instanced cubes on the triangle

64 cubes, drawn in one draw into an image, shown on the triangle from example 1. First, the triangle becomes a library part that can be dropped by itself.

On top of phase 1, it needs:

- A1 and A5 (phase 2) and A7 (phase 4);
- three later rows: A9 (`mat4` values), A14 (depth) and D17 (`recipe publish`).

It needs nothing else from phases 2 to 4, so it could run straight after phase 1 if these six rows came next.

### Publish the triangle

```text
triangle> recipe publish triangle
```

This copies `src/examples/triangle/recipes/triangle/` to `src/recipes/parts/triangle/`, and writes the part's `view.vlp` from the node that runs it. A person adds the comment at the top:

```ini
# The triangle part: one draw whose corners and tint come from C++.
[manifest]
version = 1

[node "triangle"]
recipe      = triangle
operator    = Triangle
shader      = Triangle.vert
shader      = Triangle.frag
invocations = 3
param       = speed=0.01
```

The part runs its own node, so it can be dropped by itself: `recipe drop triangle` into an empty view gives example 1 back. The [recipe map](../architecture/recipe-map.md) gains a row for it under Parts, and the include map gains the four rows the gate prints.

### The session

```text
triangle> view new src/examples/cube-screen
cube-screen> recipe drop triangle as screen
cube-screen> node add layout recipe=cubes shader=Layout.comp invocations=64
cube-screen> param set layout spacing 0.6
cube-screen> node add cubes recipe=cubes operator=Cubes shader=Cubes.vert shader=Cubes.frag invocations=2304
cube-screen> param set cubes speed 0.02
cube-screen> connect offsets layout.offsets cubes.offsets
cube-screen> connect picture cubes.color screen.triangle.picture
cube-screen> view save
```

- **The drop.** `recipe drop triangle as screen` copies the part into the view as `recipes/triangle/` (V03) and deploys it as `screen` (C1). The triangle runs at once, as in example 1. The gate asks for rows for the copy's four files, since the view lives under `src/`.
- **The new files.** The cubes' files go into `recipes/cubes/`, and the view's copy of the triangle is edited to show a picture, before the two `connect` lines. Until then, the loader names what is missing, and the rest of the view keeps running.
- **The counts.** `layout` runs one invocation per cube. `cubes` draws 36 vertices for each of the 64 cubes: 2,304.

### What `view save` writes

```ini
[manifest]
version = 1

[deploy "screen"]
recipe = triangle

[node "layout"]
recipe      = cubes
shader      = Layout.comp
invocations = 64
param       = spacing=0.6

[node "cubes"]
recipe      = cubes
operator    = Cubes
shader      = Cubes.vert
shader      = Cubes.frag
invocations = 2304
param       = speed=0.02

[connection "offsets"]
from = layout.offsets
to   = cubes.offsets

[connection "picture"]
from = cubes.color
to   = screen.triangle.picture
```

The view's folder:

```text
src/examples/cube-screen/
  view.vlp
  recipes/
    triangle/  the drop's copy, owned by this view and edited here to show a picture
      view.vlp  Triangle.cpp  Triangle.glsl  Triangle.vert  Triangle.frag
    cubes/     written for this view
      Cubes.cpp  Cubes.glsl  Cubes.vert  Cubes.frag  Layout.comp
```

### The cubes' files

`Layout.comp` places the cubes, one invocation each, with no C++:

```glsl
#version 460
#extension GL_GOOGLE_include_directive : require
#include "baseclasses/GpuLayout.glsl"

layout(local_size_x = 64) in;

layout(buffer_reference, std430) writeonly buffer Offsets {
  vec4 at[];
};

layout(set = 1, binding = 0) uniform Pass {
  float spacing;   // a param: the distance between neighbouring cubes
  Offsets offsets; // one centre per invocation, read by the cubes node
} pass;

const uint edge = 4; // cubes along each side of the lattice, so 64 fill it

void main() {
  const uint i = gl_GlobalInvocationID.x;
  const vec3 cell = vec3(i % edge, i / edge % edge, i / (edge * edge));
  pass.offsets.at[i] = vec4((cell - float(edge - 1) / 2.0) * pass.spacing, 1.0);
}
```

`Cubes.glsl`, the pass block both of the draw's shaders include:

```glsl
// The pass block both of the cubes' shaders read.
#include "baseclasses/GpuLayout.glsl"

layout(buffer_reference, std430) readonly buffer Offsets {
  vec4 at[]; // one per cube: its centre in the lattice
};

layout(set = 1, binding = 0) uniform Pass {
  mat4 spin;       // set by the Cubes operator every frame
  Offsets offsets; // connected: the layout node writes it
} pass;
```

`Cubes.vert` builds every cube from `gl_VertexIndex`, with no vertex buffer:

```glsl
#version 460
#extension GL_GOOGLE_include_directive : require
#include "Cubes.glsl"

layout(location = 0) out vec3 normal;

// One draw covers every cube: gl_VertexIndex picks the cube, then the corner.
const uint vertices_per_cube = 36; // 6 faces of 2 triangles
const uint vertices_per_face = 6;
const uint corners_per_face = 4;
const float half_edge = 0.25; // of one cube
const float distance = 3.5;   // from the eye to the lattice's centre
const float fov_y = radians(45.0);
const float near = 0.1;
const float far = 10.0;

const vec3 corner[8] = vec3[8](vec3(-1, -1, -1), vec3(1, -1, -1), vec3(1, 1, -1),
                               vec3(-1, 1, -1), vec3(-1, -1, 1), vec3(1, -1, 1),
                               vec3(1, 1, 1), vec3(-1, 1, 1));
const uint face_corner[24] = uint[24](1, 2, 6, 5, 0, 3, 7, 4, 3, 2, 6, 7,
                                      0, 1, 5, 4, 4, 5, 6, 7, 0, 1, 2, 3);
const uint face_triangles[6] = uint[6](0, 1, 2, 0, 2, 3);
const vec3 face_normal[6] = vec3[6](vec3(1, 0, 0), vec3(-1, 0, 0), vec3(0, 1, 0),
                                    vec3(0, -1, 0), vec3(0, 0, 1), vec3(0, 0, -1));

void main() {
  const uint cube = uint(gl_VertexIndex) / vertices_per_cube;
  const uint vertex = uint(gl_VertexIndex) % vertices_per_cube;
  const uint face = vertex / vertices_per_face;
  const uint at = face_corner[face * corners_per_face + face_triangles[vertex % vertices_per_face]];
  const mat3 spin = mat3(pass.spin);
  const vec3 seen = spin * (corner[at] * half_edge + pass.offsets.at[cube].xyz) -
                    vec3(0.0, 0.0, distance);
  normal = spin * face_normal[face];
  const float aspect = float(frame.resolution.x) / float(frame.resolution.y);
  const float focal = 1.0 / tan(fov_y / 2.0);
  // Vulkan's clip space: y points down, and depth runs from 0 at near to 1 at far.
  gl_Position = vec4(seen.x * focal / aspect, -seen.y * focal,
                     (seen.z * far + near * far) / (near - far), -seen.z);
}
```

`Cubes.frag`. Its output `color` is the port the `picture` connection names:

```glsl
#version 460
#extension GL_GOOGLE_include_directive : require
#include "Cubes.glsl"

layout(location = 0) in vec3 normal;
layout(location = 0) out vec4 color; // the node's picture; a connection names it cubes.color

const vec3 light = normalize(vec3(0.4, 0.7, 0.6));
const vec3 base = vec3(0.8, 0.5, 0.3);
const float ambient = 0.2;
const float opaque = 1.0;

void main() {
  color = vec4(base * (ambient + max(dot(normalize(normal), light), 0.0)), opaque);
}
```

`Cubes.cpp` spins the lattice. The matrix goes from glm straight into the pass block, since a `glm::mat4` has the memory layout of a GLSL `mat4`:

```cpp
#include "runtime/Operator.h"

#include <glm/ext/matrix_transform.hpp>

#include <cmath>
#include <numbers>

namespace {

constexpr float turn = 2.0f * std::numbers::pi_v<float>;
constexpr float tilt = 0.6f; // the x turn, as a share of the y turn

// Spins the lattice on the CPU; the shaders place, project and shade every cube.
class Cubes final : public VP::Operator {
  void bind(VP::Bind &node) override {
    _spin = node.value<glm::mat4>("spin");
    _speed = node.param<float>("speed");
  }
  void cook(VP::Cook &frame) override {
    _angle = std::fmod(_angle + _speed, turn);
    const glm::mat4 yaw = glm::rotate(glm::mat4(1.0f), _angle, glm::vec3(0, 1, 0));
    frame.set(_spin, glm::rotate(yaw, tilt * _angle, glm::vec3(1, 0, 0)));
  }

  VP::Value<glm::mat4> _spin;
  float _speed = 0;
  float _angle = 0;
};

} // namespace

VP_RECIPE(registry) {
  registry.add<Cubes>("Cubes");
}
```

### The triangle's copy, edited

`Triangle.cpp` stays as it is. The pass block gains one field:

```glsl
layout(set = 1, binding = 0) uniform Pass {
  vec4 tint;       // set by the Triangle operator every frame
  Corners corners; // written by the Triangle operator every frame, one per vertex
  Texture picture; // connected: the image the triangle shows
} pass;
```

The vertex shader places each corner on the picture:

```glsl
#version 460
#extension GL_GOOGLE_include_directive : require
#include "Triangle.glsl"

layout(location = 0) out vec2 uv;

// Where each corner sits on the picture: top middle, bottom left, bottom right.
const uint corners_per_triangle = 3;
const vec2 corner_uv[3] = vec2[3](vec2(0.5, 0.0), vec2(0.0, 1.0), vec2(1.0, 1.0));

void main() {
  gl_Position = vec4(pass.corners.at[gl_VertexIndex], 0.0, 1.0);
  uv = corner_uv[uint(gl_VertexIndex) % corners_per_triangle];
}
```

The fragment shader samples it, still tinted from C++:

```glsl
#version 460
#extension GL_GOOGLE_include_directive : require
#include "Triangle.glsl"

layout(location = 0) in vec2 uv;
layout(location = 0) out vec4 color;

void main() {
  color = pass.tint * sample_linear(pass.picture, uv);
}
```

The rows the gate prints for the view's files:

```text
| `examples/cube-screen/recipes/cubes/Cubes.cpp` | `<glm/ext/matrix_transform.hpp>` `runtime/Operator.h` |
| `examples/cube-screen/recipes/cubes/Cubes.frag` | `examples/cube-screen/recipes/cubes/Cubes.glsl` |
| `examples/cube-screen/recipes/cubes/Cubes.glsl` | `baseclasses/GpuLayout.glsl` |
| `examples/cube-screen/recipes/cubes/Cubes.vert` | `examples/cube-screen/recipes/cubes/Cubes.glsl` |
| `examples/cube-screen/recipes/cubes/Layout.comp` | `baseclasses/GpuLayout.glsl` |
| `examples/cube-screen/recipes/triangle/Triangle.cpp` | `runtime/Operator.h` |
| `examples/cube-screen/recipes/triangle/Triangle.frag` | `examples/cube-screen/recipes/triangle/Triangle.glsl` |
| `examples/cube-screen/recipes/triangle/Triangle.glsl` | `baseclasses/GpuLayout.glsl` |
| `examples/cube-screen/recipes/triangle/Triangle.vert` | `examples/cube-screen/recipes/triangle/Triangle.glsl` |
```

### What the engine adds

Example 2 uses three things the engine does not have yet. All of them go into the two headers a recipe already includes.

**`baseclasses/GpuLayout.glsl`** gains the images (A5) and the frame block (A1):

```glsl
#extension GL_EXT_nonuniform_qualifier : require

// Every sampled image, and the static samplers (RV02). A pass block names an image by a
// Texture, its index in textures[]; the loader fills it from a connection.
layout(set = 0, binding = 0) uniform texture2D textures[];
layout(set = 0, binding = 1) uniform sampler samplers[];

struct Texture {
  uint index; // 0 means unbound (RV02)
};

const uint linear_clamp = 0; // samplers[0] to samplers[3] are the static samplers

// Explicit level 0: no derivatives, so every stage and every GPU samples alike (GLSL02).
vec4 sample_linear(Texture image, vec2 uv) {
  return textureLod(sampler2D(textures[image.index], samplers[linear_clamp]), uv, 0.0);
}

// What every pass may read about the frame; the push constant holds its address (RV02).
layout(buffer_reference, std430) readonly buffer FrameBlock {
  uvec2 resolution; // of the window in pixels; an offscreen target has the same size
  vec2 cursor;      // in pixels, from the top left
  float time;       // seconds, counted from the frame index, so a replay matches (C01)
  uint index;       // frames since the view loaded
};

layout(push_constant) uniform Push {
  FrameBlock frame;
};
```

**`runtime/Operator.h`** gains `mat4` (A9). The map gains the edge `runtime/Operator.h` → `<glm/mat4x4.hpp>`:

```cpp
#include <glm/mat4x4.hpp>

template <> inline constexpr std::string_view glsl_type<glm::mat4> = "mat4";
```

**The loader** gains what the headers promise:

- Reflection reads a `mat4` (column-major, 16 bytes between columns, as glm lays it out) and a field whose type is the struct `Texture`.
- Reflection also reads a fragment shader's outputs, which are ports.
- The schedule makes an offscreen image with depth for a connected output (A7, A14).
- It fills a `Texture` with that image's index, and writes the frame block every frame (A1).

The compiled SPIR-V carries every name this needs: the struct's name `Texture`, the output's name `color`, and `ColMajor` with `MatrixStride 16` on `spin`.

### What the engine does with it

Every frame:

1. `layout` dispatches one workgroup of 64 and writes 64 centres into `offsets`: a 1,024-byte buffer that stays on the GPU, since nothing on the CPU reads or writes it.
2. `Cubes::cook` sets `spin` in the cubes' pass block.
3. `cubes` draws 2,304 vertices. Its output `color` is connected, so it renders in its own render pass, into an image the window's size with a depth attachment. A barrier first makes `offsets` visible to the vertex shader.
4. A barrier turns the image from a color attachment into a sampled image. `screen.triangle` then draws 3 vertices into the window. It samples the image through `picture`, which the loader filled with the image's index in `textures[]`.

The order follows the graph: `layout` writes what `cubes` reads, and `cubes` writes what `screen.triangle` reads.

### Choices example 2 makes

Each is a decision for the row named, and the lead settles it (A00).

- **Instancing is vertex pulling (V04).** `invocations = 2304` counts every vertex of every cube, and the shader splits `gl_VertexIndex` into the cube and its vertex. `invocations` already does this job, so V04 allows no new word for it. For Vulkan's own instancing, the word would be `instance_count` (VK04), and the shader would read `gl_InstanceIndex`.
- **The two counts are tied by hand.** 2,304 is 36 vertices for each of `layout`'s 64 cubes. A larger count would read past `offsets`, which GLSL02 forbids and which the loader cannot see. A4 could tie them, by letting a draw's count follow the length of a buffer it reads.
- **An image input is a `Texture` field (A5).** Reflection knows it by the struct's name, and the loader fills its index from a connection.
- **A draw's output is a port (A7).** Connected, the draw renders into an image. Unconnected, it renders into the window.
- **An offscreen target takes the window's size (A7).** A headless view has no window, so its targets need a size from elsewhere. That is still open.
- **Every target has depth (A14).** It is tested less-or-equal, so a flat draw at one depth is unaffected and no word is needed.
- **The view edits its copy of the triangle (V02, V03).** The library part stays example 1's triangle. The other way is to give the part an optional picture. That needs a rule for an unconnected `Texture` (for example, index 0 samples white), and A02 would have to allow it.
- **No contract is needed (RV05).** `offsets` joins two nodes of one recipe, and `picture` carries an image, not a struct.
