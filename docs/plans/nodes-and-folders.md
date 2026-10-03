# Nodes and folders

How a view's C++ and GLSL files and its folders make its nodes, and how a recipe from the library comes into a view and stays in step with it. Proposed and built on 2026-10-03, after `9656e19`, from the lead's reading of [usage](../usage.md), which shows each part at work.

> Recipes seem to have started leading a life of their own. Initially […] they were intended as a way to save a (part of a) view's nodes and drop them together into another view, allowing for them to be modified as if they were built by hand there.

The aim is a design that stays lean, holds nothing new for someone who knows C++ and GLSL (Goal-01), and uses Vulkan's names where Vulkan has one (VK04).

## The rules

1. **A node is a folder, named by its path (RV08).** `ui.panel` is `ui/panel/` in its view, beside the files of `ui` itself in `ui/`. Every node has a section, groups too, and every folder of a view is a node, but its `contracts/` and the folders of the views it hosts. A folder that no node names is a warning at load.
2. **A node's section lists its files, and the folder decides them.** Vulpen fills the `file` lines from the folder at load, after each edit and at each live scan, and `view save` writes them; a command never names a file. A node's first listing is silent, and a change to one is noted at `info`.
3. **The files say what a node runs.** One `.comp` makes a dispatch of `invocations` threads; a `.vert` and a `.frag` make a draw of `vertex_count` vertices, `instance_count` times (VK04). `operator` names the class the folder's C++ registers.
4. **A folder builds as one module.** Its `.cpp` files build together, one of them registers the folder's operators with `VP_OPERATORS`, and the build tree mirrors the folder: `out/build/<preset>/views/<view>/<folder>/`. Saving a node's `.cpp` restarts only that node's operators (live code rule 5).
5. **Includes are plain.** A node's file includes its own folder's files from beside it, what nodes share from `contracts/` at the top of the view or the library (RV05), and the engine's headers as today. No include names another node's folder, so none changes when a node is renamed or moved into a group.
6. **A view names the views it hosts**, as `[view "<name>"]`: the folder `<name>/` beside the manifest, unless `file` names another.

## Recipes

A recipe is a node of the library, named like its folder, with the nodes inside it.

- **A drop copies.** `recipe drop <recipe> <name>` copies the recipe's files into the node's folder, the recipes it uses into the folders of the nodes inside it, and the contracts they include into the view's `contracts/`. It adds the nodes and their connections as edits, and the node keeps `recipe = <recipe>@<fingerprint>`. The copy is the view's: every edit works (V02), and nothing links back (V03).
- **A sync updates a copy left as it was.** `recipe sync <node>` compares the copy with the fingerprint: its nodes' files, operators, counts, and the connections between them, but no param or log level. Unchanged, the copy takes the library's version, and keeps the values the view set for every param the library still has, its log levels and its connections to the rest of the view; a param the library let go goes too, and the sync names it. Changed, the sync refuses, naming the drop that would take the library's version. It refuses, too, when a connection from the rest of the view reaches a node the library let go.
- **The fingerprint** is FNV-1a over 64 bits, folded to 32 and written in hex, so it is the same on every machine (C01).
- **The library uses recipes as they are (V11).** A library recipe's node with `recipe = <name>` and no fingerprint takes the recipe's own node in its place, with the nodes inside it, unfolded at load, and takes only `param` and `log`. A view refuses such a node: it holds copies (V03).

## Commands

| Command | What it does |
| --- | --- |
| `node new draw <name>`, `node new dispatch <name>` | writes a node's first files from the template into its folder, and adds the node as a draw of 3 vertices or a dispatch of 64 invocations |
| `node add <name> [<word=value>...]` | adds a node; with no words, a group. A usage's last placeholder in brackets may be left out |
| `node remove <node>` | refused while a connection names it or a node is inside it; its folder stays |
| `recipe drop <recipe> <name>`, `recipe sync <node>` | above |
| `child add <name> <file>`, `child remove <name>` | edits of the view a line addresses; `child remove` reaches the view that hosts the child, so a project closes from inside |
| `deploy add`, `deploy remove` | gone |

## What was built

- **Text:** V03, V11, RV05 and RV06 reworded, and RV08 added; the include map's rule for node code; usage, the recipe map and the plans that named the old words.
- **Engine:** `runtime/Manifest.cpp` reads and writes `[view]` and `file`, keeps each node's files, and unfolds the recipes the library uses; `runtime/Edits.cpp` holds `child add` and `child remove`; `runtime/Views.cpp` hosts what each manifest names; `runtime/Modules.{h,cpp}`, renamed from `Recipes`, loads a module per folder; `runtime/cmake/nodes.cmake`, renamed from `recipe.cmake`, builds a module per folder; the file port gains `manifest()` and `remove()`, and the platform files `Files::remove`.
- **Library:** the `library` part's `node new`, copying drop and `recipe sync`; its template's files end in `.in`, in its `template` node.
- **Data:** the examples' and the fixture's nodes moved into folders of their own; the CLI app's and the parts' manifests list their nodes and files.
- **Tests:** fail-loud covers the new mistakes, the recipes the library uses, child views, and drops and syncs; golden, fuzz and soak copy whole views. Every preset passes: debug, release, asan and tsan.

## Where the build differs from the proposal

Each is the lead's to keep or undo (A00):

1. **`shader` is gone.** With every file listed, `shader` would list some files twice, so a node's shaders are the `.comp`, `.vert` and `.frag` among its files, as glslang reads the extension. A folder holding a stage a node does not run fails loud, naming its shaders. `operator` stays, since a folder's C++ may register several classes, as the fail-loud fixture's does.
2. **No node runs another node's code.** The proposal let a node name another folder's files by path. Built without it: two nodes of one class are two copies, as two drops are, until something needs sharing (C00).
3. **Every folder is a node, data folders too.** The library part's `template/` and `template/header/` are nodes that hold files and run nothing. A folder no node names is a warning, not an automatic node, so `node remove` never brings a node back by its folder.
4. **A sync lets go of a param the library let go**, naming it, rather than refusing: a sync cannot tell a value the view set from the default it copied, so refusing would stop every sync after the library drops or renames a param.
5. **`node add` takes no words** for a group, through the new optional placeholder.
6. **`child remove` reaches the host** of the child it names, from any view, as the CLI's docs had it.
7. **Renames past the decided ones:** `Recipes` to `Modules`, `recipe.cmake` to `nodes.cmake`, `recipe.so` to `module.so`, and the build target `vulpen_recipes` to `vulpen_modules`, so "recipe" means only the library's.

## Open

- **Shared C++ headers in `contracts/`** are decided, and wait for [native C++](native-cpp.md).
- **Keeping edits through a sync**, by marked comments or a three-way merge, waits until a copy needs it.
- **A sync and the contracts a drop brought:** a contract the view has already stays as it is, so a sync never updates one.
- **Moving a node into a group or out of one** is a rename that moves its folder, which waits until something needs it.
- **The library part copied into a view** finds the library from its own folder, so dropped elsewhere it would look in the wrong place; nothing drops it today.
