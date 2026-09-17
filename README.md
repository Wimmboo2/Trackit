# TrackIt

A kanban task tracker that runs entirely in the browser. One HTML file, no build step, no server, no account — projects and tasks live in `localStorage` and never leave the machine.

---

## Why I built it

Two reasons, and both shaped the code.

The first is that I wanted a task board I'd actually use. That ruled out anything needing a login or a network round-trip to move a card, and it meant the boring parts had to be right: your data is exportable, destructive actions are hard to trigger by accident, and the thing still works when browser storage is unavailable instead of silently dropping writes.

The second is that it's a portfolio piece, which is why the interaction work goes deeper than a task board strictly needs. Drag-and-drop is hand-rolled on pointer events rather than pulled from a library, and it ships with a complete keyboard equivalent — grab a card with `Space`, move it with the arrow keys, with every move announced to screen readers. Building that from scratch is the point.

---

## What it does

[SCREENSHOT: main board view — sidebar with a project selected, three columns with cards showing priority tags, due dates and tag chips]

**Projects**
- Multiple projects, each with a name, description and one of six accent colors
- Inline rename, archive/unarchive, and delete with a confirmation that names the task count you're about to lose
- Per-project completion state — a project with all tasks done gets marked complete in the sidebar

**Board**
- Three columns: To Do, In Progress, Done
- Drag cards within a column to reorder, or across columns to change status
- A live drop indicator shows exactly where the card will land
- Full keyboard alternative: `Space` to grab, arrow keys to move between positions and columns, `Space` to drop, `Escape` to cancel

[SCREENSHOT: a card mid-drag — the tilted drag overlay following the cursor, with the accent drop-indicator line visible in the target column]

**Tasks**
- Title, freeform notes, priority (High / Medium / Low), tags, and a due date
- Due dates render as relative ("in 2 days", "3 days ago") or as calendar dates, and overdue/due-today cards are styled distinctly
- Created and updated timestamps on every task
- Editing happens in a side drawer with a focus trap; closing it returns focus to whatever opened it

[SCREENSHOT: task detail drawer open over the board — title, status and priority segmented controls, due date, tag chips, notes]

**Finding things**
- Debounced search across both titles and notes
- Filter by priority (multi-select), by tag (multi-select), and hide completed tasks
- A progress bar and percentage for the active project

**Your data**
- Export everything to a timestamped JSON file
- Import a backup, with a preview of how many projects and tasks it contains before anything is replaced
- Clear all data, gated behind typing `DELETE`

[SCREENSHOT: the Data panel — export summary, import preview card showing project/task counts, and the clear-all confirmation]

**Layout**
- Below 1000px the sidebar becomes an overlay drawer with a scrim
- Honors `prefers-reduced-motion`

[SCREENSHOT: narrow/mobile layout with the sidebar collapsed and the board filling the width]

---

## How it works

### One file, two halves

`index.html` is the entire application, split at a clean seam:

```
index.html
├─ <x-dc> … </x-dc>           lines    9–448   the template: markup, bindings, no logic
└─ <script data-dc-script>    lines  449–1335  the logic: class Component extends DCLogic
```

The template is declarative and framework-flavored but not JSX — `<sc-if value="{{ x }}">`, `<sc-for list="{{ xs }}" as="y">`, `{{ }}` interpolation, plus `style-hover=` / `style-focus=` attributes that compile into real pseudo-class rules.

The seam between the two halves is a single method, `renderVals()`. It returns one flat object containing every value and every event handler the template needs, and each `{{ name }}` resolves against that object. There is no prop drilling and no context: the view is a pure function of one object, and all state transitions live in the class above it.

### The runtime

`bullshi/support.js` is a generated runtime (marked do-not-edit) that parses the template, fetches React 18.3.1, ReactDOM 18.3.1 and Babel Standalone 7.29.0 from unpkg, compiles the logic script in the browser, and renders the template as React elements.

The practical consequence: **no `package.json`, no `node_modules`, no build step** — but the first page load needs a network connection. All three CDN scripts are pinned by version and subresource-integrity hash, so a compromised or swapped CDN asset fails closed rather than executing.

### Data model

Everything is one JSON object under the `localStorage` key `trackit.v1`:

```jsonc
{
  "schemaVersion": 1,
  "projects": [
    { "id", "name", "description", "color", "createdAt", "archived" }
  ],
  "tasks": [
    { "id", "projectId", "title", "notes",
      "status":   "todo" | "doing" | "done",
      "priority": "High" | "Medium" | "Low",
      "tags": [], "dueDate": "YYYY-MM-DD" | null,
      "order": 1, "createdAt", "updatedAt" }
  ],
  "ui": {
    "activeProjectId",
    "filters": { "q", "priorities": [], "tags": [], "hideDone" }
  }
}
```

Tasks are a flat array rather than nested under projects, so filtering and search are a single pass and moving a task never restructures the tree.

### Card ordering

Card position is a float, not an array index. `orderBetween()` assigns a moved card an `order` exactly halfway between its two new neighbors, so a drag rewrites **one** task's `order` field instead of renumbering an entire column. Columns render by sorting on that value.

### The trust boundary

`normalize()` is the single gate every piece of untrusted data passes through, and both `localStorage` reads and user-supplied import files go through the same function. It rejects a mismatched `schemaVersion` outright, then type-checks each field, whitelists `status` and `priority` against the known sets, regex-validates `dueDate`, caps tags at 24 per task, drops tasks whose `projectId` no longer exists, and re-validates `activeProjectId` against surviving projects. A malformed or hostile backup file degrades to a rejection with a readable error, not a broken board.

### Persistence

Writes are debounced 300ms and flushed on unmount. On boot the app probes `localStorage` with a throwaway write; if that throws — private browsing, blocked site data — it runs in memory and shows a dismissible banner saying changes won't be saved, rather than pretending to persist.

### Styling

`styles.css` is the only stylesheet: a token sheet (`:root` custom properties, 100–900 tonal ramps generated in OKLCH, spacing and radius scales) plus a small component layer (`.btn`, `.input`, `.seg`, `.card`, `.tag`, `.dialog`). It uses `color-mix()` and `:has()`, so it targets current evergreen browsers.

---

## Design notes and problems solved

Specific decisions and bugs in this codebase, with the reasoning behind them.

### Two index spaces, one off-by-one

The keyboard move handler and `moveTask()` each built their own list of a card's siblings — but one **included** the card being moved and the other **excluded** it. The indices therefore meant different things, and the handler added `+ 1` to compensate in the wrong direction. Pressing ArrowDown once moved a card two slots, and the screen-reader announcement confirmed the wrong position out loud. ArrowUp used the same variable without the adjustment and was correct, which is exactly why it went unnoticed. ([`719c9d8`](../../commit/719c9d8))

### Index-keyed rows lose focus on reorder

With that fixed, the first keypress worked and every subsequent one did nothing. The runtime keys `sc-for` rows by array index (`key: i`), so reordering a list rewrites each card **in place** rather than moving DOM nodes. Focus stayed on the slot while the card underneath it changed, so later keypresses landed on a different task and were correctly ignored as "not the grabbed card."

The runtime is generated and can't be edited, so the fix lives in the app: after a keyboard move, find the moved card by its `data-card` id and restore focus to it. Worth knowing as a general constraint — anything in this codebase holding a reference into a rendered list has to survive nodes being reused positionally. ([`719c9d8`](../../commit/719c9d8))

### Outer box-shadows and scroll containers

Cards draw their 1px border with `box-shadow: 0 0 0 1px`, which paints *outside* the border box. The column's card list is an `overflow-y: auto` scroller that had zero vertical padding, so the scroller's clip edge sliced the ring off the first and last cards — they looked chopped. The same clip was shaving the 3px ring that marks a keyboard-grabbed card. Four pixels of vertical padding gives both room. ([`eadb8d6`](../../commit/eadb8d6))

### Dragging selected text

Pointer-based dragging over text starts a native text selection, so dragging a card highlighted its title and everything the cursor crossed. Rather than fighting it with `preventDefault()` on pointerdown — which also suppresses focus — the drag is anchored on a non-selectable element: `user-select: none` on `[data-card]` means a selection never begins. The tradeoff is that card titles can't be selected to copy; the drawer holds an editable copy of every field, and this matches how Trello and Linear behave. ([`eadb8d6`](../../commit/eadb8d6))

### Destructive actions scale their friction

Three different confirmation patterns, deliberately not one shared component: deleting a task is a two-step arm/confirm inline in the drawer; deleting a project opens a modal that names how many tasks go with it; clearing all data requires typing `DELETE`. Friction is proportional to what's unrecoverable.

### Accessibility as structure, not decoration

Modals and the drawer trap Tab and restore focus to the element that opened them. Card moves are announced through an `aria-live="polite"` region. Every interactive element has a themed `:focus-visible` ring rather than the browser default. The keyboard drag path is a genuine equivalent to the mouse path, not a fallback — it's the same `moveTask()` underneath.

### Known limitations

- **First load needs a network connection.** React, Babel and the icon font come from a CDN at runtime. There's no service worker, so this isn't offline-capable in the strict sense.
- **Data is scoped to one browser and one origin.** Clearing site data wipes it. There's no sync between devices — export/import is the transfer mechanism, on purpose.
- **Babel compiles the logic script on every page load**, which costs startup time in exchange for having no build step.

---

## Running it

**Requirements:** a current browser, any static file server, and a network connection on first load.

```bash
git clone https://github.com/Wimmboo2/Trackit.git
cd Trackit

# any static server works — this one needs no install
python3 -m http.server 8000
```

Then open <http://localhost:8000>.

Serving over HTTP rather than opening `index.html` directly matters: on a `file://` origin, browsers restrict `localStorage` and subresource-integrity checks on the CDN scripts, so the app may boot into its in-memory fallback or fail to start.

**Environment variables:** none. There is no configuration file and nothing to set up.

### Configuration

Three options are compiled into the `data-props` attribute on the logic `<script>` tag in `index.html`. Edit the `default` values to change them:

| Option | Values | Default | Effect |
| --- | --- | --- | --- |
| `showProgress` | boolean | `true` | Shows the progress bar and percentage above the board |
| `dueFormat` | `Relative` \| `Calendar` | `Relative` | Due dates as "in 2 days" or as "Sep 19" |
| `startWithSidebar` | boolean | `true` | Whether the project sidebar is open on load (wide viewports only) |

### Repository layout

```
index.html    the entire app — template above, component logic below
styles.css    design tokens (:root variables, OKLCH ramps) + component classes
favicon.svg
bullshi/
  support.js       generated template runtime — do not edit
  _ds_bundle.js    design-system bundle (currently a no-op)
  _ds_manifest.json
  _adherence.oxlintrc.json
```

---

## License

MIT. A `LICENSE` file has not been added to the repository yet.

## Contributing

This is a personal project, but bug reports and pull requests are welcome via [GitHub issues](https://github.com/Wimmboo2/Trackit/issues).

If you're changing the interaction code, please check both paths — mouse drag *and* keyboard drag (`Space`, arrow keys) — since they share `moveTask()` and a change to one affects the other.
