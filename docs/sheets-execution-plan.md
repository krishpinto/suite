# Sheets on IronCalc: execution plan

Status: draft for discussion.
Builds on `docs/adr/0001-sheets-ironcalc-calculation-core.md` and `docs/sheets-rewrite-spec.md` (branch `sheets/ironcalc-core`).
Items marked **(added)** are not in the spec. They fill gaps found while testing the current app.

All frontend paths are relative to `frontend/src/apps/sheets/`.

---

## The architecture in one picture

```
MAIN THREAD                                    WORKER THREAD
                                               │
canvas/  (draws; reads only from the cache)    │
  ▲   │ you type → command object              │
  │   ▼                                        │
  │ core/client.ts   dispatch(cmd) ─ message ─►│ core/worker.ts
  │   │                                        │   └ core/workbook.ts  apply(cmd)
  │   │                                        │       └ IronCalc (Rust → WASM)
  │   ◄──────────── message: version, values ──│
  │   ▼                                        │
core/display-cache.ts  Map "Sheet1:1:2" → "10" │
                                               │
       │ submit(cmd)            ▲ commands(seq, cmd)
       ▼                        │
collab-server/  (Node + @ironcalc/nodejs): numbers every command, saves it, broadcasts it
       │
       ▼
Frappe: Sheet Op Log (every command) + Sheet Snapshot (engine bytes)
```

**Rule:** nothing changes the workbook except a command passed to `dispatch()`.

---

## Areas

| # | Area | Decision | Phase |
|---|---|---|---|
| 1 | Engine | IronCalc (`@ironcalc/wasm`) | P1 |
| 2 | Where the engine runs | Web Worker | P1 |
| 3 | How changes are made | Commands through `dispatch()` | P1–P2 |
| 4 | How the canvas gets values | DisplayCache | P1 |
| 5 | Canvas | Split into modules, painters kept | P1 |
| 6 | Editing and input | Composables + input type detection | P2 |
| 7 | Undo/redo | Inverse commands | P2 |
| 8 | Saving | Command log + snapshots | P2 (solo), P3 (sidecar) |
| 9 | Collaboration | Sidecar sequencer | P3 |
| 10 | Deployment | Open question | before P3 |
| 11 | Permissions and security | Sidecar checks access via Frappe | P3 |
| 12 | Big-sheet performance | Worker now; dependency graph upstream later | P2+, ongoing |
| 13 | Formula correctness | Difftest gate on every engine upgrade | ongoing |
| 14 | Features | Ported as layers that read cells and write commands | P4 |
| 15 | xlsx | Import/export in the sidecar | P4 |
| 16 | Mobile and accessibility | Touch + hidden ARIA grid | P5 |
| 17 | Code quality and tests | Strict TypeScript, small files, tests per module | every phase |
| 18 | Internal launch scope | See section 21 | P1 + P2 core |

---

## 1. Engine

**Today:** `engine/formula.js` + `engine/sheet.js`. Hand-written, no parse tree, blanks counted as 0. Reproduced bugs:
- `AVERAGE` over blanks
- `"abc"="xyz"` returns TRUE
- `$A$1` gives `#NAME?`
- `2^3^2` evaluates in the wrong order

**Decision:** IronCalc.
- Rust compiled to WASM.
- MIT/Apache license.
- 494 functions.
- 99.91% agreement with the reference corpus (`engine/difftest/IRONCALC-REPORT.md`).

**Why not the alternatives:**

| Option | Reason it was rejected |
|---|---|
| HyperFormula | GPLv3 or a commercial license |
| Fixing the current engine | No tree and no correct type rules; rewriting it costs more than adopting an engine |
| `engine2` | Far from 494 functions |

**How it works inside:**
- A formula is tokenized and parsed into a `Node` tree, stored once per sheet in `parsed_formulas`.
- A cell stores the tree's index (`f`) and its last value (`v`).
- Filled-down formulas share one tree (relative references).
- There is no dependency graph. Every edit marks all formulas `Unevaluated` and recalculates the whole workbook. A cell that needs another cell calculates it first, on demand. See area 12.

## 2. Where the engine runs

**Decision:** IronCalc runs inside a Web Worker, so a slow recalculation never freezes the page.

**How:**
- `core/client.ts` creates `new Worker('core/worker.ts')`.
- The two sides talk only by `postMessage`. Each request carries a `reqId`, and the reply echoes it, which resolves the matching Promise.

| Message | Does |
|---|---|
| `init` | Load the workbook from snapshot bytes |
| `apply` | Run commands; reply with the new `version` and inverse commands |
| `readViewport` | Return display values for a rectangle (one message per screen, not per cell) |
| `readCells` | Return input, display or style for specific cells |
| `toBytes` | Return the workbook as bytes for saving |

**Rule:** the main thread never imports `@ironcalc/wasm`.

**(added) Worker crash:** IronCalc can panic on edge input.
- `client.ts` catches the worker's `error` event.
- It starts a new worker from the last snapshot and replays the commands sent since.
- The user sees a short "recovering" state, not a dead page.

## 3. How changes are made: commands

**Decision:** every change is a plain JSON object, defined in `core/commands.ts` (already written and tested):

```
{ id, actor, ts, type: 'setInput', payload: { sheet: 'Sheet1', row: 1, col: 1, input: '5' } }
```

- There are 20 types, including `setInput`, `clearContents`, `setRangeStyle`, `insertRows`, `deleteRows`, `moveRows`, `setFrozen`, `addSheet`, `renameSheet`, `setDefinedName` and `batch`.
- `validateCommand()` rejects malformed commands on the client and again on the server.
- `core/workbook.ts` `apply()` maps each type to one IronCalc call (`setInput` → `model.setUserInput`).
- `batch` pauses evaluation, applies everything, then recalculates **once**. Paste and fill use it.
- Rows and columns are 1-based in commands and 0-based in the canvas. The CellProvider converts between them.

**Why:** one shape of change gives undo (area 7), saving (area 8) and collaboration (area 9) for free.

**Why not IronCalc's own sync format (`flushSendQueue` / `applyExternalDiffs`):**
- It is binary, and the format can change between IronCalc releases. A saved log could become unreadable after an upgrade.
- JSON commands stay readable in the op log ("who changed A1 to 5").
- JSON commands keep the engine replaceable: replaying the log into a different engine rebuilds the workbook.

**P2 adds `moveSheet`**, so undoing a sheet delete can put the tab back in its old position.

## 4. How the canvas gets values: DisplayCache

**Decision:** a `Map` on the main thread (`core/display-cache.ts`), keyed `"sheet:row:col"`, with values `{ display, style?, provisional? }`.

**Read:**
1. The painter calls `CellProvider.getDisplay("B1")`.
2. It converts to `Sheet1:1:2` and looks up the cache.
3. **On a hit:** draw the value.
4. **On a miss:** draw blank, then schedule one `readViewport` for the visible area plus one extra screen.
5. Fill the cache and repaint.

**Write paths:**
- **Refill:** from `readViewport` replies.
- **Optimistic echo:** `dispatch(setInput)` writes the typed text immediately, marked `provisional`.
- **Clear:** `clear()` on every version bump. IronCalc doesn't report which cells changed, so everything is cleared.

**Known cost:** one blank frame after each edit while the refill arrives. This goes away once IronCalc can report changed cells (area 12).

## 5. Canvas

**Today:** `canvas/index.js` is one 2,044-line function with 31 callbacks. It mixes scrolling, selection, editing, autocomplete, fill handle and cell storage.

**Decision:** keep what works and split the rest.

| Keep (convert to TS) | New modules |
|---|---|
| `geometry`, `renderer`, `painters/*`, `scrollbars`, `overlay` | `index.ts` (assembly), `types.ts` (`CellProvider`, `GridHost`), `viewport.ts`, `selection.ts`, `render-loop.ts`, `input/hit-test.ts`, `input/mouse.ts`, `input/keyboard.ts` |

- **View state** lives in `core/view-model.ts`: widths, heights, scroll, freeze, hidden rows, zoom, selection.
- **The canvas stores no cell data.**
- **Size rule:** no module over 400 lines.
- **Speed targets:**
  - Scroll under 8 ms per frame at pixel ratio 2 (P1).
  - Paint under 3 ms per frame (P5).
- `utils/sheet-codec.js` is converted to TypeScript in P1.

**(added) Pixel-ratio bug:** `canvas/index.js:15` caches `devicePixelRatio` once, while `renderer.js:24` reads it live. Browser zoom or moving to another monitor draws the grid at the wrong scale. Fix it in `viewport.ts`:
- Read the ratio live.
- Resize on `matchMedia` resolution change.

## 6. Editing and input

**Today:** `components/SheetEditor/index.vue` is 305 KB and holds everything.

**Decision:** delete it. Replace it with a thin component plus composables, each under 500 lines:

| Composable | Owns |
|---|---|
| `useSelection` | Selection |
| `useCellEditing` | Editor, formula bar, autocomplete, range picker. Commit becomes `setInput` |
| `useClipboard` | Copy/paste and fill handle, sent as `batch` |
| `useFormatting` | Toolbar actions become `setRangeStyle` |
| `useHistory` | Undo/redo |
| `useKeyboard` | Shortcuts |
| `usePersistence` | Saving |
| `useCollaboration` | Collaboration |
| `useCharts` | Charts |
| `useTouch` | Touch (filled in P5) |

**(added) Input type detection** in `useCellEditing`, before building `setInput`:
- **Today:** typing `1,000` stores the text `"1,000"`, so `=A1*2` gives 2 (old engine) or `#VALUE!`.
- **Check first:** whether IronCalc's `setUserInput` already parses `1,000`, `$1,000`, `50%` and dates. Test it.
- **If it doesn't:** convert to the number and send a `setRangeStyle` with the number format, as one `batch`.

## 7. Undo/redo

**Decision:** inverse commands. IronCalc's built-in undo is not used.

**How:**
1. Before applying a command, the worker reads what will be overwritten. For example, `setInput A1` reads the old A1 input.
2. It builds the inverse command and returns it.
3. `useHistory` keeps `{ forward, inverse }`.
4. **Undo** = `dispatch(inverse)`. **Redo** = `dispatch(forward)`.

**Why:** undo goes through the normal path, so other users receive it like any edit.

**Limits:**
- Each user undoes only their own commands.
- Undo can overwrite another user's **later** edit to the same cell. Example: I set A1 = 5, you set A1 = 7, then I undo, and A1 returns to my old value, losing your 7. Accepted for v1, because last-writer-wins per cell is the conflict rule everywhere.
- Inverses of very large changes (over ~1M cells) aren't recorded, and that entry is marked non-undoable.

## 8. Saving

**Today:**
- The whole sheet is saved as one blob.
- Edits made during a save are dropped.
- Two open tabs overwrite each other.

**Decision:** the command log is the source of truth.
- **`Sheet Op Log`:** one row per command, with type, full JSON, actor, sheet and affected range.
- **`Sheet Snapshot`:** `workbook.toBytes()` stored in a binary field, written periodically.
- **Reload:** latest snapshot + the commands after it.
- **Solo mode (P2):** the client sends the pending commands through the existing `save_sheet` (`ops` parameter).
- **Collab mode (P3):** the sidecar writes the log and snapshots.
- **Feature data** (charts, filters, comments…) and view state are saved as a JSON slice next to the snapshot.
- **Version history** (`versioning/timeline.py`, `ops_for_cell`) works on command rows unchanged.

**Why it fixes data loss:** saves append commands instead of overwriting the document, so nothing typed is ever replaced by a stale copy.

**Details:**
- **Save cap:** `versioning/save.py` allows 500 commands per save (`MAX_OPS_PER_SAVE`). A large paste or fill is sent as **one `batch` command**, so it counts as one row.
- **Sidecar flushing:** the sidecar writes the log every ~2 s or every 100 commands, whichever comes first.
- **No migration:** sheets made in the current app are not converted. The app has close to no users, and a converter would carry old engine behaviour into the new one. Test workbooks are built by dispatching commands.
- **Deleted storage:** the `Sheet Collab State` doctype (the old Yjs state) is dropped together with the old collab stack.

## 9. Collaboration

**Today:** Yjs + Hocuspocus. Only single-cell edits sync, and the v2 server calls an old app name.

**Decision:** replace it with a sequencer in `collab-server/`, running `@ironcalc/nodejs`.

**How:**
1. The client sends `submit { commands }`.
2. The sidecar validates each command with the same `core/commands.ts` and applies it to its own copy of the workbook.
3. It assigns the next number (`seq`), then broadcasts `commands { seq, command }` to everyone.
4. Clients apply commands strictly in `seq` order:
   - **Own command:** it's only a confirmation, so it isn't applied again.
   - **Others' commands:** sent to the worker.
5. **If the sidecar rejects a command,** the client applies the stored inverse to roll back.
6. **Reconnect:** join again, load snapshot + commands, then resend unconfirmed ones.
7. **Conflicts:** last writer wins per cell, in `seq` order.

**Presence (cursors, who's here):** `collab/awareness.js` stays.

**Removed:** `collab/ydoc.js`, `collab/cells-binding.js`, `collab/hocuspocus-client.js`, `collab/frappe-provider.js`, Hocuspocus.

**Transports:** two ways to reach the sidecar, as today.

| Path | How | When |
|---|---|---|
| A | Through Frappe's realtime (socket.io), relayed to the sidecar | The sidecar runs but isn't reachable from the browser |
| B | Direct websocket to the sidecar | The sidecar is reachable |

Both carry the same messages: `join`, `submit`, `commands` and presence.

**Feature settings** (charts, filters, comments…) are last-writer-wins **per feature** in v1. If two users edit chart settings at once, one edit wins. Turning feature edits into commands is a v2 item.

Presence and the sidecar requirement: see "Answers to the spec's open questions".

## 10. Deployment (open)

Collaboration and xlsx both need the Node sidecar running next to Frappe.

**Questions:**
- Can Frappe Cloud run an extra Node process per bench?
- If not, is collaboration self-hosted only, or does sequencing move into Frappe (Python + socket.io)?

**This blocks P3, so it should be answered early.**

## 11. Permissions and security

- **Browser → sidecar:** the sidecar checks the user's session with `suite.sheets.collab.check_collab_access`. A read-only user receives updates, but their `submit` is rejected.
- **Sidecar → Frappe:** a shared secret in the `X-Collab-Secret` header.
- **The sidecar re-validates every command,** so a modified client can't send a malformed change.
- **(added) Audit the old app's leaks** and confirm each is gone in the new design:
  - Live edits broadcast to every user on the site
  - XSS through sheet tab names
  - Session ID exposed to page scripts

  Each gets a test, not just a deletion.

## 12. Big-sheet performance

**Today, with IronCalc:** every edit recalculates every formula in the workbook. The ADR measured ~90 ms for 100k simple cells and ~9 s for heavy sheets.

**Plan, in order:**
1. **Worker (P1).** The page never freezes, and typed text shows immediately.
2. **`batch` (P2).** Paste, fill and import recalculate once.
3. **Measure.** A benchmark with 100k formulas timing one edit. Check what sheet sizes real users have.
4. **(added) Manual calculation mode** as a fallback for very heavy sheets: recalculate on a button press, as Excel does.
5. **Dependency graph in IronCalc (upstream contribution, after P2):**
   - For each formula, read its references from the `Node` tree when it is entered.
   - Keep a reverse map: cell → formulas that use it. Ranges are stored once, not expanded.
   - On edit, walk the map to find the dirty cells, mark only those `Unevaluated`, and evaluate them. IronCalc's on-demand evaluation already handles the order.
   - Return the changed cells, so `display-cache.ts` clears only those.
   - **Hard cases:**
     - `NOW`/`RAND`/`INDIRECT`/`OFFSET`: always dirty (volatile).
     - Insert/delete rows: rebuild the map.
     - Shared relative trees: references are computed per cell.
   - Start from the existing upstream issue ("use dependency DAG") and agree the design with the maintainers before writing it. A vendored fork (MIT) is the fallback.

**Other engine limits, and the plan for each:**

| Limit | Effect | Plan |
|---|---|---|
| No batch range read | Reading a screen of cells is one call per cell | The loop stays inside the worker, so it's one message per screen (1.25 ms for 50×30). Upstream contribution later |
| No history-free bulk load | Loading a snapshot and replaying commands builds undo history it doesn't need, using memory | Replay goes through `batch` + `pauseEvaluation`. Upstream contribution later |

## 13. Formula correctness

- **Gate:** `engine/difftest/ironcalc.test.ts` runs on every `@ironcalc/wasm` upgrade. Agreement must not drop below the 0.8.4 baseline.
- **Pin the IronCalc version.** Upgrade only when the difftest passes.
- **Bugs found** in IronCalc are reported upstream with a failing case.

## 14. Features

Each feature is a separate layer. It reads cells through the `CellProvider` and changes cells only through commands. Its own settings are saved as a JSON slice.

| Feature | Writes |
|---|---|
| Pivot | `batch` of `setInput` |
| Sort | `moveRows` |
| Filter | Hides rows through the ViewModel |
| Charts, validation, comments, conditional formatting, sparklines, protection | No cell writes (draw-time or block edits) |
| Smart fill, links | `batch`/`setInput` |
| Slicers | Through sort/filter |
| Rich text | No cell writes (stored in its slice) |
| Merge | Interim `engine/merge.js` until IronCalc merged cells reach npm; then `mergeCells`/`unmergeCells` commands |

These stay in-house because IronCalc doesn't model them: pivots, charts, filter/sort criteria, validation, comments, rich text.

**Rule:** no feature imports the engine or the worker directly.

## 15. xlsx import/export

- It runs in the sidecar, because `@ironcalc/nodejs` has xlsx support and the WASM build does not.
- Upload and download go through whitelisted Frappe endpoints that proxy to the sidecar.
- `engine/xlsx-io.js` is deleted.
- **Risk:** IronCalc's xlsx code uses `zip` 0.6.6, which is unmaintained. Exposure stays in the sidecar (server side, not every browser). Watch upstream, and limit upload size.
- **Depends on area 10.**

## 16. Mobile and accessibility (P5)

- **Touch:** pan, tap to select, long-press menu, drag handles, pinch zoom.
- **Accessibility:** a hidden ARIA grid of the visible cells, focus following the selection, and the active cell announced to screen readers.
- **Paint speed:**
  - Repaint only changed rectangles.
  - Cache wrapped-text layout.

## 17. Code quality and tests

- **Strict TypeScript** for all new code, checked in pre-commit and CI (`tsc --noEmit -p frontend/tsconfig.sheets.json`).
- **Size limits:** files under 400 lines in `canvas/` and under 500 in `components/SheetEditor/`.
- **Tests:**
  - Every new `core/` and `canvas/` file has a unit test next to it.
  - Test workbooks are built by dispatching commands, not stored as files.
  - Property tests: apply then apply-inverse restores identical bytes, for every command type.
  - Convergence test: N simulated users send random commands, and all end with identical bytes.
  - E2E per phase in `e2e/drive-backed-apps/specs/sheets/`: open, edit, undo, save, reload, two-user collab, restore, xlsx import.

---

## 18. Answers to the spec's open questions

**1. Presence transport (P3).** The sidecar relays presence frames (`yjs_awareness`) unchanged.
- `collab/awareness.js` keeps working, with only a small transport adapter.
- Presence shares the connection that commands already use, so there is no second channel to secure.

**2. Path A still needs the sidecar (P3).** The sidecar becomes **mandatory for collaboration**.
- Every client must apply the same commands in the same order. Something has to hand out that order and reject invalid commands, and peer-to-peer can't do that. So collaboration without a sequencer doesn't exist in this design.
- **Without a sidecar, Sheets still works single-user:** the worker is authoritative, and saving goes through `save_sheet`.
- **Path A is kept only as a transport,** for setups where the sidecar runs but the browser can't reach its socket. Whether it's needed depends on the Frappe Cloud answer (area 10).

**3. Save cap (P2).** Use the `batch` option. One large edit becomes one command and one op-log row.
- It also matches undo: one paste = one undo step.
- Add a payload size limit, so one huge paste can't produce an oversized row.

**Spec inconsistency to fix:** P1's scope says "render an imported v1 doc", but decision 2 says there is no v1 importer. P1 should read "render a workbook built by dispatching commands", which matches its own acceptance criteria.

## 19. Bugs reproduced in the current app: re-check on IronCalc

Each becomes a test before it is called fixed.

| Input | Expected | Current app |
|---|---|---|
| `=AVERAGE(A1:A10)`, only A1:A3 filled | Average of the 3 values | Blanks counted as 0 |
| `="abc"="xyz"` | FALSE | TRUE |
| `=$A$1+1` | Works | `#NAME?` |
| `=2^3^2` | 64 (Excel order) | Wrong order |
| `=TODAY()+7` | Next week | `#VALUE!` |
| `=SUM(A:A)` | Column total | `#VALUE!` |
| Insert a row above a referenced cell | Reference shifts | Doesn't shift |
| Sort rows that contain formulas | Formulas move with their rows | Wrong rows |
| Fill `=ATAN2(A1,B1)` down | `=ATAN2(A2,B2)` | `=ATAN3(A2,B2)` |
| Type `1,000`, then `=A1*2` | 2000 | 2 |
| Two tabs edit and save | Both edits kept | One overwrites the other |
| Edit during an autosave | Edit kept | Edit dropped |
| Browser zoom or monitor change | Grid at correct scale | Wrong scale |

## 20. Not covered by the spec: to decide

| Item | Proposal |
|---|---|
| `ai` folder (AI actions on sheets) | Out of the rewrite. Later, its actions become commands, so they get undo and sync for free |
| Link preview backend | Keep as-is (server-side, already hardened); re-check after P2 |
| Existing sheets in the current app | Not migrated (decision 2). Confirm nobody needs their data, or export them to xlsx first |

## 21. Scope for the internal launch

**Goal: one user can open a sheet, type values and formulas, see correct results, undo, and save without losing anything, all running on IronCalc.**

**Ready for the internal launch:**

| Piece | What it gives |
|---|---|
| Worker host (`core/worker.ts`, `core/client.ts`) | IronCalc runs off the main thread; the page never freezes |
| DisplayCache + ViewModel | The canvas reads values without touching the engine |
| Canvas split into modules (P1) | Same look and painters, cleaner structure, pixel-ratio bug fixed |
| Cell editor + formula bar → `setInput` | Type values and formulas |
| Clear, insert/delete rows and columns | Basic structural edits, references shift correctly |
| Inverse undo/redo | Undo any of the above |
| Solo save/reload via `save_sheet` | Edits survive reload; no overwrites |
| Worker crash recovery | An engine panic doesn't kill the page |
| Single-editor lock | If a second user opens a sheet someone is editing, it opens read-only with "X is editing". Prevents two solo saves from overwriting each other until collaboration lands |
| Bug checklist (section 19) run on IronCalc | Proof the reproduced bugs are fixed |

**After the internal launch:**
- Autocomplete and range picker
- Copy/paste and fill handle
- Formatting toolbar
- Features (charts, filters, pivots…)
- Collaboration
- xlsx
- Mobile and accessibility
- Dependency graph

**Cut first if behind:** worker crash recovery, then insert/delete rows. Typing, undo and save/reload are the minimum.

**Risks:**
- Area 10 (deployment) is unanswered. It doesn't affect the internal launch, but it blocks collaboration and xlsx.
- IronCalc is pre-1.0.

## Questions for the team

1. Is the `sheets/ironcalc-core` direction still the plan? Should work continue on that branch, or a new branch from it?
2. Does the internal launch need multiple users editing the same sheet at once? If yes, collaboration (P3) and the sidecar move into launch scope, and question 3 must be answered first. If no, section 21 applies, with the single-editor lock.
3. Can Frappe Cloud run the Node sidecar? (area 10)
4. Do we agree the sidecar is mandatory for collaboration? (section 18)
5. Is contributing incremental recalculation to IronCalc in scope? (area 12)
6. Does anyone need data from sheets made in the current app? (section 20)
