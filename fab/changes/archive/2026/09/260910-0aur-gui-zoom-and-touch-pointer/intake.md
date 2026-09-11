# Intake: Zoom and touch pointer modes (S4 / V2)

**Change**: 260910-0aur-gui-zoom-and-touch-pointer
**Created**: 2026-09-10

## Origin

> Drafted via `/fab-draft` as stage S4 of the combined GUI execution queue
> (`fab/plans/sahil/26-09-10-gui-combined-execution.md`). Implements § V2
> ("Zoom and touch pointer modes") of
> `fab/plans/sahil/26-09-10-gui-viewer-ergonomics.md`. One-shot, no
> conversational discussion — the design record is the ergonomics plan's
> § Decision log (V-D6, V-D7, V-D8, binding) and its § UX / § V2 Do / § V2
> Acceptance sections, reproduced here in full.

Waits for S2 (`gui-fixed-geometry-and-resize`) merged — V2 is frontend-only
and builds on V1's fixed-geometry model (a stable desktop is what makes a
persistent zoom/pan state meaningful; under `auto` the desktop itself moves,
which would fight a fixed zoom level). Per the combined-execution plan, this
change adopts S3's ("gui-desktop-picker") settings/palette control **only
if** the zoom/pointer-mode rows want a similar server-driven select — the
plan's own read is that they do not (both are purely client-side postures
with fixed, small option sets: `{50,75,100,125,150,200}` for zoom,
`{trackpad, touch}` for pointer mode), so this change touches S3's territory
only in the shared palette file (`src/lib/palette/gui.ts`), adding new rows
beside S3's, never modifying S3's control.

## Why

**The pain.** `1:1` at 1080p on a phone is a 5-screen pan; `fit` mode is
unreadable at small sizes. There is no zoom control at all today —
`GUI: Fit` / `GUI: 1:1` are the only two view modes, both all-or-nothing.
Touch input maps every tap directly to a left click (noVNC's default): there
is no right-click, no scroll gesture, no way to send a modifier key, and no
way to precisely hit a small target (a 12-px IceWM close button) without an
extremely steady finger.

**The consequence of not fixing it.** A phone viewer of the gui tile is
functionally read-only today — useful for glancing at what an agent is
doing, useless for actually driving the desktop. Given V1 just made the
desktop size a deliberate host choice (including a portrait 1080×1920
preset explicitly for phone-first sessions), shipping V1 without V2 leaves
that portrait preset unreachable in practice: a 1080×1920 desktop tapped
directly is still untappable.

**The approach, and why.** Zoom is per-viewer, stepped, and pinchable
(V-D6): a `zoom` posture beside the existing `rk-gui-view` posture — `fit`
(default) or `{50, 75, 100, 125, 150, 200}` percent — with palette rows,
Ctrl+wheel/Ctrl+=/Ctrl+- chords, and pinch on touch; any non-fit zoom pans
by drag (fine) or two-finger drag (coarse), reusing noVNC's clip/drag
viewport machinery. Touch gets two pointer modes (V-D7): `touch` (today's
tap-to-click) and `trackpad` (relative cursor — one-finger drag moves the
pointer, tap clicks, two-finger tap right-clicks, two-finger drag scrolls,
long-press = press-and-hold), default `trackpad` on coarse pointers. A
modifier/special-key bar ships alongside (V-D8) because "trackpad mode
without modifiers is half a phone" — a phone still cannot send Ctrl-C to a
terminal without it.

Alternatives the plan rejected and this intake does not reopen: reopening
D1–D6/D8–D10 of the parent surface plan; C6 bandwidth work (separate
concern); file transfer, audio, multi-monitor (V-D14, permanently out of
scope for this whole plan).

## What Changes

Frontend only (`app/frontend/`).

### 1. Postures — `src/lib/gui-posture.ts`

Two new localStorage-backed postures beside the existing `rk-gui-view`/
`rk-gui-lock` pattern:

- `rk-gui-zoom`: `"fit" | 50 | 75 | 100 | 125 | 150 | 200` — per-viewer
  (localStorage, not host state — zoom is how *this* screen shows the
  shared desktop, per V-D6's rationale).
- `rk-gui-pointer`: `"touch" | "trackpad"` — per-viewer. Default resolution:
  `trackpad` when the viewer is coarse-pointer, `touch` when fine (where the
  distinction is moot — fine pointers already get precise clicks).

Read/write helper functions matching the existing posture-file shape (get
with default-resolution logic, set, and a change-notify mechanism consistent
with how `rk-gui-view`/`rk-gui-lock` currently notify `gui-surface.tsx`).

Tests: default resolution per pointer type; round-trip get/set; the zoom
step table (`fit→100→125→150→200` up, reverse down, `100→75→50→fit` below
100%).

### 2. Zoom application — `src/components/gui-surface.tsx`

**Spike first** (plan's explicit instruction): noVNC's public API exposes
only `scaleViewport` (uniform fit) and `clipViewport` (1:1, drag-to-pan) —
no first-class "zoom to N%" primitive. Two candidate mechanisms:

1. Drive `Display.scale` directly through the RFB instance's internal
   display object (undocumented but reachable — the mechanism `clipViewport`
   itself uses internally).
2. A CSS `transform: scale()` on the canvas wrapper `<div>`, with our own
   pan logic (translate offset clamped to keep the scaled canvas covering
   the viewport), independent of noVNC's viewport modes entirely.

Spike both against the pinned noVNC version in `package.json`; **the intake
does not pick one** — this is V-D6's explicitly marked Likely row. Pick
whichever proves stable (no visual artifacts at each of the six zoom steps,
correct behavior when the browser window itself resizes mid-zoom, no
interference with `resizeSession`'s `auto`-gated behavior from V1) and
**record the decision in the plan's Design Decisions** (a new subsection in
this change's own `plan.md`, per the SRAD framework's "apply decides and
records" rule — this is exactly the kind of under-specified point apply
resolves inline, not an intake-time question). The posture contract
(`rk-gui-zoom` values, palette rows, chords) is identical either way — only
the rendering internals differ.

Additional surface work regardless of which mechanism wins:
- The 1.5 s corner badge (`150%`) on any zoom change, auto-hiding.
- Pan gestures: drag (fine pointer) / two-finger drag (coarse) whenever the
  zoomed content exceeds the tile's viewport, reusing noVNC's existing
  clip/drag viewport gesture plumbing where the chosen mechanism allows, or
  a from-scratch pan-with-clamp otherwise.
- Keyboard chords on the focused tile: Ctrl+wheel steps zoom by one notch
  per wheel tick (direction-sensitive), Ctrl+= zoom in, Ctrl+- zoom out,
  Ctrl+0 zoom to fit — registered through the existing keybinding registry
  (so they appear in the Shortcuts tab) and reclaiming focus over noVNC's
  own canvas event handler the same way other tile chords already do.
- Pinch-to-zoom on coarse pointers, mapped to the same step table (pinch
  distance delta crossing a threshold advances/retreats one step — exact
  pixel-to-step mapping is an implementation detail the plan leaves open,
  Tentative-graded below).

Tests: vitest for the posture-driven zoom-step table and the badge's
show/auto-hide timing (fake timers); Playwright desktop: Ctrl+wheel steps
zoom and the badge shows `125%`; Playwright mobile (`hasTouch`, 375 px):
pinch reaches 200% and pans, with intent comments per Constitution § Test
Intent Comments.

### 3. The trackpad pointer-translation layer — `src/components/gui-pointer.ts` (new)

The touch↔pointer translation noVNC does not provide:

- **One-finger drag** → relative pointer move (gain 1.0–1.5, tunable
  constant, not user-exposed) — moves an rk-drawn or noVNC-native cursor
  indicator by the delta rather than jumping to the touch point.
- **Tap** → click at the current (relative) cursor position, using a
  180 ms / 10 px movement threshold to distinguish a tap from the start of a
  drag.
- **Two-finger tap** → right-click (button 3) at the current cursor
  position.
- **Two-finger drag** → scroll wheel events, 1:1 with drag distance, no
  momentum.
- **Long-press** (500 ms, movement under threshold) → press-and-hold (mouse-
  down without release, held until the finger lifts) — the mechanism a
  drag-and-drop or a context-hold interaction needs.

This layer sits **in front of** noVNC's pointer handler and emits synthetic
noVNC pointer moves/clicks; it disables noVNC's own `dragViewport` gesture
handling while trackpad mode is active (per V-D7's explicit note — the two
gesture systems would otherwise fight over the same touch events, plan Risk
3). `touch` mode (the other posture value) is the noVNC default passthrough
— this layer is inert when the posture is `touch`.

Unit tests with synthetic pointer/touch events: one-finger drag → relative
move sequence with the gain applied; tap under both thresholds → click, tap
over either threshold → no click (correctly reclassified as a drag start);
two-finger tap → right-click; two-finger drag → wheel events with correct
sign/magnitude; long-press → mousedown held, released on finger-lift;
`dragViewport` is disabled while this layer is active and re-enabled when
switching back to `touch` mode.

### 4. The key bar — `src/components/gui-keybar.tsx` (new)

Coarse-pointer-only, docked as a one-row strip under the tile (hidden on
fine pointers): `Esc  Tab  Ctrl  Alt  ⇧  ←  ↑  ↓  →  ⌨`. Modifier keys
(`Ctrl`, `Alt`, `⇧`) are **latching**: one tap = held for the next
non-modifier key sent (then auto-released), two taps = locked (rendered
pressed, `Ctrl ●`, stays held across multiple subsequent keys until tapped a
third time to release), a third tap releases. Non-modifier keys (`Esc`,
`Tab`, arrows) send immediately via noVNC `sendKey`, composed with whatever
modifiers are currently latched/locked. `⌨` focuses a hidden `<input>` to
raise the platform's on-screen keyboard (the noVNC-UI trick — a focused,
visually-hidden text input is what triggers a mobile OS's soft keyboard),
forwarding the resulting keystrokes through `sendKey` rather than letting
the browser insert them into the hidden input's value.

Tests: vitest for the latch state machine (single tap → armed-for-one,
double tap → locked, subsequent tap while locked → released; sending a
non-modifier key while armed-for-one consumes the arm and returns to
unarmed); Playwright mobile: the key bar renders on a coarse viewport,
tapping `Ctrl` then a letter key sends one chord (asserted via the RFB
mock's `sendKey` calls in the ungated half of the suite).

### 5. Palette rows — `src/lib/palette/gui.ts`

`GUI: Zoom in` / `GUI: Zoom out` / `GUI: Zoom to fit` (mirroring the chords
in item 2); `GUI: Pointer → Trackpad` / `GUI: Pointer → Touch` (with the
descriptions from § UX: "one finger moves, tap clicks, two-finger tap
right-clicks, two-finger drag scrolls" / "tap where you touch"); `GUI: 1:1`
becomes an alias for the 100% zoom step rather than noVNC's separate
`clipViewport` mode (V2's zoom mechanism subsumes it — kept as a named row
for continuity per § UX).

### 6. Tests (summary; see per-item detail above)

Vitest: posture helpers, zoom step table, pointer translation (tap,
two-finger tap, drag → wheel), key bar latch states. Playwright mobile
375 px (`hasTouch`): key bar renders, `Ctrl`+letter sends one chord, pinch
changes the zoom badge. Playwright desktop: Ctrl+wheel steps zoom, badge
shows `125%`.

### 7. Docs

`docs/specs/gui.md` § The tile — zoom (postures, chords, badge), pointer
modes (`touch`/`trackpad` semantics), key bar (latching modifiers, `⌨`
behavior). Memory via hydrate (§ Affected Memory below).

### Acceptance (from the plan, binding)

- On a phone against this VM's daemon: trackpad mode hits a 12-px IceWM
  close button; two-finger drag scrolls a terminal; `Ctrl` + `c` interrupts
  a running command; pinch reaches 200% and pans.
- On the laptop: Ctrl+= steps to 125% with the badge shown.
- `just test` green.

## Affected Memory

- `run-kit/gui`: (modify) `rk-gui-zoom`/`rk-gui-pointer` postures; the zoom
  application mechanism chosen at apply time (`Display.scale` vs CSS
  transform — recorded once decided); the trackpad translation layer and
  its disabling of noVNC's `dragViewport`; the key bar's latching-modifier
  model; new Design Decision (whichever zoom mechanism the apply-time spike
  picks, with the rejected alternative noted)
- `run-kit/ui/lenses-and-layout`: (modify) § GUI Surface — zoom, pan, pointer
  modes, the key bar as tile-level chrome
- `run-kit/ui/keyboard-and-palette`: (modify) § The `GUI:` palette family —
  the zoom/pointer rows and the new Ctrl+wheel/Ctrl+=/Ctrl+-/Ctrl+0 chords in
  the Shortcuts tab

## Impact

**Code (`app/frontend/src/`)**: `lib/gui-posture.ts` (two new postures),
`components/gui-surface.tsx` (zoom application, badge, pan, chords, pinch),
`components/gui-pointer.ts` (new — trackpad translation layer),
`components/gui-keybar.tsx` (new — key bar), `lib/palette/gui.ts` +
`app.tsx` (five/six new rows) — each with vitest coverage; new Playwright
specs (one ungated for chords/postures, one mobile-tagged for touch/pinch/
key-bar).

**Contracts that change shape**: none server-side — this change is entirely
frontend/client-state. No API route, stream field, or Go type changes.

**Behavior change for existing hosts**: none by default for a fine-pointer
desktop viewer beyond the new zoom chords being available (opt-in via
keypress, no change to default rendering); a coarse-pointer (phone) viewer
gets `trackpad` as its new default pointer mode instead of today's direct
`touch` passthrough — a deliberate, documented behavior change (V-D7),
reversible per-viewer via `GUI: Pointer → Touch`.

**Tests**: vitest throughout; Playwright desktop (Ctrl+wheel) and mobile
(`hasTouch`, pinch, key bar, trackpad taps) specs, both with intent comments
per Constitution § Test Intent Comments. Gates: frontend vitest + Playwright
suites, `just test`.

**Dependencies**: none new — uses the already-pinned noVNC version's
existing (if partly undocumented) API surface; no new npm package.

## Open Questions

None as intake-blocking questions. V-D6's zoom mechanism is explicitly
marked Likely by the plan and is this change's own apply-time spike (item
2) — not an open question, a scoped decision point the apply-entry agent
resolves and records per the SRAD "apply decides and records" rule. The
exact pinch-distance-to-zoom-step mapping is left as an implementation
detail (Tentative, below) rather than a question, since any reasonable
threshold satisfies the acceptance criterion ("pinch reaches 200% and
pans").

## Assumptions

| # | Grade | Decision | Rationale | Scores |
|---|-------|----------|-----------|--------|
| 1 | Certain | `rk-gui-zoom` (`fit` \| `{50,75,100,125,150,200}`) and `rk-gui-pointer` (`touch`\|`trackpad`) are per-viewer localStorage postures, not host state | Plan V-D6/V-D7 verbatim — zoom/pointer mode are how *this* screen shows the shared desktop | S:90 R:85 A:90 D:90 |
| 2 | Certain | Default pointer mode is `trackpad` on coarse pointers, `touch` on fine | Plan V-D7 verbatim | S:90 R:90 A:90 D:90 |
| 3 | Tentative | Zoom rendering mechanism (`Display.scale` via the RFB instance vs. CSS `transform: scale()` + own pan) is undecided at intake — the apply-entry agent spikes both against the pinned noVNC version and records the choice as a plan Design Decision | Plan V-D6 marks this row explicitly Likely; the plan itself instructs "the intake spikes both … and picks one" — this is the SRAD apply-decides-and-records case, not an intake question | S:40 R:45 A:35 D:35 |
| 4 | Certain | Trackpad translation layer disables noVNC's own `dragViewport` gesture handling while active; re-enables it when switching back to `touch` | Plan V-D7 + Risk 3 verbatim — the two gesture systems fight over the same touch events otherwise | S:85 R:80 A:85 D:85 |
| 5 | Certain | Modifier keys in the key bar are latching: one tap = armed-for-one, two taps = locked, next tap while locked = released | Plan V-D8 verbatim | S:85 R:85 A:85 D:85 |
| 6 | Certain | `⌨` raises the on-screen keyboard via a hidden focused input (the noVNC-UI trick), forwarding keystrokes through `sendKey` rather than the input's own value | Plan V-D8 verbatim, naming the specific mechanism | S:80 R:80 A:80 D:80 |
| 7 | Certain | `GUI: 1:1` becomes an alias for the 100% zoom step, replacing (not living beside) noVNC's separate `clipViewport` 1:1 mode | Plan § UX: "`GUI: 1:1` alias of 100% (kept for continuity)" — V2's zoom mechanism subsumes the old 1:1 mode | S:80 R:75 A:80 D:80 |
| 8 | Tentative | Pinch-distance-to-zoom-step mapping (the exact threshold that advances one step) is an implementation detail; any reasonable threshold satisfying "pinch reaches 200% and pans" is acceptable | Plan states the acceptance outcome, not the exact gesture-to-step formula | S:35 R:80 A:40 D:35 |
| 9 | Certain | Ctrl+wheel/Ctrl+=/Ctrl+-/Ctrl+0 register through the existing keybinding registry (visible in the Shortcuts tab) and reclaim focus over noVNC's canvas handler like other tile chords | Plan V2 Do item 2 names this mechanism explicitly, matching the existing chord-registration convention other tile features use | S:75 R:80 A:85 D:75 |
| 10 | Confident | This change touches `src/lib/palette/gui.ts` only in the shared-file sense (adding new rows beside S3's), never modifying S3's desktop-picker control | Combined-execution plan's stage description: "adopts the S3 control if the geometry row wants it, otherwise touches S3 only in the palette file" — this plan's own V-D6/V-D7 rows are pure client postures with no server-discovered option set, so no adoption is warranted | S:65 R:80 A:75 D:65 |

10 assumptions (7 certain, 1 confident, 2 tentative, 0 unresolved).
