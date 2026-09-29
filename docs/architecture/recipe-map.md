# Recipe map

Every recipe in `src/recipes/`: what it deploys, which core ports it uses, and which contracts it reads and writes. Recipes never include each other (RV00), so the [include map](include-map.md) cannot show how they stack; this page does, for the same reason (A00). The rules are RV05–RV07 in the [requirements](requirements.md) and V11 in the [principles](principles.md). It is kept by hand until the `view.vlp` files declare their children; then a gate checks it as `include-map.py` checks the include map.

Paths are from `src/recipes/`.

## Core ports

The core (`baseclasses/`, `runtime/`) knows no recipe. These are the ports the recipes below use:

| Port | What a part gets | Status |
| --- | --- | --- |
| input | keys and pointer | `InputPort` |
| files | read, watch and save (RA04) | `FilePort` |
| commands | send text, read the log, list the registered commands (RV04) | `CommandPort` |
| graph | read the view; writes stay commands (V06) | [D1](migration-and-implementation.md#open-decisions) |
| connections | the contracts on its connections | [D5](migration-and-implementation.md#open-decisions) |
| terminal | stdin lines and stdout, for a CLI with no window (V07) | not a port yet; `Terminal` is in `baseclasses/Platform.h` |

## Contracts

`contracts/` holds `Rect`, `Label`, `Curve`, `Palette`, `Font`, `Item` and `Relation`: each is what one connection carries (RV05).

## Parts

Parts hold all recipe code and deploy nothing (RV06). A drawing part draws into its host's target.

| Part | Code | Ports | Reads | Writes |
| --- | --- | --- | --- | --- |
| `rects` | shaders | — | Rect, Palette | draws |
| `glyphs` | shaders | — | Label, Font, Palette | draws |
| `curves` | shaders | — | Curve, Palette | draws |
| `image` | shaders | — | Rect, an image handle | draws |
| `font` | `Font` | files | — | Font |
| `palette` | `Palette` | files | — | Palette |
| `split` | `Split` | — | Rect | Rect |
| `list` | `List` | — | Item, Rect | Rect, Label, Item |
| `hit` | `Hit` | input, commands | Rect, Item | Item, Rect |
| `keys` | `Keys` | input, files, commands | — | — |
| `text` | `Text` | input, files, commands | Rect | Label, Rect |
| `command-line` | `CommandLine` | input, commands, terminal | Rect | Label, Item, Rect |
| `command-items` | `CommandItems` | commands | Item | Item |
| `graph` | `Graph`, shaders | input, commands, graph | Rect, Relation | Rect, Label, Curve; draws its backdrop |
| `relations` | `Relations` | files | — | Relation |
| `modes` | `Modes` | commands | — | — |

## Components

| Component | Deploys |
| --- | --- |
| `panel` | list, hit, rects, glyphs |
| `dock` | split, hit, rects |
| `menu` | command-items, list, hit, rects, glyphs |
| `tooltip` | list, rects, glyphs |
| `text-area` | text, rects, glyphs |
| `find-bar` | command-line, rects, glyphs |
| `terminal` | command-line, list, rects, glyphs |
| `graph-editor` | graph, relations, rects, glyphs, curves |

## Apps

| App | Deploys |
| --- | --- |
| `cli` | command-line |
| `ide` | palette, font, keys, modes, image; dock, panel ×4, text-area, find-bar, terminal, graph-editor, menu ×2, tooltip |
