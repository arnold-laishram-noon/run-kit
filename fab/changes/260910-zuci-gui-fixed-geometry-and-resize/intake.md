# Intake: Fixed geometry — `gui.geometry`, `rk gui resize`, the resize endpoint, the palette rows (S2 / V1)

**Change**: 260910-zuci-gui-fixed-geometry-and-resize
**Created**: 2026-09-10

## Origin

> Drafted via `/fab-draft` as stage S2 of the combined GUI execution queue
> (`fab/plans/sahil/26-09-10-gui-combined-execution.md`). Implements § V1
> ("Fixed geometry: `gui.geometry`, `rk gui resize`, the resize endpoint, the
> palette rows") of `fab/plans/sahil/26-09-10-gui-viewer-ergonomics.md`. This
> is the plan's own D7-amending stage — it changes the parent surface plan's
> resize-policy decision, not merely adds to it. One-shot, no conversational
> discussion — the design record is the ergonomics plan's § Decision log
> (V-D1 through V-D5, binding) and its § UX / § V1 Do / § V1 Acceptance
> sections, reproduced here in full.

Waits for S1 (`gui-session-starters-and-wm-verb`) merged — the combined-
execution queue places S2 second because the `app/backend/cmd/rk/gui.go`,
`cmd/rk/gui_supervise.go`, and `docs/site/skill/gui.md` files S1 touches are
shared files S2 also touches (Shared-files list in the combined-execution
plan). S2 rebases over S1's already-trimmed skill page.

**Coordination point 1 is binding on this change**: the settings-dialog
choice control (a "string with server-supplied choices" select) belongs to
S3 (`gui-desktop-picker`), not to this change. This change ships
`gui.geometry` as a **plain validated text field** in the Settings dialog
(riding the existing generic `ui: true` string-row renderer) plus its own
palette rows and docs — it does **not** build or modify the dialog's control
model, and does not wait for S3 (S2 merges before S3 starts per the queue
order, so this ordering is moot in practice, but the scope boundary matters
regardless of order: a later stage, S7 or a micro-fix, may switch the
geometty row to S3's picker control once it exists).

## Why

**The pain (today's D7 behavior).** `internal/gui/backend.go`'s Xvnc argv is
fixed `-geometry 1920x1080` at start with `-AcceptSetDesktopSize` on, but the
live desktop size *follows the focused fine-pointer viewer's tile* via
`gui-surface.tsx`'s `applyRfbProps`: `rfb.resizeSession =
!coarsePointer && focused && !resizeLocked && !hostLocked`. Every layout
change — toggling the sidebar, splitting a tile, zen-zooming — resizes the
guest desktop, so windows reflow, the taskbar re-wraps, and anything an
agent measured with `rk gui shot`/`rk gui click` moves under it. The
existing pins (`GUI: Lock resolution` viewer-local, `rk gui lock` host-side)
only pin *whatever size is current* — a stopgap, not a fix. On a phone the
coarse viewer never resizes by design, so it renders a 1920×1080 desktop
scaled into ~375 px: unreadable and untappable.

**The consequence of not fixing it.** Every downstream ergonomics stage
(S4's zoom/pointer work, S6's quality/stats, S7's toolbar) assumes a desktop
whose size is a deliberate, stable choice — not something that silently
moves every time a person resizes their browser window. Screenshot- and
click-based agent automation (`rk gui shot`, `rk gui click`) is unreliable
against a moving target.

**The approach, and why.** Make the desktop size a **host preference**,
fixed by default; the follow-the-tile behavior becomes the explicit `auto`
value of a new registry key `gui.geometry` (V-D1) — the exact behavior D7
already implements, just opt-in instead of the only behavior. Presets plus
custom (V-D2): `1280x720`, `1600x900`, `1920x1080` (default, unchanged from
today), `2560x1440`, and a portrait preset `1080x1920` for the phone-first
session; `Custom W×H` accepts 320–7680 per side. Apply live via RandR
(`xrandr`), never a restart (V-D3) — the spike already proved
`-AcceptSetDesktopSize` plus `xrandr --output <first-connected> --mode WxH`
reflows the WM live with nothing killed. When `gui.geometry` is not `auto`,
`resizeSession` is false for every viewer (V-D4) — the existing lock pins
stay meaningful under `auto` (D7's escape hatches) and read as disabled
descriptions under a fixed geometry, never disappearing. Aspect ratio is
preserved by construction (V-D5): `scaleViewport` fit mode already
letterboxes uniformly; a new `Match this tile` row does a one-shot resize
to the closest-aspect preset rather than re-enabling continuous follow.

Alternatives the plan rejected and this intake does not reopen: server-side
framebuffer scaling (V-D14, out of scope — belongs to a later C6 change);
per-viewer desktop sizes (impossible — one shared desktop, V-D1's whole
point); reopening D1–D6 or D8–D10 of the parent surface plan.

## What Changes

Backend (`app/backend/`) and frontend (`app/frontend/`) plus docs.

### 1. The `gui.geometry` settings key — `app/backend/internal/settings/settings.go`

Placed immediately after `gui.wm` (S1's territory, now on main):

```go
{
    key: "gui.geometry", kind: "string", def: "1920x1080",
    desc:     "The GUI desktop's pixel size. Fixed sizes keep windows where they are; Auto follows the focused desktop viewer's tile. Applies live.",
    category: "behavior", ui: true, live: true,
    parse:     quoteTrimmedScalar(func(s *Settings) *string { return &s.GUIGeometry }),
    serialize: quotedScalar("gui.geometry", func(s *Settings) *string { return &s.GUIGeometry }),
    read:  func(s *Settings) any { return s.GUIGeometry },
    apply: <the string-scalar apply helper existing rows use>,
},
```

`Settings.GUIGeometry string` beside `GUIWM`. Validator accepts the literal
string `"auto"` or a `WxH` shape with both W and H in 320–7680 inclusive;
anything else is a validation error at `Save`/apply time. Round-trip tests:
default `1920x1080` omitted on serialize (matches the existing
omit-when-default convention), `auto` and a custom `WxH` both round-trip.
The settings inventory count used by any test asserting a fixed total (15
after S1's `gui.wm` addition) becomes 16.

**`ui: true` rendering** — per Coordination point 1, this key rides the
**generic string-row renderer** the All-settings panel already uses for
`ui: true` string fields (the same renderer `gui.wm` uses today, per S1).
This change does **not** add an `options:` hint or any select/picker
metadata to the registry row — that richer control model is S3's to build,
for the choice it actually needs (server-discovered candidates). A plain
text field here is sufficient for V1's acceptance and keeps this change's
surface to "one settings key," not "one settings key plus a new control
kind."

### 2. Geometry parsing and RandR argv — `internal/gui/geometry.go` (new)

```go
// ParseGeometry accepts "auto" (auto=true, w=h=0) or "WxH" with w,h in
// 320-7680 inclusive; anything else is an error naming the accepted shapes.
func ParseGeometry(s string) (w, h int, auto bool, err error)

// XrandrResizeArgv builds the argv for a live resize on display: when the
// mode WxH is not already listed by xrandr, --newmode/--addmode first
// (mode line computed via the standard CVT/GTF-free simple formula the
// spike used), then --output <output> --mode WxH. output is never
// hardcoded — see XrandrQueryOutput.
func XrandrResizeArgv(display, output string, w, h int) [][]string

// XrandrQueryOutput parses `xrandr --query` (run against display) for the
// first output reported "connected" — VNC-0 on TigerVNC per the spike, but
// probed rather than assumed (V-D3's Likely row: the exact incantation for
// arbitrary custom modes on Xtigervnc was spike-verified for 536x799 only;
// this change re-verifies 1080x1920 and 2560x1440 before shipping).
func XrandrQueryOutput(ctx context.Context, display string) (string, error)
```

Tests with a fixture `xrandr --query` output (parses the first connected
output; a query with no connected output is an error); `XrandrResizeArgv`
for a mode already in the `xrandr` fixture's mode list (skips
`--newmode`/`--addmode`) and for a mode not listed (includes them);
`ParseGeometry` table test (auto, valid WxH at each boundary 320/7680,
below/above bounds, garbage string).

### 3. Supervisor start-time geometry — `cmd/rk/gui_supervise.go`

`-geometry` argv reads `gui.geometry` at supervise start: `auto` resolves to
`1920x1080` (today's default, unchanged bytes-on-the-wire), any other value
is used verbatim (already validated at settings-apply time, so no re-parse
error path needed here). Log line: `gui: desktop 1600x900 (gui.geometry)`
for a fixed value, `gui: desktop 1920x1080 (auto — follows the focused
viewer)` for `auto`. This is a start-time read only — the live-resize path
(item 5 below) is a separate, additive mechanism that does not touch the
supervisor.

Test: the geometry argv and log line for both branches (fixed value; auto).

### 4. Stream and status document — `internal/daemon/gui.go`, `api/sse.go`

Both the SSE stream entry and the status document gain `geometry`
(`"WxH"` or `"auto"`), read from the `gui.geometry` setting each tick — **no
new stamp**: the setting itself is the source of truth (Constitution II),
unlike `wm` which needs a supervisor-side stamp because it depends on what
actually resolved. The actual live pixel size still rides the existing
`width`/`height` fields from the RFB probe — `geometry` and `width`/`height`
can disagree transiently mid-resize, which is expected and matches how
`wm`/pin already can differ from the resolved binary.

Test: stream payload and status document both carry `geometry` reflecting
the current setting value.

### 5. The resize endpoint — `api/gui.go` + `router.go`

`POST /api/gui/{id}/resize`, body `{"geometry":"WxH"|"auto"}` (Constitution
IX — POST, no new verb shape):

| Condition | Response |
|---|---|
| id/body shape invalid, or geometry out of 320–7680 range | 400 |
| gui disabled or not reachable | 409 |
| `geometry` is `auto` | 200, persists the setting only (no xrandr call) |
| `geometry` is `WxH` | runs `xrandr` under a 10 s bound (Constitution: Process Execution — `exec.CommandContext` with timeout), then persists the setting; a failed xrandr leaves the setting **unwritten** and returns 500 with the command's stderr tail |
| success | `200 {"ok":true,"geometry":"1600x900","was":"1920x1080"}` |

Handler test with lookPath/exec seams covering every row above.

### 6. `rk gui resize` — `cmd/rk/gui.go`

Per § UX:

```
$ rk gui resize 1600x900
resized :10 to 1600x900 (was 1920x1080)
$ rk gui resize auto
desktop follows the focused viewer (gui.geometry=auto)
$ rk gui resize 100x100
Error: geometry 100x100 out of range (320–7680 per side)
$ rk gui status
gui: on (Xtigervnc, :10, 1600x900 fixed, 1 viewer, icewm-session)
```

Gated like `rk gui launch` (`guiDarwinRefusal`, then `guiRequireReachable`).
`guiStatusSummary` appends `fixed`/`auto` to the geometry segment (reading
the live setting, not a stamp); the doctor row inherits the same rendering.
The stream entry gains `"geometry":"1600x900"` (or `"auto"` when following),
per item 4.

Test: the three example invocations above via the CLI's exec/lookPath
seams; the status summary's `fixed`/`auto` suffix.

### 7. Frontend API client — `app/frontend/src/api/client.ts`

`GuiSignal.geometry: string`, `GuiStatus.geometry: string`,
`resizeGui(geometry: string): Promise<...>` calling the new endpoint. Both
new optional-safe fields (existing consumers ignore unknown JSON keys).

### 8. `resizeSession` gains the `auto` gate — `src/components/gui-surface.tsx`

`resizeSession = !coarsePointer && focused && !resizeLocked && !hostLocked
&& geometry === "auto"` — the single added clause. Fit mode already
letterboxes uniformly (`scaleViewport`), so no other rendering change is
needed; header doc comment updated to explain the new gate.

Test: vitest truth table for `resizeSession` across the `geometry` values
(`auto` preserves today's four-clause behavior; any fixed value forces
`false` regardless of the other three clauses).

### 9. Palette rows — `src/lib/palette/gui.ts` + `app.tsx`

Per § UX "Resolution (V1)" table:

| Row | Does |
|---|---|
| `GUI: Resolution → 1280×720` … `→ 2560×1440`, `→ 1080×1920 (portrait)` | `POST /api/gui/host/resize {"geometry":"1280x720"}`; the current value's row carries description `current` |
| `GUI: Resolution → Match this tile` | picks the closest-aspect preset to the focused gui tile's pixel size and posts it (one-shot, not a follow) |
| `GUI: Resolution → Custom…` | one-field prompt `Width×Height`, reusing the existing one-field prompt component (the session-name prompt shape); accepts `1440x900`, `1440×900`, `1440 900`; validates 320–7680 client-side before posting |
| `GUI: Resolution → Auto (follow this tile)` | posts `auto`; description `today's behavior — the desktop follows the focused fine-pointer viewer` |
| `GUI: Lock resolution` / `Unlock resolution` | unchanged under `auto`; under a fixed geometry the row is **disabled** with description `resolution is fixed (1920×1080) — pick Auto to follow the tile` (V-D4 — the row never disappears) |

All rows hidden on the mirror backend (macOS), gated `enabled && reachable`
per the existing palette convention. This change does **not** add the
select-control version of this row to the Settings dialog (Coordination
point 1) — only the palette rows and the plain text field.

Tile behavior: with a fixed desktop, fit mode letterboxes on the stage
ground; no toast on resize; the desktop visibly reflows (no debounce/hide
needed — the resize is user-initiated, not continuous).

### 10. Tests

- Go: items 2–6 above, each with its own unit test.
- Vitest: item 8's `resizeSession` truth table; the palette rows' gating and
  request bodies; the settings round-trip.
- Playwright `gui-surface.spec.ts` (ungated, stubbed stream): `geometry:
  "1600x900"` ⇒ the Lock row renders disabled with the exact copy;
  `GUI: Resolution → 1280×720` posts the exact expected body.
- Playwright real-rig (Xtigervnc-gated, intent comments per Constitution §
  Test Intent Comments): `rk gui resize 1280x720` ⇒ the status document's
  `width`/`height` follow within 5 s and the tile canvas letterboxes.

### 11. Docs

- `docs/specs/gui.md` § Resize policy — rewritten: D7 becomes the `auto`
  value of `gui.geometry`; the pins' meaning is stated per value (V-D4);
  § Agent verbs gains `resize`; § The switch gains the new settings row
  mention.
- `docs/site/skill/gui.md` — **adds without trimming** (per Coordination
  point 2, S1 already did the trim): `rk gui resize` usage, and the note
  that a fixed desktop is what keeps `rk gui shot`/`rk gui click` coordinates
  stable. At most one new line net beyond the `resize` mention, staying
  comfortably under the 150-line cap S1 restored headroom for.
- The parent surface plan (`26-09-09-gui-surface.md`)'s D7 row gets a
  pointer to this plan (per the ergonomics plan's own Pickup protocol item
  4) in the same PR.
- Memory via hydrate (§ Affected Memory below).

### Acceptance (from the plan, binding)

- On this VM, `rk gui resize 1600x900` reflows IceWM live with no app
  killed.
- `rk gui status` reads `1600x900 fixed`.
- The tile letterboxes; toggling the sidebar no longer changes the desktop.
- `GUI: Resolution → Auto` restores the follow behavior and the Lock rows
  re-enable.
- `rk gui restart` comes back at the chosen size.
- The portrait preset renders on a 375-px phone as a readable full-width
  desktop.
- `just test` green.

## Affected Memory

- `run-kit/gui`: (modify) `gui.geometry` settings key and its `auto`/`WxH`
  semantics; the D7 policy rewrite (auto = today's follow behavior, fixed =
  the new default); `rk gui resize` and `POST /api/gui/{id}/resize`;
  `resizeSession`'s new `geometry === "auto"` gate; the Lock rows' disabled-
  under-fixed-geometry behavior; new Design Decision (the desktop size is a
  host preference, not a per-viewer follow, by default)
- `run-kit/configuration`: (modify) § Settings Registry gains `gui.geometry`
  (string, default `1920x1080`, behavior, ui yes, live yes); inventory count
  15 → 16
- `run-kit/api-and-sockets`: (modify) `/api/gui/*` route list gains
  `POST /api/gui/{id}/resize` with its status-code table; the `event: gui`
  slot's payload gains `geometry`
- `run-kit/ui/lenses-and-layout`: (modify) § GUI Surface — the tile's
  letterbox behavior under a fixed geometry, `Match this tile`
- `run-kit/ui/keyboard-and-palette`: (modify) § The `GUI:` palette family —
  the five new `Resolution →` rows

## Impact

**Code (`app/backend/`)**: `internal/settings/settings.go` (`gui.geometry`
row + `Settings.GUIGeometry`), `internal/gui/geometry.go` (new),
`cmd/rk/gui_supervise.go` (start-time geometry), `internal/daemon/gui.go` +
`api/sse.go` (stream field), `api/gui.go` + `router.go` (resize handler),
`cmd/rk/gui.go` (`resize` verb, status summary) — each with tests.

**Code (`app/frontend/src/`)**: `api/client.ts` (types + `resizeGui`),
`components/gui-surface.tsx` (`resizeSession` gate), `lib/palette/gui.ts` +
`app.tsx` (five palette rows) — each with tests; one Playwright spec
addition (ungated) plus one real-rig case (Xtigervnc-gated).

**Contracts that change shape** (additive): `gui.StreamEntry` and
`gui.Status` gain `geometry`; a new `POST /api/gui/{id}/resize` route;
`Settings` gains `GUIGeometry`. No signature of an existing exported
function changes (unlike S1's `GUISessionOptions` widening) — this is
additive-only.

**Behavior change for existing hosts**: the desktop stops following the
focused viewer's tile by default (V-D1's whole point) — the plan's Risk 4
notes the letterbox-bars-where-the-tile-used-to-fill regression this can
cause on a first laptop viewing, mitigated by `Match this tile` being one
row away and `Auto` restoring D7 exactly. The default value `1920x1080`
matches today's `-geometry` argv, so a host that never resizes its viewer
sees no visual change at all.

**Tests**: Go unit tests throughout; vitest for the frontend truth table and
palette rows; one ungated Playwright spec; one Xtigervnc-gated real-rig
case. Gates: `cd app/backend && go test ./...`, `just test`, `just build`,
frontend vitest + Playwright suites.

**Dependencies**: none new. `xrandr` is already on PATH per the parent
plan's spike verification (`internal/gui/backend.go`'s existing use); this
change is the first to invoke it for an arbitrary custom mode beyond the
spike's single verified case.

## Open Questions

None. The ergonomics plan's § Decision log resolves every design point for
V1 and its § UX fixes the exact copy. V-D3's exact `xrandr` incantation for
1080×1920 and 2560×1440 is marked Likely by the plan and is explicitly
listed as this stage's own re-verification task (item 2/11's acceptance),
not an open question — recorded as a Confident assumption below with its
fallback (the `--newmode`/`--addmode` path already scoped in item 2).

## Assumptions

| # | Grade | Decision | Rationale | Scores |
|---|-------|----------|-----------|--------|
| 1 | Certain | `gui.geometry` (string, default `1920x1080`, `ui: true`, `live: true`); `auto` or `WxH` with 320–7680 per side; Xvnc argv reads it at supervise start | Plan V-D1 verbatim; matches today's default byte-for-byte so no visual change for a host that never touches it | S:95 R:75 A:95 D:95 |
| 2 | Certain | Presets `1280x720`, `1600x900`, `1920x1080`, `2560x1440`, `1080x1920` (portrait); `Custom W×H` 320–7680; presets are frontend-only, backend validates shape/bounds only | Plan V-D2 verbatim | S:90 R:85 A:95 D:90 |
| 3 | Certain | Live apply via `xrandr` (`--newmode`/`--addmode` fallback then `--output <probed> --mode WxH`), never a restart; output name probed via `xrandr --query`, never hardcoded | Plan V-D3; `-AcceptSetDesktopSize` already on, spike proved live RandR reflow | S:85 R:70 A:85 D:85 |
| 4 | Confident | The exact `xrandr` incantation for 1080×1920 and 2560×1440 on Xtigervnc is re-verified during apply (spike only verified 536×799); a mode not in the current list gets the `--newmode`/`--addmode` fallback | Plan marks this row Likely explicitly; the fallback path is already scoped, only the "already listed vs needs newmode" branch differs per mode | S:60 R:80 A:60 D:60 |
| 5 | Certain | Under any non-`auto` geometry, `resizeSession` is false for every viewer; the viewer-local and host-side Lock pins stay meaningful only under `auto` and render as a disabled description row otherwise (never removed) | Plan V-D4 verbatim | S:90 R:80 A:90 D:90 |
| 6 | Certain | Aspect ratio preserved by construction — `scaleViewport` fit mode, never stretched; `Match this tile` is a one-shot resize to the closest-aspect preset, not a continuous follow | Plan V-D5 verbatim | S:90 R:85 A:90 D:90 |
| 7 | Certain | The Settings-dialog control for `gui.geometry` is a **plain validated text field** (the existing generic `ui: true` string-row renderer); the richer select/picker control belongs to S3 and is out of scope here | Combined-execution plan Coordination point 1 — explicit stage boundary | S:90 R:80 A:90 D:90 |
| 8 | Certain | The resize endpoint persists the setting only on success; a failed `xrandr` call leaves `gui.geometry` unwritten and returns 500 with the command's stderr tail | Plan V1 Do item 5; keeps the setting truthful to what actually happened on the display | S:80 R:85 A:85 D:80 |
| 9 | Confident | `geometry` and the live `width`/`height` stream fields may transiently disagree mid-resize; no new stamp is introduced for geometry (unlike `wm`, which needs a supervisor-side stamp) | Constitution II — state derived at request time; the setting itself already is the source of truth for the target size | S:70 R:85 A:80 D:70 |
| 10 | Confident | `Custom…` prompt reuses the existing one-field prompt component (the session-name prompt shape) rather than a new dialog | Plan V1 Do item 9 names this component explicitly; consistent with existing palette prompt patterns | S:65 R:90 A:80 D:70 |

10 assumptions (7 certain, 3 confident, 0 tentative, 0 unresolved).
