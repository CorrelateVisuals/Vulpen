# Graph and relations

A proposal the lead accepted whole on 2026-10-16 (A00); its [order](#order) says which steps are built. The ask, on 2026-10-16, in a note beside the docs (items 4 to 7 of it; the [handoff](../logs/development-20261016T205135Z.md) answers 1 to 3):

- "4. ide nodes visible but not their files"
- "Nodes in the POC behaved and looked better. We need to get closer to how POC handled the graph of nodes."
- "5. we also need to be able to load baseclasses and runtime or any other external folders repository; look to POC how it managed to load this vulpen repo. Supported languages: C++, GLSL, Python, CMake, ini, manifest.vlp and shell."
- "6. nodes should show more information, in line with POC relationships and connections. Altough we need to resolve the problem of not enough space for edges and bottom of internal connectors; make it look beautifuly insightful."
- "7. we also need the toggle buttons to turn on or off certain connections."
- "If this is part of ide-port-plan.md prioritize these parts, else do it first."

Items 6 and 7 are rows of the [IDE port](ide-port.md) not yet ticked: E16, the `relations` part, and D20, `relations <kinds>`; F5 waits for E16. Item 5 is what fed E16 in the POC. So they come first, after the [terminal app](terminal-app.md), which is built.

## What the POC did

Read from `Generations/vulpen` at `50df0ac0`, its canvas's goldens and its code.

- **A folder anywhere as a node.** `[node "Base"] kind = src, nest = true, root = ../src/vulkan_base` showed the engine's own sources as a box holding a tile a file and a box a subfolder; `nest root Rework vulpen-rework` aimed one at a sibling checkout. Every file showed, of any language.
- **Relations, 13 kinds in 5 layers**, each drawn by its own hue and dash: *flow* (a connection; what a mode presents), *structure* (a node inside one; an instance's clones; a source and its header, `implements`; a class and its base's file, `inherits`), *code* (`#include`, Python `import`; the capabilities a node uses; the ABI stamp), *build* (a CMake file naming a file) and *docs* (a Markdown link; an id a doc defines, as `RV00`, cited elsewhere; a path mentioned in prose). The engine read what C++ states; a recipe, `ide-relations`, read the rest through the file port, cached by size and time, so a change redrew in about 2 s.
- **Toggles.** `relations [all|none|<kind|layer>...|+<word>|-<word>]`, all on at first, and a row of buttons along the canvas's bottom edge: `flow`, `structure`, `code`, `build`, `docs`, then `hidden` and `loose`.
- **Wires** run in right angles, and one with no port to land on leaves a tile's bottom edge, labeled with what it reaches.

## What this tree has

- **The graph part** (728 lines): a box a node, its name and a row a file; a curve a connection, from the writer's right to the reader's left, along a lane below each column it passes; a box a hosted view; selection, places and a color a kind, in `.vulpen/graph.ini`.
- **A node that uses a library recipe as it is**, as every node of the cli and the ide does, has no files of its own: its code is the recipe's, in the library (item 4). Its box shows its name alone.
- **The `relations` part and the `Relation` contract** are empty skeletons, a comment each.
- **The ports a part needs are there**: the file port lists folders and reads files, a node reads any view by name, and the graph draws curves of any color.

## Decisions

### 1. A folder in the graph (item 5)

- **A, pick. `open <folder>` shows the folder in the graph**, while the ide runs, as `open <file>` shows a file in its editor: the `editor` part gains a second target, as its `window` param picks the editor (a setting, not a mechanism), and the graph shows the folder in place of the current view, a box a folder, nested as nodes nest (RV08), a row a file, its tab `graph - src/baseclasses`; `view switch` shows a view again. Without the ide, `open <folder>` starts the person's editor, as now. Recipes only, about 150 lines.
  - Pros: no new command or word (V04, V06); a folder is something the ide looks at, not part of a project (V03), so nothing in a manifest names a path outside its view.
  - Cons: the folder shows only while the graph shows it; the log keeps the param that names it, as it keeps a file opened.
- **B. A graph command, `graph <folder>`.** Pros: plain. Cons: a command more, beside `open`, that does what `open` does for files.
- **C. A manifest word, as the POC's `root =`**: a node of a view standing for a folder elsewhere. Pros: a project keeps the folders it watches. Cons: a word whose node runs nothing and whose folder leaves the view (V03, V04); the engine learns a kind of node.

### 2. Which relations, found by whom (item 6)

- **A, pick. The `relations` part finds them through the file port**, for the folder or the view shown, read again only when a file changes, and hands them to the graph as `Relation`s, a pair of paths and a kind; the graph draws each kind in a hue of the theme, by layer. First kinds:
  - *flow*: connections and a view's window taken, as now;
  - *structure*: a node inside a node, and a recipe a node uses;
  - *code*: `#include` in C++ and GLSL, `import` in Python;
  - *build*: a CMake file naming a file or a folder;
  - *docs*: a Markdown link to a file, and an id (`RV00`) a doc defines, cited in code or another doc.
  - Recipes only (V05), about 300 lines: the part, its parsers, and the graph's drawing.
  - Pros: the graph answers which file needs which, which test or doc names which code, what CMake builds; a parser a format, each small.
  - Cons: reading a tree of files costs time after each change; the POC's looked again every 120 frames, about 2 s, and walked its engine's 300 files cached by size and time. `mentions`, the POC's noisiest kind, and `inherits` and `implements` wait until someone asks for them (C00).
- **B. The POC's 13 kinds at once.** Cons: kinds with nothing to say here (capabilities, the ABI stamp, clones), and the most numerous, mentions, drown the rest.

### 3. Turning relations on and off (item 7)

- **A, pick. A param of the graph, `relations`, its layers on, as `flow,structure,code,build,docs`**, and a row of buttons along the graph's bottom edge, a layer each, lit while on, whose press sends `param set … relations …`: so a save keeps the choice and a replay restores it (V08), as the modes part's `mode`. No new command (V06). Recipes only, about 60 lines, with the `list` and `hit` parts the panel uses.
- **B. A command, `relations [all|none|<layer>...]`**, as the POC's and D20. Cons: a command for what a param already carries.

### 4. What a box shows (items 4 and 6)

- **A, pick.**
  - **A recipe's files**: a node that uses a library recipe as it is lists the recipe's files, from the library, under a row naming the recipe, and a press opens them, so the cli's and the ide's nodes show their code (item 4). The graph reads the view as it runs, each recipe unfolded: an engine read of any view by name as `Cook::view()` reads the node's own, about 10 lines.
  - **Ports**: a row a port, those it reads at the left edge and those it writes at the right, so a connection lands on the port it joins, not on the name.
  - Recipes about 120 lines, engine about 10.
- **B. Files only, as now.** Cons: item 4 stays.

### 5. Room for curves (item 6)

- **A, pick.** The gap between columns grows with the curves that cross it, a lane each, so none run over another; a curve between nodes inside one box runs inside that box, below its rows, and the box grows to hold it; a relation with no port leaves its box's bottom edge, as the POC's, labeled with what it reaches. Recipes only, about 100 lines.
- **B. As now**, a lane below each column passed. Cons: the crowding the lead saw.

## Order

| Step | Decisions | What runs after it |
| --- | --- | --- |
| 1 | 4 | a box lists a recipe's files and the node's ports; the cli's and the ide's nodes show their code |
| 2 | 5 | room for curves, and curves inside a box |
| 3 | 1 | `open <folder>` shows a folder in the graph: `open src/baseclasses` |
| 4 | 2 | the `relations` part: includes, imports, CMake, links and ids, drawn by layer |
| 5 | 3 | the layer buttons |

About 750 lines of recipes and 10 of engine, ±50%.

The lead accepted decisions 1 to 5, each with its pick, on 2026-10-16 ("all 20 handoff questions accepted").

- **Step 1, built** (2026-10-16): `frame.unfolded(name)` reads any view as the next frame runs it, as `frame.view()` reads a node's own, and a recipe's own node keeps the recipe's name once unfolded, so the graph sees which recipe a node's code came from. The graph builds its boxes from the view so read: a box a node, its name, then a row a port, a port it reads at the left and one it writes at the right, then a row naming the recipe it uses, then its files, each row of a file or a recipe opening it; and a curve a reader, from port to port, each port marked where its curves land. `view switch ide` shows the `ide` whole, its 16 nodes in its box, each with the code its recipe brings. Calls made while building it:
  - **A node inside another shows inside its box**, as its folder is inside that node's (RV08), the nodes a recipe brings too: decision 4's "a recipe's files" read with decision 5's "a curve between nodes inside one box runs inside that box … and the box grows to hold it" and decision 1's "nested as nodes nest". So a component's box holds the boxes of its parts, each listing its own files, rather than one list of every file under the component's row. A box holding boxes is framed below its rows, not filled, so the curves inside it show.
  - **Ports come first, under the name, read and written side by side on one row**, then the recipe, then the files: the plan said a row a port, and side by side halves the rows a box with both needs.
  - **The recipe's row reads `recipe <name>`, in the muted color, and a press on it opens the recipe's `view.vlp`**, which no box listed before: a recipe's own node lists no `view.vlp` among its files.
  - **Columns are found at each level**: the boxes inside a box, and those at the top, go a column past every box beside them that they read from, a box reading from what lies inside another as from that one. A connection closing a cycle at a level orders no column, found by a walk from each box in the manifest's order: the `ide`'s panels and their contents read from each other, the content its room and the panel its tabs, which had sent the boxes' columns far apart.
  - **The engine keeps a recipe's name on its unfolded node**, 4 lines beside the read's 13, which the plan's "about 10" did not count: unfolded, a node had lost which recipe it became.
  - **A box a recipe brings is part of its user's box**: a press on it selects the user's box, a drag moves that box, and it has no place of its own in the `.ini`. A box inside another is placed from where that box's inner boxes start, and a drag keeps it inside, which grows to hold it.
  - **Until step 2, a curve runs straight from port to port**, without the lane below each column it passed, which the graph had: step 2 brings lanes back, in the gaps and below the columns.
- **Step 2, built** (2026-10-17): a curve runs at right angles, rounded at each corner, and over no box and no other connection's lane. At each level it passes, the box holding both its ends or the top, it turns only in a lane of a gap between two columns and passes a column only along a track of the channel above the columns or below them; each gap is as wide as a column's gap and its lanes, a lane a connection turning there, and each channel as tall as its tracks, so the `ide`'s box grows to hold them. `view switch ide` shows every curve of the `ide` so, its panels' tabs reading back from their contents included. Calls made while building it:
  - **A curve crosses a box's edge once a connection**: it leaves a box holding its writer at the box's right edge and enters a box holding its reader at its left, and inside each it keeps to that box's lanes, so a palette entering a panel once reaches the panel's `rects` and `glyphs` from there. Decision 5's "a curve between nodes inside one box runs inside that box" read for a curve with only one end inside: before, the line from a port inside a box to its edge ran straight over the boxes beside it, as the `dock`'s `split.first` over its `rects`.
  - **A connection running back, as one closing a cycle, takes a track** as one passing a column does, from the gap after its writer to the gap before its reader, rather than a bend across the boxes between; so the gap before the first column and the one after the last hold lanes too, a lane either side of them.
  - **Its track lies in the channel nearer where it leaves**, above the columns or below them, and two connections share a track where the gaps they span do not meet. A gap's lanes are ordered to cross the fewest: those running back nearest the column before, then those passing on, one turning up the further left the higher it comes from, one turning down the further right, then those reached from a track at the right.
  - **Boxes a person placed keep lanes**, where the work in progress had dropped them for every curve of a level once a box there was dragged, and the first drag places every box of the view: the columns the curves keep to are then the runs of boxes whose widths meet, left to right, and the gaps between them no longer widen, so lanes draw closer where a gap is narrow, and the tracks above the boxes where a box sits near the top of the one holding it. Right after a first drag every curve keeps its way; only the dragged box's change.
  - **The `Curve` contract gives its two handles**, `leave` and `arrive`, so a line and a quarter circle are both a curve and `curves` draws each as before; it no longer works out a wire's bend itself. A corner's radius is half a lane, so two lanes side by side turn apart. Its room grows from 1,024 curves to 8,192, a line and a corner each of a curve's way.
  - **A writer holding its reader, or the other way round, bends straight from port to port**, as there is no level holding both to turn at.
  - **A relation with no port leaving its box's bottom edge** waits for step 4, which brings the first relation (C00).
  - **`Layout.h` runs a little past 800 lines**, saying why at its top (CPP12): the boxes and the curves' ways are worked out together, a level at a time.
- **A level at a time, built** (2026-10-17), the lead's note of that morning, item 2: "we have lost the zooming into nodes ability the POC had. Now I see a lot of nodes in on screen, but are they in the same folder/nested node? In the POC we showed a node containing nodes, and zooming into the node would show its internals." The POC showed one scope at a time, entered by zooming in on a node or a double click and left by zooming out, with a portal where a connection reached past the scope (`RuntimeCanvasCamera.cpp`, `RuntimeInputCanvas.cpp`). So the graph shows the nodes at the top, or the inside of one node, that node's box framing the boxes of the nodes inside it, and beside it a box `outside` at either side whose rows name where a connection reaching in or out goes. Calls made while building it:
  - **Step 1's nesting holds one level at a time**: a box holding nodes lists them, a row `node <name>` each, and shows them inside its frame once the graph shows its inside, rather than every level nested at once, which decision 1 of the last handoff kept.
  - **A box holding nodes shows the ports its insides read and write across its edge as its own**, named from it, as `split.area` on a dock's box, so a component shows what it takes and gives, and the curves at its level land there.
  - **The node shown is the graph's `inside` param**, which a second press or the wheel sets with a line (`ide: param set ide.graph.graph inside ide.dock`), a setting rather than a command of the graph's own, as a split's `ratio` and the text's `file` are: a replay restores it (V08), and a save of the `ide` keeps it, as it keeps the seam. With none, or one the view shown does not hold, the graph shows the top, so a switch to another view and back finds the same inside.
  - **What lies outside is a box at either side**, `outside`, a row a connection reaching in or out, named whole, as `ide.palette.palette`, rather than the POC's portal rings: it reuses the boxes, ports and lanes, and a curve crosses the frame's edge as step 2 has it.
  - **Zooming in to three times as near as home, or as near as it goes, shows the inside of the box under the pointer; zooming out past half as near as home, or as far as it goes, the level above**, the POC's thresholds as this graph's zoom counts them; a second press on the frame goes up too. Each level shown starts at home.
  - **A view's top shows its top nodes**, even a lone one, as the `ide`'s and the `cli`'s, rather than going into it at once; a second press or the wheel goes in.
- **Step 3, built** (2026-10-17): `open <folder>`, while the `ide` runs, shows the folder in the graph, as decision 1 has it: the `editor` part's `graph` param names the graph, `ide:ide.graph.graph` in the CLI's manifest, as its `window` param names the editor, and `open` sets the graph's `folder` param for a folder, as it sets the editor's `file` for a file. The graph reads the folder as a view's nodes, a node a folder, and shows it a level at a time as it shows nodes. `open src/baseclasses` shows the engine's sources, a row a file, and `examples` a box listing `triangle/`; `open src/runtime` its sources, `cmake` and `recipes`, whose inside lists `cli`, `ide` and `library`. Calls made while building it:
  - **A level at a time, as the nodes now show**, rather than every folder nested at once: a folder's box frames a box each folder in it, which lists its files and its folders, as `triangle/`, and a second press or the wheel goes in or out, as with nodes.
  - **A folder opened shows its inside at once**, rather than a lone box at a top, since `open` names the folder to look at; zooming out of it, or a second press on its frame, shows the view again, as switching views does, which takes the `folder` param back.
  - **A folder is read as its level shows**: the folder shown, the folders in it with their files, and theirs by name; a file added since shows once the level is shown again, and step 4's relations will need a reading kept up to date anyway. Hidden files and folders, as `.git` and `.vulpen`, are left out, as a manifest leaves them.
  - **Folders are no nodes**: a press selects none, a drag moves none, and none is placed in an `.ini`, which a folder has none of; their kinds' colors are the view's.
  - **A path holding a blank cannot be opened in the graph**, as a line splits its words at blanks, with no quoting; it was so for `open` of any path already.
- **Step 4, built** (2026-10-17): the `relations` part, as decision 2 has it, in the `graph-editor` component, its output to the graph's new `relations` input, as the `Relation` contract, now a C++ header, since a relation is two paths between C++ nodes. It reads the files of the view the graph shows, or of the folder its `folder` param names, through the file port, and finds what links them, line by line, no compiler asked; the graph draws each relation as a curve in its layer's hue, kept to the lanes as a connection is. `open src/runtime` shows each `.cpp` a column before the headers it includes, `Operator.h` last, with what it cites and includes past the folder in the box `outside`; the `ide`'s inside shows each node's includes of the contracts and the shared code, and the principles its comments cite, joining a track a layer to `outside`. Calls made while building it:
  - **The relations part holds the folder shown**, as its `folder` param, which `open` sets through the `editor` part's `folders` param (step 3's `graph`, renamed as it no longer names the graph), and hands the graph the folder, as a view's nodes, with the relations: one owner of what is shown (C02), and no connection from the graph back to it, which would close a cycle. The graph no longer reads the folder itself.
  - **The kinds**: code, a quoted `#include` in C++ and GLSL, reached from the file's folder or a folder holding it, as an include path reaches, a Python `import` and a shell script's `source`; build, a CMake word naming a file or a folder, past the variables before it; docs, a Markdown link to a file, and an id a document of the part's `docs` param's folder, `docs/architecture/`, defines, cited anywhere; structure, a manifest's `recipe =`. Includes in angle brackets, the system's and the libraries', are left out.
  - **Read again as what is shown changes, and every two seconds only while a reading stays under a megabyte**: the file port can say a file's text changed only for files a node opened while it bound, so `src/`, with its libraries, is read again only as it is shown anew. A write time on the file port, an engine change, would let any folder be read again only as it changes.
  - **A file of the folder whose inside shows is a box of its own**, as the POC showed it, so the relations between its files show, and its name opens it; and **relations order columns** as connections do, a file a column before what it includes.
  - **A relation ends on a box's name**, not on a file's row, and **a box showing both ends holds it**, as each box using one recipe holds the recipe's own includes, which had joined every box using that recipe to every other.
  - **A box's relations of a layer share their lanes, and every box's reaching out share one track a layer**, so the `ide`'s inside, whose nodes all include contracts and cite the principles, keeps three tracks for them rather than forty.
  - **A row of `outside` names its path from the folder holding the folder shown, or from the view's folder**, as `../../Operator.h` from the `ide`'s.
  - **The hues**: a connection in the accent color, structure in the directive color, code in the keyword color, build in the number color, docs in the string color, as the theme colors code.
- **Step 5, built** (2026-10-17): the graph's `relations` param names the layers shown, as decision 3 has it, `flow,structure,code,build,docs` joined by commas, or `none`, and unset every layer; a row of buttons along the graph's bottom edge, a layer each, lit while it shows, sends the line turning it off or on. In the `ide`, a press on `docs` and one on `code` left the connections alone. Calls made while building it:
  - **The buttons sit in a row of their own**, which a `bar` at the bottom of the `graph-editor` keeps, the graph taking the rest, rather than over the graph's ground, as the POC drew them, so they hide no box; the host connects its Rect to `bar.area` now. A `list` lays them out and a `hit` sends the line of the one pressed, as a panel's tabs, with `rects` and `glyphs` of their own: four nodes more in the component, and in the `ide`'s manifest the theme, the font, its atlas and its metrics to them.
  - **A layer hidden takes no lanes**: the graph lays out again without its curves, rather than leaving their room empty.
  - **Every layer shown leaves the param unset**, as it was, so a save of a view never holds the choice it started with.
  - **`Graph.cpp` runs past 800 lines**, saying why at its top (CPP12): a class in the unnamed namespace keeps its methods to one file.

## The lead's todo of 2026-10-17

The lead's note `docs/todo.md`, written once the steps above were built, asks for more of the graph, "always aim for improvement and adhere to principles and requirements", the POC a reference where needed:

- "1a. 'graph - cli - cli' view and outside boxes in graph cannot be selected or moved. Have their title bar be dragable."
- "1b. dragging inside of the transparent box area of a view (like 'graph - ide - ide' does) not work, it should as we can see the grid there."
- "1c. nodes that overlap should hide the box and text behind the top one. The one selected should move to/on top."
- "1d. nodes should be resizable by grabbing corners, minimal size has to be current size, but they should be able to be made bigger and changed proportions if desired."
- "2a. zooming into 'graph - cli' node allows to be zoomed out again. however zooming into 'graph - ide' node, only containing view.vlp(?), we cannot zoom back out again."
- "2b. For 'graph - cli' no view.vlp file to be seen (neither for traingle and wave)."
- "3. can we load baseclasses and runtime to see them as nodes too? perhaps by zooming out of start graph? In fact we want to be able to see this whole repository. It's okay that we have cli and ide loaded, but if we go out we should be able to find all our files and folders from the repository. Including the .hidden files."
- "4. hovering on a connection should show its useful details for the developer."
- "5. entering a node with files, like triangle with it's internal .cpp and frag, vert and glsl files should show the files as nodes, and their connections between them."

Recipes only, each item, as the ports a part needs are there (V05).

- **1a to 1c, built** (2026-10-17): the frame, the box of the node whose inside shows, and the boxes of what lies outside move as any box does, by a drag on their rows, and the `.ini` keeps where they were left; a drag on the frame's inside, below its rows, where the ground shows through, moves the graph, and with shift draws a select box, as on the ground; and a box shows over those beside it once a press or `select` raises it, the box holding it with it, its fill hiding the boxes behind it and each row of text cut where a box drawn after it lies over it. Checked on the `cli`'s inside, its frame dragged, then `outside` over the frame's boxes, whose rows it cut, then a drag on the frame's inside, which moved the graph; and on the `ide`'s inside, whose boxes the lead's `.ini` overlaps, each row cut at the box over it. Calls made while building it:
  - **The frame is a node of the view, so a press and `select` select it**, as at the level above, and `delete` removes it, which shows the level above; its place is kept by its node's name, which its box at the level above uses too: one place a node, so at the top a frame shows where its box was.
  - **What lies outside is no node**, so no line selects it: a press raises it and lets go of the selection, and a drag moves it alone, kept in the `.ini` as `<frame> outside left` or `<frame> outside right`, or `outside left` at the top, a name with a blank, which no node's holds.
  - **A box a recipe brings moves itself**, kept by its name, as each node's name is one in the view as it runs: decision 6 of the [handoff](../logs/development-20261017T060352Z.md) had a press on it select its user's box and a drag move that box, when every level showed nested; one level at a time, its user is the frame.
  - **A drag on a box nothing moves**, as a folder's, which no `.ini` keeps, **moves the graph**, as the ground does, and letting go where it was still opens its row.
  - **Only a frame's rows hide what lies under them**: a press, the wheel and a select box on its inside reach the ground, and a select box touching only its inside selects no frame.
  - **The order lasts while the view shows**, so a restart draws the boxes as built, the frame first, then each box after the box holding it: it is a look of the moment, as the pan is, which the `.ini` leaves out (C00).
  - **A row cut at the boxes over it is a Label a piece**, at most 16 a row, each holding only the characters it shows; glyphs draws every row after rects draws every box, so the graph cuts the rows rather than drawing a box at a time (A14's depth is a later row). The order and the pieces live in a header of their own, `Covered.h`.
- **1d, built** (2026-10-17): a drag on a corner of a box pulls that corner while the opposite one stays, making the box larger, or smaller down to the size its rows and the boxes inside it need, and the `.ini` keeps the size, `size <box> = width height`, in pixels at zoom 1. Checked on the `cli`'s inside: `inspect` pulled larger by its lower right corner, whose column widened and pushed the boxes under it down, and `outside` by its upper left, which moved it up and left while its lower right stayed. Calls made while building it:
  - **The corners show while the pointer is over a box**: two short arms along its edges from each corner, muted, the one under the pointer in the accent color; a press within 8 pixels of a corner, or a quarter of a small box's side, takes it.
  - **A size the box needs anyway leaves the `.ini`**, so it keeps only boxes made larger, and a box whose rows later need more grows past what was kept.
  - **A left or a top corner moves the box too**, so its first drag places every box where it shows, as a move does.
  - **At a level the graph lays out, a box made larger widens its column and pushes the boxes below it down**, as a larger box would; at one placed by hand, every other box stays where it is.
  - **A frame is a box that holds boxes**, as the layout says, rather than one taller than its rows: a box made taller is filled whole.
  - **A folder's boxes keep their size**, as they keep their places: no `.ini` holds them.
- **2a, 2b and 5, built** (2026-10-17): zooming out of the top of a view another hosts shows that view's top, through `view switch`, as entering a view's box shows its inside; a view's top shows a box for its manifest, which a press opens, and a view's box lists its top nodes before the file it is hosted from; and a node holding files is a box one goes into, as one holding nodes is, its inside showing a box a file and the relations between them. Checked on the triangle project: its node's inside showed `Triangle.cpp`, `Triangle.frag`, `Triangle.vert` and `Triangle.glsl`, the two shaders' includes of `Triangle.glsl` as curves; its top showed the node and `view.vlp`; and the wheel out of it showed the `cli`'s top, `view.vlp` there too, and the `ide`'s and the triangle's boxes listing `node ide` and `node triangle`. Calls made while building it:
  - **A view's manifest is a box at its top**, named `/view.vlp` in the `.ini`, the name of the view's own folder being empty, as a file's box inside a node is named `<node>/<file>`: a `/`, which no node's name holds. Not a frame round the top, as a node's inside has: that would hold every view's top boxes in one more box and move every place a person kept there.
  - **The manifest's lines draw no relations**, as the relations part reads the files of the view's nodes; its `recipe =` lines show already as each box's `recipe <name>` row.
  - **A view's box lists the nodes at its top, `node <name>`**, as a box holding nodes lists them, so the `ide`'s box reads `node ide` above `view.vlp`; the graph builds its boxes again as a view hosted changes.
  - **The view vulpen started with has no view above it** until item 3: zooming out of its top goes nowhere.
  - **A node's files leave its rows once its inside shows**, as a folder's did, each a box of its own; a recipe's node shows the recipe's files so.
  - **An include the build's include path finds, as `runtime/Operator.h` from a project, draws no curve**: the relations part finds a name from the file's folder and the folders holding it, as before, so `Triangle.cpp` shows none.
- **3, built** (2026-10-17), with phase 7 of the [last handoff](../logs/development-20261017T060352Z.md), the relations read a little a frame: zooming out of the top of the view vulpen started with shows the folder holding its folder, where its folder is a box, and zooming out of a folder the folder holding it, up to the top of its repository, the nearest folder holding `.git`; zooming into a folder that a view vulpen hosts is shows that view again, as `view switch` does. Hidden files and folders show, `.git`, `.vulpen` and `.gitignore` too. Checked on the `cli`: the wheel out of its top showed `Vulpen/src/runtime/recipes`, `cli` and `ide` each listing `.vulpen/` and `view.vlp`, and `library`, the manifests' recipes as curves to it; and a run on the repository's top, with the debug log, read 1,960 files and 2,251 folders, `out/` with its builds included, in 233 frames, about four seconds at 60 frames a second, finding 6,185 relations. Calls made while building it:
  - **The relations part's `folder` param names the folder whose inside shows**, a level at a time, which `open`, a second press and the wheel set, one owner of what shows (C02), where it had named the folder opened, all of which it read, and the graph's `inside` param walked the folders inside it. It reads the folder shown, the folders in it with their files, and the names of theirs, so the top of the repository shows its boxes as fast as a small folder does; the graph's `inside` is a view's alone again.
  - **Zooming out of a folder opened shows the folder holding it**, rather than the view, as decision 24 of the last handoff had it: going up stops at the top of the repository, or outside one at the folder shown, and the view shows again by zooming into its folder, by `view switch`, or by switching views in the Perform panel.
  - **Hidden files and folders show**, where decision 25 left them out, as the note asks; but **the relations part walks past hidden folders for what to read**: they hold the state of tools, `.git`'s objects and the graph's own `.ini`s, in no language it reads.
  - **Every reading is read a little a frame**: in the frame what is shown changes, as much as a small reading, a megabyte, so a small folder's curves come with its boxes; past that 64 KB a frame, about a millisecond's, a folder listed counting as 4 KB, the boxes showing at once with no curve, rather than those of what showed before, and the curves once read. Phase 7 of the last handoff, the readings again every two seconds, is so built (its decision 40).
  - **A reading reads 64 MB at most and walks 8,192 folders at most**, so its memory stays bounded (A03), where reading a tree had stopped at 2,048 folders.
  - **A reading's cost goes to the log at debug**: the files and folders read, the kilobytes, the frames and the relations found (C03, C09).
  - **`out/` is read as any folder**, the build's CMake files naming the sources included: leaving out what `.gitignore` names would take a reader of its patterns (C00).
  - **What lies past the repository's top is no relation of a folder shown**: the CMake files a build writes under `out/` name the system's own, as `/usr/bin/cc` and CMake's modules, which had filled `outside` at the top with some sixty rows; a view keeps its relations past its folder, as to the library, as before.
  - **The tab names the folder shown from the folder holding the repository's top**, as `graph - Vulpen/src/runtime/recipes`.
- **4, built** (2026-10-17): the curve under the pointer, where no box lies, shows over the others in the text's color, and a tooltip beside the pointer gives its details: a connection's name, its writer and each reader, as the manifest names them; a relation's layer and the boxes it joins, then a row each relation its curve stands for, the file, the line that says it, and what there names the other. Checked on the `cli`'s inside, a `docs` curve reading `docs: projects to ../../../../docs/architecture/requirements.md` and `../library/parts/projects/Projects.cpp:64 RP02`; and on the `ide`'s inside, the track from the terminal back to its panel reading `connection terminal-tabs`, `from ide.terminal.command-line.tabs` and its two readers. Calls made while building it:
  - **The tooltip is F7 of the [IDE port](ide-port.md), built here**: a component of `list`, `rects` and `glyphs`, which the `graph-editor` lists last, so it draws over the graph and its buttons; the graph hands it its rows as Items and the Rect they show in, and the `ide`'s manifest connects the theme, the font, its atlas and its metrics to it. Its rows come from the graph, not from `hit`'s hovered Item, which F7 planned for help on an Item: a curve is no Item.
  - **The tip shows at the font's own size**, so it reads alike at any zoom, below and right of the pointer, or above or left where that leaves the area, inside it; 16 rows at most, the last then saying how many more there are, and cut at the area's width.
  - **A `Relation` keeps the line that says it, counted from 1, and what there names the other**: an include's name, a module, a script, a recipe, a CMake word, a link's target or a cited id. A relation shows once however many lines of its file say it, at the first.
  - **A connection's tip lists every reader**, not only the one the curve reaches, as a developer asks what a connection feeds. What it carries, its contract, is no part of the view the graph reads, so the tip names ports alone.
  - **The pointer is over a curve within 4 pixels of it**, measured along its lines and along a bend in 16 pieces; of two that near, the one drawn later.

## Not in it

- **Typed paths from the current project.** Items 1 and 2 of the note, `open Triangle.cpp` in the triangle project, are a question of where a typed path counts from, which the [handoff](../logs/development-20261016T205135Z.md) put to the lead, who picked 15B; built on 2026-10-17, as the [handoff](../logs/development-20261017T060352Z.md) says: a word without a `/` names a file the view holds.
- **Editing the graph by drag**, as drawing a connection between two ports: E13's "drags that edit the graph wait for ports in the model".
