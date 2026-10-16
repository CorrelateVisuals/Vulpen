# Terminal app

A proposal for the lead (A00). The ask, on 2026-10-16: "./run.sh src/recipes/apps/ide/view.vlp works well, but we should also have a terminal only app, that allows us enough options to build our own ide from scratch, nothing more nothing less. … And here it is the question what is engine and what is not." Its first answer (editing by rows, a `<text>` word, `recipe publish`, raw keys) came back the same day with three remarks, and this version replaces it:

- **a)** "no text file (c++, glsl, pythong, etc) editing needed, just open in external text editor that can be configured. For us default visual studio code. Keep things minimal."
- **b)** "we need to not look further as todays cli vs ide. But we need cli to become useful outside of ide too, standalone. I think it has to do with abstraction level and who does what." "So the CLI is the master and has a child view IDE that uses the CLI but adds the interface ontop with additional functionality and quality of life imrpovements. But the basic usage of commands should be inside of the cli app." "This is an opportunity to shrink LOC and increase reusability as well as define the proper boundries."
- **c)** "the cli view and cli view with child ide view should be visible in the node graph. So that we can see their nodes in addition to being able to navigate it's structure with the terminal. Now the projects we make like triangle inside of the ide (or our cli) should be it's own view, seperated from the cli and ide."

Nothing here is built. Each decision says what it means, its pros and cons, whether the engine or only recipes change, and a pick. Reply like "1A 2A 3A …".

## The idea

- **The cli is the app.** It is the view vulpen starts with, and it holds every command a person needs. It runs alone at a terminal, headless until a project draws (V07), on any machine Vulpen builds for (C10).
- **The ide is a view the cli hosts.** It adds the window: the graph, the Perform panel, a terminal in the window, keys, modes and zoom. It changes views only through commands, as every front end does (V06), so closing it costs no action, and a GUI of one's own, or this ide changed to taste, can take its place.
- **A project is a view of its own**, which the cli hosts beside the ide, never inside the ide or among the cli's nodes (V03). The cli keeps it, with a window or without; the ide shows it.

```text
cli           the view vulpen starts with: ./run.sh src/recipes/apps/cli/view.vlp
├── ide       the window, hosted by the cli (decisions 1 and 6)
├── triangle  a project: view load src/examples/triangle
└── wave      another, beside it
```

In C08's terms, the model is the views the engine runs, with their files; the controller is the command port with the cli's commands; and the terminal, the graph and the Perform panel are views onto the model. Text files are written in the person's own editor, which `open` starts, VS Code unless set otherwise; Vulpen keeps no editor (decisions 2 and 3).

## Who does what

| | What it holds | Its commands |
| --- | --- | --- |
| engine | the views and their files, the GPU, the window, and the general ports: commands, input, files, the terminal, graph reads, and a new one that starts a program | `quit`, `source`, `log save`, `input …`, the edits (`node add`, `node set`, `node remove`, `connect`, `disconnect`, `param set`, `param unset`, `view add`, `view close`), `view save`, `view list`, `image clear`, and `view go` (new) |
| cli | `command-line` on the terminal, `inspect`, `library`, and `editor` (new) | `help`, `complete`, `clear`, `ls`, `info`, `delete` (from the graph), `recipe list`, `recipe drop`, `recipe sync`, `node new`, `view new`, `view load`, and `open` (new) |
| ide | `palette`, `font`, `keys`, `viewport`, `modes`, the dock and its panels, `graph`, the Perform panel, and a terminal in the window | `mode`, `zoom`, `focus`, `select`, `selected` |
| project | its own nodes and their files | those its nodes register, if any |

**The ide registers only commands about what the window shows; every command that reads or changes a view or a file is the cli's or the engine's.** So the cli alone does everything the ide does, and a GUI that replaces the ide only has to show and send lines (V05, V06). A run of the cli registers 39 commands (37 now); the ide adds 9 (16 now), so a run with both has 48 (53 now): `present`, `write`, `close`, `close force`, `find` and `quit force` go, `open` and `delete` move into the cli, and `view go` is new. Once built, this rule could join V06.

## Today's cli and ide, function by function

Only what one of the two has today, as the lead asked (b): nothing both lack is added.

| Function | Today | After |
| --- | --- | --- |
| projects: new, load, save, close | both | the cli |
| nodes, recipes, connections, params | both | the cli and the engine |
| read a view: `ls`, `info` | both | the cli |
| write C++, GLSL, Python | the ide's editor: `open`, `write`, `close`, `find` | the person's editor, which `open` starts (2, 3) |
| remove nodes with the nodes inside them | the ide: select, then `delete` | the cli: `delete <node>…`; the graph's Delete key sends `delete` (5) |
| switch projects | the ide: a Perform tab, `present` | the engine: `view go <name>`, which a Perform tab sends (4) |
| see the cli's and the ide's own nodes | the terminal: `: ls` | the terminal and the graph, after `view go` (4) |
| see a project run | both: the window | both |
| Tab, history, a caret | the ide's terminal | the OS terminal too, last (7) |
| selection, focus, modes, zoom, panels | the ide | the ide |

## A session

```text
$ ./run.sh src/recipes/apps/cli/view.vlp
> view new src/examples/box                # a project beside the cli, now the current view
src/examples/box> node new draw box        # its first files, from the template
src/examples/box> open                     # VS Code opens src/examples/box; a save there swaps
                                           # in live, and the window, open since the first
                                           # draw, shows it
src/examples/box> view save
src/examples/box> view go                  # the cli itself
> ls                                       # the cli's parts, and box/, a view it hosts
> view load src/recipes/apps/ide           # the window, which presents box
src/recipes/apps/ide> ls                   # the ide's own nodes, as the graph shows them now
src/recipes/apps/ide> view go box          # the prompt, both terminals' tabs, the graph and
                                           # the Perform panel follow
src/examples/box> view close ide           # the window is box's again; box and the cli run on
```

With the ide from the start: `./run.sh src/recipes/apps/cli/view.vlp src/recipes/apps/ide/view.vlp` (decision 6). Its window, without the editor:

```text
+--------------------------+--------------------------+
| graph - box              | box                      |
|  the current view        |  the Perform panel       |
+--------------------------+--------------------------+
| terminal - box                                      |
|  the log, and the line typed                        |
+-----------------------------------------------------+
```

## What changes, in lines

Lines of code, estimated (±50%); today the engine has 7,994 and the recipes 5,507.

| Change | Decision | Engine | Recipes |
| --- | --- | --: | --: |
| `open` on a process port, and the cli's `editor` part | 2 | +40 | +40 |
| the ide's editor and the quit guard go | 3 | −40 | −900 |
| the current view: `view go`; `present` and the parts' own following go | 4 | +30 | −120 |
| `delete` moves into the cli | 5 | 0 | 0 |
| the ide under the cli | 1 | +75 | −10 |
| vulpen hosts the views named after the first | 6 | +15 | 0 |
| **without 7** | | **about +120** | **about −990** |
| raw keys at the OS terminal | 7 | +150 | +50 |

So the tree loses about 870 lines, and the ide's test the editor's run, about 80 more.

## Decisions

### 1. Where the ide and the projects run

Today a view shows only the views it hosts: the `ide`, as the view vulpen starts with, hosts the projects it presents. With the cli as that view, the ide and the projects are hosted side by side, and the engine must let one view show another beside it.

- **A. The cli hosts the ide and every project, side by side.** The ide presents a project beside it, as it presents one it hosts today. The window keeps one owner (A01): a view that presents another, else the view hosted first that draws, so a project the ide does not present draws nowhere, as today. Engine, about 75 lines: that owner; a connection from the window of a view beside its own; a project's work run before that of the view showing it; a node that reads any view by name, knows its own view's name, since the ide no longer is the view vulpen starts with, and knows whether the window is open; and a save that leaves out a connection from the window of a view beside the one saved, since a manifest names only what is inside it (V03).
  - Pros: the projects are the cli's, with a window or without: `view close ide` leaves them running, the first of them drawing into the window again, and the ide, loaded again, presents them; `view new` and `view load` never ask whether a window runs, so the cli knows nothing of the ide (C08); a GUI of one's own presents them the same way, and the graph walks the cli, the ide and the projects as one tree of views (V01).
  - Cons: the largest engine change here, and a structural one (A00); in the cli alone, a second project that draws, draws nowhere until the first closes, where today it draws over it.
- **B. The ide hosts the projects**, and the cli hosts the ide. No engine change for the window.
  - Cons: the cli must know whether an ide runs, to host a project in it, so the controller depends on its view; `view close ide` closes the projects too; a project loaded before the ide stays the cli's, and the ide cannot show it; the projects sit inside the ide, not beside it.
- **C. The ide as nodes of the cli's view**: the `ide` app uses the `cli` app as a recipe and adds the window, as it adds the cli's parts today, and RV06 lets an app use an app. No engine change for the window, and `./run.sh src/recipes/apps/ide/view.vlp` starts it as now.
  - Cons: the ide is no view of its own: it neither loads nor closes as one, and its nodes share the cli's manifest; the cli no longer has the ide as a child, the ide has the cli.
- Pick: **A**, the boundary the remarks ask for; C is the cheaper fallback, by about 70 engine lines.

### 2. Opening files

- **A. `open [<file>]`**, a command of the cli, starts the person's editor on a file or a folder; with none, on the current view's folder, which VS Code opens as a workspace. The program is a param of the cli's new `editor` part, `program=code`; another is a `param set` for a session, or that line of the cli's manifest. The graph's press on a file sends `open`, as now. Engine, about 40 lines: a process port, which starts a program and does not wait for it, through the platform files (RA01), and refuses a program the PATH lacks, naming it (A02). Recipes, about 40: the `editor` part, named for its job (RV07).
  - Pros: one command, any editor; a save there swaps in live, as a save from any editor does now; Vulpen keeps no editor.
  - Cons: a program that needs more words than the file, as a terminal editor in a terminal of its own, is named through a two-line script; the program sits in a file the repo tracks, so a person's own choice shows as a change.
- **B. The program from `$VISUAL` or `$EDITOR`.**
  - Cons: those mostly name a terminal editor, which would fight the cli for the terminal; and a second place to set it, beside the manifest (C02).
- Pick: **A**.

### 3. The ide's own editor

- **A. It goes.** The editor panel, the `text` part with its syntax colors (872 lines of code), `text-area` and the empty `find-bar` leave, with `write`, `close`, `close force`, `find` and `control+s`. So does the quit guard, `Cook::hold` and `quit force` (about 40 engine lines), which only the editor used (C00). The dock keeps the graph beside the Perform panel, over the terminal.
  - Pros: one way to write code, in a real editor; the largest cut (C04); `open` has one owner (B5).
  - Cons: it undoes the editor built from 2026-10-10 to 10-15 (E11, F3), and the quit guard of 10-14 (D1); nothing is written inside the window.
- **B. It stays**, beside `open`, under a name of its own, such as `edit`.
  - Cons: two ways to write code, and two commands for one job (V06); about 900 lines kept.
- Pick: **A**.

### 4. The current view

Today the terminal's lines and the graph follow the view presented, else the newest hosted ([views and modes](views-and-modes.md), 1A). The cli and the ide cannot be presented, so they are reached only by `:` and `ide:`, and the graph never shows them.

- **A. The command port keeps the current view**, where a line naming none goes. `view go <name>` moves it there, and `view go` alone to the cli; `view new` and `view load` move it to the view they host; `view close` of it, or of the view holding it, moves it to the closed view's host. Both terminals' prompts and tabs, the graph and the Perform panel follow it: the graph shows the cli's own nodes after `view go`, the ide's after `view go ide`, and a double click on a view's box goes into it. The Perform panel presents the current view while it is a project, keeps the one it presents while the current view is the cli or the ide, and with none presented, presents the newest project; so a Perform tab sends `view go <name>`, and `present` goes. `view go` changes no view, so the log keeps no line of it, and each line it keeps still names its view (V08). Engine, about 30 lines; recipes, about 120 fewer: the shared `Presented`, `present`, and the following that the command-line and the graph each do.
  - Pros: one place says which view is worked on (C02), alike for both terminals, the graph, scripts and agents; the cli and the ide are walked through as any project is, in the terminal and in the graph (V01); one command still switches every panel, as 1A did.
  - Cons: an engine change to the command port (A00). It revises 1A, built on 2026-10-16: `view go` in place of `present`. `view close` of the current view goes back to the cli, not to the project before. And once the view it presents closes, the Perform panel presents the newest project left, where the lead chose nothing on 2026-10-15: an ide presenting nothing would let a project take the window (decision 1).
- **B. As now.**
  - Cons: c) is not met: the graph shows neither the cli nor the ide.
- **C. The graph walks the tree on its own**, by presses on views' boxes.
  - Cons: the graph and the terminal follow different views again, which views and modes ended.
- Pick: **A**.

### 5. Removing nodes

- **A. `delete [<node>…]` moves into the cli**, to `inspect`, beside `info [<node>]`. It removes the nodes named, the nodes inside them and their connections, as the graph's does now, and closes the views named; with none, what the graph selects, asked with `selected`, as `info` asks. The graph keeps `select` and `selected`, which are the window's, and its Delete key sends `delete`. Recipes only: about 50 lines move.
  - Pros: the cli removes a dropped recipe in one line, where now it removes each node, the innermost first, after its connections.
  - Cons: `inspect` reads and now removes, which its name does not say.
- **B. It stays the graph's.**
  - Cons: the cli alone keeps the long way.
- Pick: **A**.

### 6. Starting the ide

`./run.sh src/recipes/apps/ide/view.vlp` would run the ide without the cli's commands, so the ide starts inside the cli.

- **A. vulpen hosts the views named after the first**, each named for its folder, as `view load` names it: `./run.sh src/recipes/apps/cli/view.vlp src/recipes/apps/ide/view.vlp`, or the cli with a project. The log keeps each as a `view add`, so a replay hosts them (V08). Engine, about 15 lines in the runtime's arguments.
  - Pros: one line starts both, and the ide, hosted first, has the window from the first frame.
  - Cons: a longer line than now.
- **B. Typed**: the cli, then `view load src/recipes/apps/ide`. Nothing to build.
  - Cons: a line typed at every start.
- Pick: **A**; B works either way.

### 7. Typing at the terminal

- **A. Raw keys, as the last step.** The platform files put the terminal in raw mode where a person types (RA01), the terminal port hands its keys on, and the cli's `command-line` edits the line with the code its window face has: Tab, history, the arrows and a caret. Engine about 150 lines, recipes about 50; T41, the port's thread under TSan, comes with it.
  - Pros: the cli alone types as the ide's terminal does.
  - Cons: escape sequences that differ between terminals, and the Windows console, to test.
- **B. Never**: `rlwrap` gives history from outside.
  - Cons: no Tab, though every command declares its completion (RV04).
- Pick: **A**.

## Smaller calls

Made in this proposal; each is the lead's to undo.

- **The terminal in the window registers no command.** `help`, `complete` and `clear` are the cli's, since a name has one owner (B5). Tab sends `complete` as now; `clear` answers the text that clears a terminal, which the window's terminal takes as a terminal does; and `control+l` sends `clear` from that terminal, not from the keymap, so the answer reaches it.
- **Only the cli reads the terminal vulpen was started from.** That terminal still takes lines while the ide runs, as the lead chose on 2026-10-13; the ide's terminal no longer reads it too.
- **The end of standard input ends the run only while no window is open**, so a launcher that gives none still starts the ide, and a piped script still ends a run with no window. A node then reads whether the window is open, a few engine lines, counted in decision 1.
- **A node reads any view by its name**, as a command does, where now it reads only the views its own view hosts: the graph shows the cli, and the Perform panel's tabs list the projects beside the ide. Reads change nothing, so views still change only by commands (V06).
- **A nested view shows through its project**: `view go inner`, for a view a project hosts, presents that project, as a nested view shows through its host ([views and modes](views-and-modes.md), 3.2A).

## Order

Each step is a commit that runs, checked against the principles and requirements again as it lands.

| Step | Decisions | What runs after it |
| --- | --- | --- |
| 1 | 2, 3 | `open` starts VS Code, from the cli and from a press in the graph; the ide's editor and the quit guard are gone |
| 2 | 4 | `view go`: both terminals, the graph and the Perform panel follow one current view, the ide's own included; `present` is gone |
| 3 | 5 | `delete` in the cli; the graph's Delete key sends it |
| 4 | 1 | the engine half of 1A, as a step of its own (A05): a view presents a view beside it, the window keeps one owner, a node reads any view; the ide is still the view vulpen starts with |
| 5 | 1, 6 | the ide under the cli: it loses `inspect`, `library` and `editor`, which are the cli's, and the terminal it read; `./run.sh …/cli/view.vlp …/ide/view.vlp` starts both, and the graph shows the cli and the ide |
| 6 | 7 | raw keys: Tab and history at the OS terminal |

The ide's test, `ide.py`, follows each step: step 1 drops its editor run, and step 5 starts it as the cli with the ide.

## Not in it

- **`recipe publish`** (D17): neither app has it today, so it waits for a plan of its own (b).
- **Editing by rows, the `<text>` word, row edits in the log**: no text is edited in Vulpen (a).
- **Commands per view** (6A of the first version): each command has one owner, the engine, the cli or the ide, so no view needs commands of its own.
- **In the cli alone, the window shows the first project that draws**, not the current view. Following `view go` there would flash a project over the ide for a frame as the ide presents its first one, so it waits until two drawing projects are worked on at once (C00).
