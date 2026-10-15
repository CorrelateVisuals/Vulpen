# Recipe map

Every recipe in `src/recipes/`: which recipes it uses, which core ports it uses, and which contracts it reads and writes. Node code has no rows in the [include map](include-map.md), and recipes never include each other (RV00), so that page cannot show how they stack; this page does, for the same reason (A00). The rules are RV05–RV09 in the [requirements](requirements.md) and V11 in the [principles](principles.md). A recipe uses another with a node of its `view.vlp` that names it, as `[node "cli.inspect"] recipe = inspect`. It is kept by hand until every component's and app's `view.vlp` names the recipes it uses; then a gate checks it as `include-map.py` checks the include map.

Paths are from `src/recipes/`.

## Core ports

The core (`baseclasses/`, `runtime/`) knows no recipe. These are the ports the recipes below use:

| Port | What a part gets | Status |
| --- | --- | --- |
| input | keys, pointer and clipboard | `InputPort` |
| files | read, watch and save (RA04) | `FilePort` |
| commands | send text, read the log, list the registered commands (RV04) | `CommandPort` |
| graph | read the view; writes stay commands (V06) | `runtime/View.h`, once [D1](migration-and-implementation.md#open-decisions) is settled |
| connections | the contracts on its connections | [D5](migration-and-implementation.md#open-decisions) |
| terminal | stdin lines and stdout, for a CLI with no window (V07) | `TerminalPort` |

## Contracts

`contracts/` holds `Rect`, `Label`, `Curve`, `Palette`, `Font`, `Item`, `Typed` and `Relation`: each is what one connection carries (RV05).

## Shared code

`parts/shared/` holds code parts share, for a job too small to be a node: a header, and its `.cpp` if it has one, which the header includes (RV09). It reaches no port; the part that includes it hands it what it needs.

| Shared code | What it does | Included by |
| --- | --- | --- |
| `Scroll` | rows shown from a first row that the wheel and a drag on a bar's thumb move, keeping a step's fraction | `text`, `command-line` |
| `Words` | the run a double click selects in a row: letters, digits and `_`, blanks, or other characters | `text`, `command-line` |
| `PanZoom` | how far what an area shows moved and how near it is: a drag moves it, the wheel zooms it about the pointer, and `h` brings it home | `graph`, `image` |

## Parts

Parts hold all of the library's code but the shared code, and use no other recipe (RV06). A drawing part draws into its host's target.

| Part | Code | Ports | Reads | Writes |
| --- | --- | --- | --- | --- |
| `rects` | shaders | — | Rect, Palette | draws |
| `glyphs` | shaders | — | Label, Font, Palette, an image | draws |
| `curves` | shaders | — | Curve, Rect (the area it cuts at), Palette | draws |
| `image` | `Image`, shaders | input and commands (the focus), with `moves=yes` | Rect, an image: a draw's, a hosted view's window, or one C++ fills; Typed, with `moves=yes` | draws the image into the Rect, letterboxed, and with `moves=yes` moved and zoomed, cut at the Rect's edge |
| `font` | `Font` | files, commands | — | Font, an image |
| `palette` | `Palette` | files | — | Palette |
| `viewport` | `Viewport` | — | — | Rect: the window's |
| `split` | `Split` | input, commands | Rect | Rect: two, and the ground and seam for `rects` |
| `bar` | `Bar` | — | Rect, Font | Rect: two, the bar and the rest |
| `list` | `List` | — | Item, Rect, Font | Rect, Label; Rects, where each Item shows, for `hit` |
| `hit` | `Hit` | input, commands | Rect, Item | — (the hovered Item once a tooltip reads it) |
| `keys` | `Keys` | input, files, commands, graph | — | Typed: the keys and text no chord takes, and the node in focus |
| `text` | `Text` | input, files, commands | Font, Rect, Typed | Label, Rect, Item (a tab a file open) |
| `command-line` | `CommandLine` | input, commands (the log too), terminal | Font, Rect, Typed | Label, Item (its completions, and its tab), Rect |
| `command-items` | `CommandItems` | commands | Item | Item |
| `graph` | `Graph`, shaders | input, files, commands, graph | Rect, Font, Typed, Palette (Relation once relations land) | Rect: its boxes; Label, Curve, Item (its tab); draws its ground, lines that move and zoom with it |
| `relations` | `Relations` | files | — | Relation |
| `modes` | `Modes` | commands, graph | Rect, Typed (the keys no part in focus takes) | Rect: two, the dock's room and what is presented; Item (a tab a hosted view); the connection that presents a view |
| `inspect` | `Inspect` | commands, graph | — | — |
| `library` | `Library` | files, commands | — | — |

## Components

| Component | Uses |
| --- | --- |
| `panel` | bar, list, hit, rects, glyphs |
| `dock` | split, rects |
| `menu` | command-items, list, hit, rects, glyphs |
| `tooltip` | list, rects, glyphs |
| `text-area` | text, rects, glyphs |
| `find-bar` | command-line, rects, glyphs |
| `terminal` | command-line, list, hit, rects, glyphs |
| `graph-editor` | graph, relations, rects, glyphs, curves |

## Apps

| App | Uses |
| --- | --- |
| `cli` | command-line, inspect, library |
| `ide` | palette, font, viewport, keys, modes, image, inspect, library; dock, panel ×4, text-area, find-bar, terminal, graph-editor, menu ×2, tooltip |
