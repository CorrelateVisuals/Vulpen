# Views, modes and the Perform tabs

A proposal for the lead (A00), asked on 2026-10-15: "We have to better think out UX for how multiple views and nested views and the perform and edit modes work and the present panel tabs." The same day the lead set it aside: "skip for now". Each question gives the situation, what each option means, its pros and cons, whether the engine or only recipes change, and a pick.

**Built on 2026-10-16**, on the lead's word: "implement views-and-modes.md, pick A everywhere." Under each question, **Built** says how, and names each call made while building it, which waits on the lead. "Now" describes the tree before.

## Now

The `ide` hosts projects, each a view (V03). `view load` hosts one, and each is presented as it comes, in a Perform tab of its own (M4); a press on a tab presents that view. Three panels then follow two different rules:

- the terminal's lines go to the view hosted most recently, unless a line names one (`wave: ls`);
- the graph shows the view hosted most recently;
- the Perform panel shows the view presented.

So after a press on an older project's tab, the Perform panel shows that project while the terminal and the graph stay on the newest. `view close` (built on 2026-10-15) unpresents the view it closes and presents nothing in its place, as the lead chose.

## 1. Which view you work on

- **A. The view presented.** The terminal's lines and the graph follow the view whose window the `ide` shows, the `present` connection; with none, as in the CLI or after `view close`, the view hosted most recently, as now. A press on a Perform tab, or `present b` typed, switches all three panels at once.
  - Pros: one answer everywhere, and no new command or state: the switch is `present`, which the log already keeps as a connection (V06, V08), and the connection stays the one place the choice lives (C02).
  - Cons: with nothing presented, the terminal and the graph fall back to the newest view while the Perform panel shows nothing, until the next `present`.
  - Recipes only: the command-line and graph parts read their view's connections (B8), as the image part does since step 2, and look only for a connection from a hosted view's window, so neither knows the modes part (V05).
- **B. A current view of its own**, set by a new command (`view show <name>`) that all three panels follow and a tab press sends, which presents too. Pros: the CLI could use it as well. Cons: a second record of the same choice beside the `present` connection (C02), and one more command (decision 2 of the handoff, "options, not variants").
- **C. As now**, and a graph tab per loaded view later (G1's C). Pros: no work. Cons: the panels keep disagreeing.
- Pick: **A**.
- **Built** (763bf81): the shared `Presented` (RV09) finds the view presented, the one a view hosts whose window a connection of that view takes, the newest of them when several do; the command-line and graph parts follow it, else the view hosted most recently, and their tabs name it. Two calls:
  - *As the frame began.* Both read the presented view from the frame's view, as the image part does, so a line in the same frame as a `present b` still goes to the view shown before it, and lines pasted after a `view load c` while `b` is presented go to `b` until the modes part presents `c`, the next frame. A typed line is always a frame later. Reading the connections as the lines so far left them would need the engine to hand a node its own view so (a new `Cook` read), and would still leave the `view load` case, since `c` is presented only as modes next cooks.
  - *`view list` lists the views the view a line addresses hosts*, not every view the process hosts, as `child list` did. The terminal's "hosted most recently" must be the graph's, the newest the `ide` hosts itself; with the old list, a project hosting a view of its own sent the CLI's lines into that inner view. A view a project hosts is reached by its name, and `<project>: view list` lists them.

- Now: refused, as no two views share a name ("two nodes or views are named wave").
- **A. Refused, naming the view and that it is hosted**, so the person presses its tab. Recipes only (the library part's message). Pros: nothing changes but the words. Cons: none known.
- **B. It switches to the view**: the library part would send `present`, a command of the modes part, which the CLI does not have (V05). Cons: a part depending on another part's command.
- Pick: **A**.
- **Built**: `view load` and `view new` of a name hosted already, anywhere, are refused before anything is sent: `a view named wave is hosted already, from …/wave/view.vlp`.

## 3. Nested views: a project that hosts views of its own

Addressing needs nothing new: every hosted view's name is unique, so `<name>: …` reaches a nested view directly, and `view close <name>` reaches its host from anywhere.

- **3.1 In the graph.**
  - **A. A view the shown view hosts is a box of its own**, of kind `view` (step 12's colors), listing its `view.vlp`; a curve leaves it where its host takes its window; a press on its file opens it, and `delete` on it sends `view close <name>`, which is "the same for a deleted node". Recipes only (the graph part reads the view's `[view]` entries). Pros: the project's structure shows whole (V01). Cons: about 40 lines in the graph part.
  - **B. Not shown, as now.** Cons: a project's nested views are invisible in the graph.
  - Pick: **A**.
  - **Built** (763bf81), with three calls: `select` takes a hosted view's name, so a press selects its box and `delete` closes it in the same group as the nodes it removes; `info` shows a view selected under `[view "<name>"]`, its file whole and each connection that takes its window, since `info` with no node shows the selection and would else refuse; and the `.ini` names its kind `view`, as `color view = <role>`.
- **3.2 Presenting a nested view.**
  - **A. Only views the `ide` hosts itself**, as now: a nested view shows through its host, as its host composes its window. Pros: a view stays whole (V03), and nothing changes. Cons: a nested view cannot fill the Perform panel alone.
  - **B. Any nested view**: an engine change, a connection from a grandchild's window. Cons: it reaches past a view's own boundary.
  - Pick: **A**. The graph showing a nested view's own graph waits until a project with nested views is in use (C00).
  - **Built**: as before, but `present` of a view a project hosts names why it is refused: `inner is hosted by a view this view hosts, and shows through it`.

## 4. Perform mode with nothing presented

- Now: the window shows nothing but its ground, and `escape` goes back.
- **A. `mode perform` refuses while nothing is presented**, naming `present`, and `view close` of the view presented in perform mode goes back to edit mode. Recipes only (modes, about 6 lines). Pros: never a blank window that looks stuck (A02).
- **B. As now.**
- Pick: **A**.
- **Built**, with two calls:
  - *A view hosted that modes has not presented yet is presented first*, as the frame would present it as it comes, so `f1` in the same frame as `view load` finds the view rather than being refused; the `ide` test's chord in its first frame is that case.
  - *Perform mode gives way whenever nothing is presented nor about to be*, not only after `view close`: a view presented and closed in one frame, or a replayed `param set ide.modes mode perform` with nothing hosted, would else leave the window blank.

## 5. A refused `quit` in perform mode

- Now: the window's close button sends `quit`, which an unsaved file refuses; the refusal goes to the terminal, which perform mode hides, so the button seems to do nothing.
- **A. A line refused in perform mode brings edit mode back**, so the refusal shows: the modes part reads the log, as the terminal does. Recipes only, about 4 lines.
- **B. Perform mode shows the newest error over the view.** Cons: a text layer over the view, more code.
- Pick: **A**.
- **Built**, with one call: *every error the log prints* brings edit mode back, not only a refused line's, since the log does not tell them apart and each hides alike in perform mode, as a live build that fails.

## 6. Close marks on the terminal's and the graph's tabs

The lead chose a close mark on each tab (old 28). On the editor's tabs it closes the file, and on the Perform tabs the view (`view close`), unpresenting it first. The terminal's and the graph's tab each name the view the panel follows, so their mark has no file or view of its own.

- **A. No mark on those two tabs.** One place closes a view, the Perform tab; a panel closes by its seam (D10). Pros: no two marks close the same view. Cons: not every tab has a mark, as the lead asked.
- **B. Their mark closes the panel**, by setting its seam's ratio. Cons: a closed panel comes back only by dragging the seam out of the window's edge.
- **C. Their mark closes the view they name too.** Cons: three marks close one view, and a terminal tab's mark ending a project surprises.
- Pick: **A**.
- **Built**: nothing changed, as the two tabs had no mark; the docs say so.

## 7. `child add` and `child list` as `view` words

The log keeps primitives, and `child add <name> <file>` is the one that hosts a view. It starts a view whose `view.vlp` does not exist yet, so a log replays from an empty folder. `view load` is the one people type: it refuses a folder with no `view.vlp` and names the view for its folder. So two commands stay, but they can share a group.

- **A. `view add <name> <file>` and `view list`**, beside `view close`, `view load`, `view new` and `view save`, so `help` shows one `view` group. Engine: two command names, with their tests and docs. Cons: a log written before still says `child add`, and no longer replays.
- **B. Keep `child add` and `child list`.**
- Pick: **A**.
- **Built** (4e2097a): `view add <name> <file>` and `view list`, with the call under 1 that `view list` lists the views the view a line addresses hosts. A log saved before says `child add`, and no longer replays.

## What follows a switch

These stay as the lead kept them (old 10 and 11), and with 1A they follow the view presented: the selection goes, `select`, `selected` and `delete` act on the view the graph shows, and the graph and the Perform panel start at home (step 2). The terminal's and the graph's tabs name the view (`terminal - demo`, `graph - wave`).

## Known limits, not proposed

- A presented view reads the window's pointer in window pixels, which match its image only in perform mode, and less so now that the Perform panel pans and zooms. No presented project reads the pointer yet (C00).
- A host whose connections take the windows of two views shows two; 1A then follows the newest of them.
