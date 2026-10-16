# Terminal app

A proposal for the lead (A00), asked on 2026-10-16: "./run.sh src/recipes/apps/ide/view.vlp works well, but we should also have a terminal only app, that allows us enough options to build our own ide from scratch, nothing more nothing less. Command driven where we can make nodes and edit files and present them so that we can expand the language from the language this is fractal vulpen. And here it is the question what is engine and what is not." Nothing here is built. Each decision says what it means, its pros and cons, whether the engine or only recipes change, and a pick. Reply like "1A 2A 3A …".

## The aim

A person or an agent, typing only commands at a terminal, makes a project, adds nodes, writes their C++ and GLSL, sees them run, and turns what works into library recipes that the next project uses, so the language grows from itself (V01, V02, V11). Every step is a command (V06), so a script or an agent runs the same session, and it runs headless until a node draws (V07). That is enough to build the `ide` from scratch, since the `ide` is only recipes (V05): parts of C++ and GLSL, and components and apps that compose them (RV06). The terminal app's own parts are library recipes too, so it edits them from inside itself, and a save swaps into the app as it runs (live code).

"Nothing more, nothing less" is held to one session, the loop below: the terminal app has what the loop needs, and nothing the loop does not (C00).

## The loop

A sketch of the session the terminal app must run, from an empty folder to a component in the library. The lines marked new are proposed below; the rest runs today.

```text
$ ./run.sh src/recipes/apps/cli/view.vlp
> view new src/examples/box                        # a project
src/examples/box> node new draw box                # its first files, from the template
src/examples/box> open box/Box.frag                # the editor, on the terminal (new)
src/examples/box> row show                         # its rows, numbered (new)
src/examples/box> row set 14   color = vec4(1.0, 0.5, 0.0, 1.0);   # one row as typed, blanks kept (new)
src/examples/box> write                            # saved: the live build swaps it in, and
                                                   # the window, open since the first draw, shows it
src/examples/box> view save
src/examples/box> recipe publish part box          # src/recipes/parts/box/ (new)
src/examples/box> view new src/examples/pair
src/examples/pair> node add pair                   # the component's own node, a group
src/examples/pair> recipe drop box pair.left       # the part reused, as copies
src/examples/pair> recipe drop box pair.right
src/examples/pair> recipe publish component pair   # src/recipes/components/pair/: uses box twice (new)
```

An app is published the same way, and `./run.sh src/recipes/apps/<name>/view.vlp` runs it, as the `ide` runs now.

## What the CLI has, and what the loop lacks

The `cli` app is terminal only already: `command-line` on the terminal port, `inspect` and `library` ([recipe map](../architecture/recipe-map.md)). Step by step:

| Step | Commands | Today |
| --- | --- | --- |
| a project | `view new`, `view load`, `view save`, `view close` | yes |
| nodes and wires | `node new draw\|dispatch`, `node add`, `node set`, `node remove`, `connect`, `disconnect`, `param set`, `param unset` | yes |
| read the graph | `ls`, `info` | yes |
| write C++ and GLSL | — | **no**: the editor (`open`, `write`, `close`, `find`) is the `ide`'s, in a window only, and a row of code cannot pass through a command, which splits a line at blanks |
| see it run | none needed: a hosted view draws into the window while its host draws nothing, the window opens with the first draw (B14), and a save swaps in ([live code](live-code.md)) | yes |
| a part from a node | — | **no**: `recipe publish` (D17) was never built |
| a component or an app from a project | — | **no**: the same command, its drops becoming uses |
| reuse | `recipe drop`, `recipe sync` | yes |
| replay | `source`, `log save` | yes |
| type at ease | Tab, history, the arrows | **no**: the terminal port reads whole lines, so the shell edits the line and Tab does nothing (B11) |

## Engine or not

The rule is the principles' own. The engine is the floor a headless replay needs with no recipe loaded: the edits, the log, save, hosting, input and `quit` ([migration](../architecture/migration-and-implementation.md#commands-the-engine-keeps-the-mechanism-recipes-bring-the-vocabulary)); the general ports: commands, files, input, the terminal and graph reads ([recipe map](../architecture/recipe-map.md#core-ports)); and the machinery under them: the GPU, the window, the schedule and the live build. A job a recipe can do through those ports, with no private way in, is a recipe's. A recipe that needs more gets a general port, never the feature (V05), and a new port changes the architecture first, in a step of its own the lead decides (A00, A05).

Applied to the gaps:

| Gap | Engine | Recipes |
| --- | --- | --- |
| write code | a command word that keeps its blanks (decision 3), about 25 lines | the `text` part edits by rows, on the terminal and in the window (decision 2), about 120 lines |
| publish | nothing: the file port reads, lists and saves | the `library` part (decision 4), about 120 lines |
| see it run | nothing | nothing |
| type at ease | the terminal port hands keys, in raw mode from the platform files (decision 7), about 150 lines | `command-line`'s terminal face edits the line as its window face does, about 50 lines |

So the loop needs one small engine change, and ease a second; the rest is recipes, and the `cli` app gains one node, `text`. Two engine questions stand behind it, decisions 5 and 6, and neither blocks the loop.

## Decisions

### 1. Which app

- **A. Grow the `cli` app** by addition: it gains a node of the `text` part with `param = on=terminal`, and the `library` part gains publishing. Recipes only. Pros: one terminal app, as now (C06), and RV07 keeps its name. Cons: none known.
- **B. A new app beside `cli`.** Cons: two terminal apps that differ by one node.
- Pick: **A**.

### 2. Editing files by command

- **A. The `text` part edits by rows, in both faces.** It gains a terminal face, `param = on=terminal`, as `command-line` has, which prints where the window face draws, and one group of commands on the file shown (V06): `row show [<value>...]` prints rows numbered, all of them, one, or from and to; `row set <value> <text>` replaces a row; `row add <value> <text>` adds one after it; `row remove <value> [<value>]` removes one, or from and to. `open`, `write`, `close` and `find` stay as they are. In the `ide` the same commands edit the editor's buffer, so a script or an agent edits there by text too, where today only keys do. Recipes only, about 120 lines; `Text.cpp` would pass 800 lines, so the rows go to a file of the part's own (CPP12).
  - Pros: one editor a run, and one owner of its commands (B5); an agent edits by rows, as it reads a file.
  - Cons: rows by number are slow for a person writing much code at once.
- **B. `edit <file>` runs the person's `$EDITOR`** in the terminal until it exits. Engine: a process port that hands the terminal to a program and waits (platform files, RA01).
  - Pros: a real editor.
  - Cons: the editing happens outside Vulpen (V01), an agent cannot use it, and the frames stop meanwhile (VK02).
- **C. No editing in Vulpen**: files are written in any editor beside it, and the live scan swaps a save in.
  - Pros: nothing to build.
  - Cons: not command driven, which the aim asks.
- Pick: **A**. B can come later, as a recipe on a process port, if a person wants it.

### 3. A command word that keeps its blanks

A row of code keeps its indent and its blanks, but the command port splits a line at blanks, and `input text` and `find` join their words with single blanks.

- **A. A placeholder `<text>`**, last in a usage, takes the rest of the line as typed, blanks kept, but the one after the word before it. Engine: the command port's split and its list of kinds, about 25 lines; nothing to complete. The log keeps it as one word, which a replay splits back alike, since the usage is the same (V08). `input text` and `find` take it too, so typing two blanks types two.
  - Pros: one rule, and no quoting to learn.
  - Cons: only a usage's last argument can hold blanks.
- **B. Quotes in the command language**, `"…"` with `\"` and `\\`, for any argument.
  - Pros: any argument holds blanks.
  - Cons: more rules for a person and for completion, which every placeholder kind must follow.
- Pick: **A**.

### 4. Publishing

- **A. `recipe publish part|component|app <node>`**, the kind a word of the command, as in `node new draw|dispatch` (V06). A part copies the node's folder into `recipes/parts/<name>/` with its `view.vlp`, and each contract or shared code it includes that the library lacks (RV05, RV09). A component or an app writes a `view.vlp` whose nodes use what they were dropped from: a drop left as it was, its fingerprint holding, becomes `recipe = <name>` with its params, and the connections between them stay. A drop changed since is refused, naming it, since it is no longer the library's: it is published as a part first. Refused too: a name the library has, a part that holds drops, and a component or an app with code of its own (RV06). Recipes only, the `library` part, about 120 lines; `Library.cpp` would pass 800 lines, so publishing goes to a file of the part's own (CPP12). The recipe map is kept by hand until H2, so the answer names the row it needs there.
  - Pros: the library grows by addition, from a project, by command (V01, V11).
  - Cons: a changed drop takes two publishes, the part and then its composition.
- **B. Parts only**; components and apps written by hand.
  - Cons: composing, the step the `ide` is made of, is not command driven.
- Pick: **A**.

### 5. Which view the window shows

In the terminal app every hosted view that draws, draws into the window, the newest over the older. The `ide` chooses one with `present`, through its modes part.

- **A. As now**: host the project you look at, and `view close` the others. Nothing changes.
  - Pros: nothing to build; the loop works on one project at a time.
  - Cons: two projects that draw overlap in the window.
- **B. The window of a host that draws nothing goes to its newest hosted view alone.** Engine, about 15 lines, where the views gather their draws.
  - Pros: the last `view load` shows alone, as the `ide` presents a view as it comes.
  - Cons: an engine rule for what the `ide` does in a recipe; a view hosted to run beside the one shown stops drawing.
- **C. A `present` part in the terminal app.**
  - Cons: a part that draws opens the window at load, so the app would no longer start headless (V07).
- Pick: **A**, until two drawing projects are worked on at once (C00).

### 6. A project whose parts register commands the app has

A command name has one owner in a run (B5). So a project hosted in the terminal app cannot run a part that registers `open`, `help`, `ls` or any other name the app's parts own: the project's node is left out, naming the command. Building the `ide`'s own kinds of part, an editor or a command line, is such a case; so is a finished `ide` hosted in the terminal app, or an `ide` in the `ide`.

- **A. Commands belong to a view.** A node's command answers the lines addressed to its view and to the views that view hosts, the nearest owner winning, and the engine's commands answer everywhere; `help` and `complete` list what the addressed view reaches. Engine: the command port's registry and lookup, and `usages()` by the view, about 150 lines; the parts that find a command by its name, as `inspect` finds `selected` and the keys part a keymap line's command, ask their own view.
  - Pros: views are fractal for their commands too (V01, V03): an `ide` runs inside the terminal app, or inside itself.
  - Cons: the largest engine change here, and a structural one (A00); a line's meaning then depends on the view it goes to.
- **B. As now**: such a project runs as its own process, `./run.sh <project>/view.vlp`, with the library's `command-line` dropped into it to type to; publishing and composing stay in the terminal app.
  - Pros: no engine change.
  - Cons: two processes while building such a part, and no app is hosted by an app.
- Pick: **B** for the loop, and **A** as a step of its own once the lead wants an `ide` inside an `ide`. This is where "what is engine" is open: B keeps command ownership where it is, and A moves it into the views.

### 7. Typing at the terminal

- **A. Raw keys, once the loop runs.** The platform files put the terminal in raw mode where a person types (RA01), the terminal port hands its keys on as events, as the input port does the window's, and `command-line`'s terminal face edits the line with its window face's code: Tab, history, the arrows and a caret. The B11 row says the port's interface stays and no recipe changes; Tab cannot work so, since the part that completes must see the key, so B11 changes to this. Engine about 150 lines, recipes about 50; T41, the port's thread under TSan, comes with it.
  - Pros: the terminal app types as the `ide`'s terminal does.
  - Cons: escape sequences that differ between terminals, and the Windows console, to test.
- **B. Now, before the loop.**
  - Cons: ease before what the loop needs (C00).
- **C. Never**: `rlwrap` gives history from outside.
  - Cons: no Tab, though every usage declares its completion (RV04).
- Pick: **A**.

### 8. Row edits and the log

- **A. Out of the log**, as typing in the editor is: the file on disk is the record, and a replay needs the files in their folders, as it does today (V03, V08).
  - Pros: the log stays the graph's.
  - Cons: a session's edits to files replay only from the files.
- **B. In the log**, so a replay writes the files too.
  - Cons: a row edit replays right only on the file it was made on, so a replay over other files corrupts them without a word.
- Pick: **A**.

## Order

Each step is a commit that runs, and is checked against the principles and requirements again as it lands.

| Step | Decisions | What runs after it |
| --- | --- | --- |
| 1 | 3 | `<text>`: `input text` and `find` keep blanks |
| 2 | 1, 2, 8 | the `cli` opens a file, shows it and edits it by rows, and writes it |
| 3 | 4 | `recipe publish`: the loop runs, from an empty folder to a component |
| 4 | — | the loop as a test: a script piped into the `cli` publishes under a scratch name it removes after, and every file must hold exactly what the script wrote, since a row lost or doubled is silent |
| 5 | 7 | raw keys: Tab and history at the terminal |
| 6 | 6A, if chosen | commands per view |

One detail for step 2: the end of piped input sends `quit`, which refuses while a file has changes, so a script that leaves a file unwritten would never end. The CLI then ends the run with `quit force`, the refusal above it naming the file.

## Not in it

What the `ide` has and the loop does not need stays the `ide`'s: keys and focus, modes and `present`, the graph and its selection, the font's zoom, the palette and the panels. Each is a recipe the `ide` composes (V05); the terminal app needs none of them, and a project can drop any.
