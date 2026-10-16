# Terminal app

A proposal for the lead (A00). The ask, on 2026-10-16: "./run.sh src/recipes/apps/ide/view.vlp works well, but we should also have a terminal only app, that allows us enough options to build our own ide from scratch, nothing more nothing less. … And here it is the question what is engine and what is not." Its first answer (editing by rows, a `<text>` word, `recipe publish`, raw keys) came back the same day with three remarks, and this version replaces it:

- **a)** "no text file (c++, glsl, pythong, etc) editing needed, just open in external text editor that can be configured. For us default visual studio code. Keep things minimal."
- **b)** "we need to not look further as todays cli vs ide. But we need cli to become useful outside of ide too, standalone. I think it has to do with abstraction level and who does what." "So the CLI is the master and has a child view IDE that uses the CLI but adds the interface ontop with additional functionality and quality of life imrpovements. But the basic usage of commands should be inside of the cli app." "This is an opportunity to shrink LOC and increase reusability as well as define the proper boundries."
- **c)** "the cli view and cli view with child ide view should be visible in the node graph. So that we can see their nodes in addition to being able to navigate it's structure with the terminal. Now the projects we make like triangle inside of the ide (or our cli) should be it's own view, seperated from the cli and ide."

Asked whether the cli alone is then fully capable, the lead answered the same day: "we would need tab complete and history in the os terminal", which answers decision 7; and, after first allowing one project at a time, chose that the window show the current view ("revert 1. to your previous suggestion"), which is now part of decision 1.

Then the lead added, the same day: flags to start with, `./run.sh --cli` and `--ide`, the default "for now, but should be easy to switch to --cli"; a new folder layout; and `projects/`, "the folder where we work from", where new views land, a view from elsewhere is copied in, each runs by a flag of its own, and a view is saved as another or renamed. Its two questions are answered under [folders](#folders).

Nothing here is built. Each decision says what it means, its pros and cons, whether the engine or only recipes change, and a pick. Reply like "1A 2A 3A …".

## The idea

- **The cli is the app.** It is the view vulpen starts with, and it holds every command a person needs. It runs alone at a terminal, with Tab and history there (decision 7), headless until a project draws (V07), on any machine Vulpen builds for (C10).
- **The ide is a view the cli hosts.** It adds the window: the graph, the Perform panel, a terminal in the window, keys, modes and zoom. It changes views only through commands, as every front end does (V06), so closing it costs no action, and a GUI of one's own, or this ide changed to taste, can take its place.
- **A project is a view of its own**, in `projects/`, which the cli hosts beside the ide, never inside the ide or among the cli's nodes (V03); views nested in it work as today. The cli keeps it, with a window or without; the ide shows it.

```text
cli           the view vulpen starts with: ./run.sh --cli
├── ide       the window, hosted by the cli: ./run.sh --ide, the default (decisions 1 and 6)
├── project1  a project: view new project1, in projects/project1/
└── wave      another, beside it: view load src/runtime/examples/wave
```

In C08's terms, the model is the views the engine runs, with their files; the controller is the command port with the cli's commands; and the terminal, the graph and the Perform panel are views onto the model. Text files are written in the person's own editor, which `open` starts, VS Code unless set otherwise; Vulpen keeps no editor (decisions 2 and 3).

## Starting

```text
./run.sh --cli        the cli alone: vulpen src/runtime/recipes/cli/view.vlp
./run.sh --ide        the cli hosting the ide (decision 6); ./run.sh alone does the same, for now
./run.sh --project1   the project alone, as the view vulpen starts with (decision 10)
```

The default is one line at the top of `run.sh` and of `run.ps1`, `default=--ide`, so switching to `--cli` is one word. The flags are the scripts' own, laid over vulpen's arguments, so the engine knows no app or project by name (V05); `--release` and vulpen's options (`--log`, `--frames` …) work as now. Tooling only (RC00): about 15 lines a script.

## Folders

```text
src/baseclasses/                    the GPU layer: no views, no commands
src/baseclasses/examples/triangle/  a triangle from the baseclasses alone (new)
src/external-libraries/
src/runtime/                        the runtime: views, commands, ports
src/runtime/examples/               views the runtime runs, as wave and triangle (from src/examples/)
src/runtime/recipes/cli/            the cli, all of Vulpen by itself (from src/recipes/apps/cli/)
src/runtime/recipes/ide/            the ide, which the cli hosts (from src/recipes/apps/ide/)
src/runtime/recipes/library/        the recipes views copy from (from src/recipes/)
    components/  contracts/  parts/
projects/                           the projects: where the work is (new)
```

Each folder holds what builds on the one above it, so the tree shows how includes flow (A06): the baseclasses know nothing of the runtime, and the runtime nothing of its recipes, examples or projects.

- **The cli hosting the ide, in this layout: yes.** Hosting goes by a view's name and file, not by where its folder is: the cli hosts the ide with `view add ide <file>`, from any folder. The ide's folder belongs beside the cli's, not inside it, since the cli is whole without it (V03). One thing the move needs: the runtime knows an app today by its place in the library, `recipes/apps/<name>/`, and only that place lets an app's nodes use the library's recipes as they are (V11). With the apps beside the library, the runtime learns their new place: about 10 lines in `Manifest.cpp` and `nodes.cmake`.
- **Components and parts are not the same** (RV06): a part holds code, one job's C++ and shaders, and uses no other recipe; a component holds no code, only a `view.vlp` that composes parts and other components. So both stay, unless decision 8 merges them.
- **The baseclasses' triangle** is one `main.cpp` and two shaders, drawn with `Window`, `Engine`, `Shader` and `Pipeline` alone, about 80 lines, its `main()` within A04's 30. The baseclasses build as a library of their own, which vulpen and the example both link, so an include of the runtime in them fails the build (RA00).
- **The move** changes the paths in `CMakeLists.txt`, the tests, the include map's gate, the statistics, RV06's folders and the recipe map, and about 130 lines of docs, and nothing else. It lands first, as a step of its own (A05), so no later step is built twice.

## Projects

`projects/`, at the top of the repository, is where the work is: the cli and the ide make new views there, and a view from elsewhere is copied in. The build compiles each view in it, as it compiles the examples.

```text
$ ./run.sh                                 # --ide, the default
> view new project1                        # projects/project1/, now the current view
projects/project1> view save as project2   # a copy, with the edits since the last save, in
                                           # projects/project2/; work goes on in project2, and
                                           # project1 stays as it was saved
projects/project2> view rename test_project
                                           # its folder becomes projects/test_project/
projects/test_project> view load src/runtime/examples/wave
                                           # a view from elsewhere, by its path
src/runtime/examples/wave> view save as wave2
                                           # copied into projects/wave2/
$ ./run.sh --test_project                  # the project alone
```

- **Names.** A word without a `/` names a project in `projects/`, and one with a `/` is a path from where vulpen started, as now, for `view new` and `view load` alike. A name is letters, digits, `_` and `-`, so it serves as an address too (`project1: ls`), and as a flag (`--project1`). Engine: a `<view>` placeholder, which keeps a name and resolves a path, about 10 lines.
- **`view save as <name>`** writes the view, its edits included, into `projects/<name>/`: its manifest, every node's folder, its contracts and shared code, the views nested in it and its `.vulpen/` look. The cli then works on the copy, and the original stays as it was last saved. So it also brings a view from anywhere into `projects/`.
- **`view rename <name>`** saves the view, renames its folder where it is, and hosts it under the new name. Nothing inside a project names it: nodes and their C++ classes are named for themselves, and the contracts' namespace and the modules' paths come from the folder at build time, so the folder carries it all, and the views nested in it move with it.
- **Where they live.** Both are the cli's, in a part of their own, `projects`, which takes `view new` and `view load` from `library`, whose file is near its 800 lines (CPP12): about 100 lines more. Engine: the file port writes a view's manifest to another file and renames a folder (RA04), about 20 lines. The folder is named by a param of the part, from the part's own folder (RP02), so a person who works elsewhere changes one line.

## Who does what

| | What it holds | Its commands |
| --- | --- | --- |
| engine | the views and their files, the GPU, the window, and the general ports: commands, input, files, the terminal (its keys too, decision 7), graph reads, and a new one that starts a program | `quit`, `source`, `log save`, `input …`, the edits (`node add`, `node set`, `node remove`, `connect`, `disconnect`, `param set`, `param unset`, `view add`, `view close`), `view save`, `view list`, `image clear`, and `view go` (new) |
| cli | `command-line` on the terminal, `inspect`, `library`, and `editor` and `projects` (new) | `help`, `complete`, `clear`, `ls`, `info`, `delete` (from the graph), `recipe list`, `recipe drop`, `recipe sync`, `node new`, `view new`, `view load`, and `open`, `view save as` and `view rename` (new) |
| ide | `palette`, `font`, `keys`, `viewport`, `modes`, the dock and its panels, `graph`, the Perform panel, and a terminal in the window | `mode`, `zoom`, `focus`, `select`, `selected` |
| project | its own nodes and their files, and the views nested in it | those its nodes register, if any |

**The ide registers only commands about what the window shows; every command that reads or changes a view or a file is the cli's or the engine's.** So the cli alone does everything the ide does, and a GUI that replaces the ide only has to show and send lines (V05, V06). A run of the cli registers 41 commands (37 now); the ide adds 9 (16 now), so a run with both has 50 (53 now): `present`, `write`, `close`, `close force`, `find` and `quit force` go, `open` and `delete` move into the cli, and `view go`, `view save as` and `view rename` are new. Once built, this rule could join V06.

## Today's cli and ide, function by function

What one of the two has today, as the lead asked (b), and what the lead asked for since: `projects/`, save as and rename.

| Function | Today | After |
| --- | --- | --- |
| projects: new, load, save, close | both, by path | the cli, by name in `projects/` or by path |
| save as, rename | neither | the cli: `view save as`, `view rename` |
| nodes, recipes, connections, params | both | the cli and the engine |
| read a view: `ls`, `info` | both | the cli |
| write C++, GLSL, Python | the ide's editor: `open`, `write`, `close`, `find` | the person's editor, which `open` starts (2, 3) |
| remove nodes with the nodes inside them | the ide: select, then `delete` | the cli: `delete <node>…`; the graph's Delete key sends `delete` (5) |
| switch projects | the ide: a Perform tab, `present` | the engine: `view go <name>`, which a Perform tab sends (4) |
| choose which project the window shows | the ide: the one presented; the cli: none, all draw over each other | both: the cli's window shows the current view, the ide's Perform panel the project presented (1, 4) |
| see the cli's and the ide's own nodes | the terminal: `: ls` | the terminal and the graph, after `view go` (4) |
| Tab, history, a caret | the ide's terminal | the OS terminal too (7) |
| selection, focus, modes, zoom, panels | the ide | the ide |

## A session

```text
$ ./run.sh --cli
> view new box                             # projects/box/, beside the cli, now the current view
projects/box> node new draw box            # its first files, from the template
projects/box> open                         # VS Code opens projects/box; a save there swaps in
                                           # live, and the window, open since the first draw,
                                           # shows it
projects/box> view save
projects/box> view load src/runtime/examples/triangle
                                           # another view, now current: the window shows it
src/runtime/examples/triangle> view go box # the window shows box again
projects/box> view go                      # the cli itself
> ls                                       # the cli's parts, and box/ and triangle/, its views
> view load src/runtime/recipes/ide        # the window, which presents the newest view
src/runtime/recipes/ide> ls                # the ide's own nodes, as the graph shows them now
src/runtime/recipes/ide> view go box       # the prompt, both terminals' tabs, the graph and
                                           # the Perform panel follow
projects/box> view close ide               # the window shows box again; the views run on
```

`./run.sh --ide` starts the cli with the ide hosted from the first frame (decision 6). Its window, without the editor:

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

Lines of code, estimated (±50%); today the engine has 7,994 and the recipes 5,507. "Other" is examples, scripts and CMake.

| Change | Decision | Engine | Recipes | Other |
| --- | --- | --: | --: | --: |
| the folder move; the runtime learns the apps' place | — | +10 | 0 | 0 |
| the baseclasses' triangle, and the baseclasses as a library | — | 0 | 0 | +95 |
| `projects/`: names, `view save as`, `view rename`, a flag a project | 9, 10 | +30 | +100 | +30 |
| `open` on a process port, and the cli's `editor` part | 2 | +40 | +40 | 0 |
| the ide's editor and the quit guard go | 3 | −40 | −900 | 0 |
| the current view: `view go`; `present` and the parts' own following go | 4 | +30 | −120 | 0 |
| `delete` moves into the cli | 5 | 0 | 0 | 0 |
| the ide under the cli, and the window's owner | 1 | +85 | −10 | 0 |
| `--cli` and `--ide`; vulpen hosts the views named after the first | 6 | +15 | 0 | +30 |
| Tab, history and the arrows at the OS terminal | 7 | +150 | +50 | 0 |
| **in all** | | **about +320** | **about −840** | **about +155** |

So the tree loses about 365 lines, and the ide's test the editor's run, about 80 more. The editor's removal pays for the rest: the terminal's keys, which decision 7 asks for, are most of what the engine gains, and `projects/`, the flags and the baseclasses' triangle are new.

## Decisions

### 1. Where the ide and the projects run, and what the window shows

Today a view shows only the views it hosts: the `ide`, as the view vulpen starts with, hosts the projects it presents. With the cli as that view, the ide and the projects are hosted side by side, and the engine must let one view show another beside it, and decide which view the window shows.

- **A. The cli hosts the ide and every project, side by side.** The ide presents a project beside it, as it presents one it hosts today. The window keeps one owner (A01):
  - a view that presents a view beside it, or has presented one, keeps the window until it closes: so the ide keeps it while it presents nothing, as after `view close` of the project it showed;
  - with none, the current view (decision 4), or the project holding it, while it draws: so in the cli alone, `view go` switches what the window shows;
  - else the view hosted first that draws.

  A view starts drawing the frame after it is hosted, so the ide presents a new project before it ever shows. A view the window does not show draws nowhere and keeps running, as today; views nested in a project draw as today. Engine, about 85 lines: that owner; a connection from the window of a view beside its own; a project's work run before that of the view showing it; a node that reads any view by name, knows its own view's name, since the ide no longer is the view vulpen starts with, and knows whether the window is open; and a save that leaves out a connection from the window of a view beside the one saved, since a manifest names only what is inside it (V03).
  - Pros: the projects are the cli's, with a window or without: `view close ide` leaves them running, the current one shown, and the ide, loaded again, presents them; `view new` and `view load` never ask whether a window runs, so the cli knows nothing of the ide (C08); in the cli alone, one command chooses the project worked on and shown (b); a GUI of one's own presents them the same way, and the graph walks the cli, the ide and the projects as one tree of views (V01).
  - Cons: the largest engine change here, and a structural one (A00).
- **B. The ide hosts the projects**, and the cli hosts the ide. Less engine change for the window.
  - Cons: the cli must know whether an ide runs, to host a project in it, so the controller depends on its view; `view close ide` closes the projects too; a project loaded before the ide stays the cli's, and the ide cannot show it; the projects sit inside the ide, not beside it.
- **C. The ide as nodes of the cli's view**: the `ide` app uses the `cli` app as a recipe and adds the window, as it adds the cli's parts today, and RV06 lets an app use an app. No engine change for the ide's window.
  - Cons: the ide is no view of its own: it neither loads nor closes as one, and its nodes share the cli's manifest; the cli no longer has the ide as a child, the ide has the cli.
- Pick: **A**, the boundary the remarks ask for; C is the cheaper fallback, by most of those 85 lines.

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

- **A. The command port keeps the current view**, where a line naming none goes. `view go <name>` moves it there, and `view go` alone to the cli; `view new` and `view load` move it to the view they host; `view close` of it, or of the view holding it, moves it to the closed view's host. The prompt in both terminals, the window terminal's tab and the graph follow it: the graph shows the cli's own nodes after `view go`, the ide's after `view go ide`, a nested view's after `view go <name>`, and a double click on a view's box goes into it. The Perform panel presents the current view while it is a project, keeps the one it presents while the current view is the cli or the ide, and when the ide starts, presents the newest project; once the project it shows closes, it shows nothing, as the lead chose on 2026-10-15. So a Perform tab sends `view go <name>`, and `present` goes. `view go` changes no view, so the log keeps no line of it, and each line it keeps still names its view (V08). Engine, about 30 lines; recipes, about 120 fewer: the shared `Presented`, `present`, and the following that the command-line and the graph each do.
  - Pros: one place says which view is worked on (C02), alike for both terminals, the graph, the window, scripts and agents; the cli and the ide are walked through as any project is, in the terminal and in the graph (V01); one command still switches every panel, as 1A did.
  - Cons: an engine change to the command port (A00); it revises 1A, built on 2026-10-16: `view go` in place of `present`; and `view close` of the current view goes back to the cli, not to the project before, as the cli's does now.
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

### 6. What `--ide` runs

The lead chose the flags ([starting](#starting)). Behind `--ide`, the cli starts and hosts the ide.

- **A. vulpen hosts the views named after the first**, each named for its folder, as `view load` names it: `--ide` runs `vulpen …/cli/view.vlp …/ide/view.vlp`, and `--ide --project1` adds the project. The log keeps each as a `view add`, so a replay hosts them (V08). Engine, about 15 lines in the runtime's arguments.
  - Pros: the ide, hosted first, has the window from the first frame; any view starts beside the cli the same way.
  - Cons: none known.
- **B. `--ide` hands vulpen a one-line script**, `view load src/runtime/recipes/ide`, through `--source`. No engine change.
  - Cons: a file of one line beside the scripts, and a second for a project started with the ide.
- Pick: **A**.

### 7. Typing at the terminal

Answered by the lead on 2026-10-16: "we would need tab complete and history in the os terminal."

- **A, chosen: raw keys.** The platform files put the terminal in raw mode where a person types, and give it back as they found it when the run ends (RA01). The terminal port hands its keys on, and the cli's `command-line` edits the line with the code its window face has: Tab, history, the arrows and a caret. A piped script reads lines as now, and a run in the background still reads nothing. Engine about 150 lines, recipes about 50; T41, the port's thread under TSan, comes with it.
  - Cons: escape sequences that differ between terminals, and the Windows console, to test; a run killed outright (`kill -9`) cannot give the terminal back, which `reset` then does.

### 8. Components and parts

The lead asked whether they are the same; they are not ([folders](#folders)).

- **A. Both stay**: the folder says whether a recipe holds code, and the rule each kind keeps is checked by its place (RV06).
  - Pros: a person sees where the code is (A00); the recipe map keeps its two tables.
  - Cons: two words for recipes.
- **B. One `components/` folder**: a component either holds code and uses nothing, or uses others and holds no code. Recipes and docs only: RV06 rewritten, the two folders merged, and a check that reads a recipe's content instead of its place.
  - Pros: one word fewer.
  - Cons: the folder no longer shows where the code is.
- Pick: **A**.

### 9. Projects and git

- **A. `projects/` is ignored by git**, but for a README that keeps the folder there. Each person's work stays theirs, and a project can be a git repository of its own inside it.
  - Pros: nobody's experiments in Vulpen's history; a project worth sharing becomes an example in `src/runtime/examples/`, or recipes in the library.
  - Cons: Vulpen's repository keeps no copy of a project.
- **B. Tracked**: the projects are part of Vulpen's history.
  - Cons: every person's work in one history, and every build compiles everyone's projects.
- Pick: **A**.

### 10. What `./run.sh --project1` runs

- **A. The project alone**, as the view vulpen starts with, as `./run.sh src/examples/wave/view.vlp` runs a view now: no commands typed, its window, or headless (V07). With `--cli` or `--ide` before it, the tool with the project loaded.
  - Pros: the project as it would ship, in one word; the tool with it in two.
  - Cons: none known.
- **B. The ide with the project loaded.**
  - Cons: running the project alone then takes its path.
- Pick: **A**.

## Smaller calls

Made in this proposal; each is the lead's to undo.

- **A view that has presented keeps the window until it closes**, not only while it presents (decision 1): the frame where the ide presents nothing, as after a close, would else hand the window to a project, and the lead's choice of an empty Perform panel after a close would need undoing.
- **A nested view shows through its project**: `view go inner`, for a view a project hosts, shows that project, in the window and in the Perform panel, as a nested view shows through its host ([views and modes](views-and-modes.md), 3.2A).
- **The terminal in the window registers no command.** `help`, `complete` and `clear` are the cli's, since a name has one owner (B5). Tab sends `complete` as now; `clear` answers the text that clears a terminal, which the window's terminal takes as a terminal does; and `control+l` sends `clear` from that terminal, not from the keymap, so the answer reaches it.
- **Only the cli reads the terminal vulpen was started from.** That terminal still takes lines while the ide runs, as the lead chose on 2026-10-13; the ide's terminal no longer reads it too.
- **The end of standard input ends the run only while no window is open**, so a launcher that gives none still starts the ide, and a piped script still ends a run with no window. A node then reads whether the window is open, a few engine lines, counted in decision 1.
- **A node reads any view by its name**, as a command does, where now it reads only the views its own view hosts: the graph shows the cli, and the Perform panel's tabs list the projects beside the ide. Reads change nothing, so views still change only by commands (V06).
- **A bare name means `projects/`**: `view new zz-demo` made `./zz-demo/` until now, and makes `projects/zz-demo/`; a path keeps its meaning.
- **A project named like a flag** of the scripts or of vulpen (`cli`, `ide`, `release`, `log` …) runs by its path, since the flags come first.
- **`view new` and `view load` move to the new `projects` part**, beside `view save as` and `view rename`, so `library` keeps to recipes and nodes.

## Order

Each step is a commit that runs, checked against the principles and requirements again as it lands.

| Step | Decisions | What runs after it |
| --- | --- | --- |
| 1 | 8 | the folder move, as a step of its own (A05); `./run.sh --cli` and `--ide`, the default, each still running its app as the view vulpen starts with |
| 2 | — | the baseclasses' triangle, and the baseclasses built as a library of their own |
| 3 | 9, 10 | `projects/`: `view new` and `view load` by name, `view save as`, `view rename`, and `./run.sh --<project>` |
| 4 | 2, 3 | `open` starts VS Code, from the cli and from a press in the graph; the ide's editor and the quit guard are gone |
| 5 | 7 | Tab, history and the arrows at the OS terminal |
| 6 | 4 | `view go`: both terminals, the graph and the Perform panel follow one current view, the ide's own included; `present` is gone |
| 7 | 5 | `delete` in the cli; the graph's Delete key sends it |
| 8 | 1 | the engine half of 1A, as a step of its own (A05): a view presents a view beside it, the window's owner, a view draws from the frame after it is hosted, a node reads any view; in the cli alone, `view go` now switches the window; the ide is still the view vulpen starts with |
| 9 | 1, 6 | the ide under the cli: it loses `inspect`, `library`, `editor` and `projects`, which are the cli's, and the terminal it read; `./run.sh --ide` starts the cli hosting it, and the graph shows the cli and the ide |

The ide's test, `ide.py`, follows each step: step 1 moves its paths, step 4 drops its editor run, and step 9 starts it as the cli with the ide.

## Not in it

- **`recipe publish`** (D17): neither app has it today, so it waits for a plan of its own (b).
- **Editing by rows, the `<text>` word, row edits in the log**: no text is edited in Vulpen (a).
- **Commands per view** (6A of the first version): each command has one owner, the engine, the cli or the ide, so no view needs commands of its own.
- **Renaming a node with its view**: `view rename` renames the view's folder only; a node named like its project keeps its name.
