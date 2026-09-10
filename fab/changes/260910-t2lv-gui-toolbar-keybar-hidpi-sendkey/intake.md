# Intake: Session toolbar, HiDPI, and Send key (S7 / V4)

**Change**: 260910-t2lv-gui-toolbar-keybar-hidpi-sendkey
**Created**: 2026-09-10

## Origin

> Drafted via `/fab-draft` as stage S7 of the combined GUI execution queue
> (`fab/plans/sahil/26-09-10-gui-combined-execution.md`). Implements § V4
> ("Session toolbar, HiDPI, Send key") of
> `fab/plans/sahil/26-09-10-gui-viewer-ergonomics.md`. One-shot, no
> conversational discussion — the design record is the ergonomics plan's
> § Decision log (V-D10, V-D12, V-D13, binding) and its § UX / § V4 Do /
> § V4 Acceptance sections, reproduced here in full.

Waits for S4 (`gui-zoom-and-touch-pointer`) merged, S6
(`gui-quality-presets-and-stats`) optional (the combined-execution plan
marks S6 as "optional" for S7's dependency — this change's toolbar pill
includes a quality control and a stats toggle per § UX, both of which S6
introduces; if S6 has not yet merged when S7 starts, the pill's quality and
stats controls degrade gracefully to no-ops or are stubbed pending S6,
rather than blocking S7 — see Assumption below). This is the **last** stage
in the queue: per the combined-execution plan, "the pill mirrors rows every
earlier stage created" — the toolbar is a thin wiring layer over controls
S1–S6 already built (resolution is S2's, zoom/pointer is S4's, quality/stats
is S6's), plus two genuinely new features this change introduces itself:
HiDPI and Send key.

## Why

**The pain.** Two contexts make the command palette unreachable or clumsy:
**fullscreen** (the palette's `Cmd+K` shortcut still technically works, but
a person in fullscreen watching an agent work does not want to break flow
to open a command menu for a quick zoom-in), and **a phone** (no physical
keyboard, so `Cmd+K` has no natural trigger, and even if triggered, typing a
search query on a phone to find "zoom in" is worse than a visible button).
There is also no way to render the tile crisply on a HiDPI (Retina) display
in 1:1 view — the canvas renders at CSS pixel density, not device pixel
density, so text and fine UI elements in the mirrored desktop look soft on
a 2x display. And there is no way to send a chord a browser keyboard cannot
express at all (`Ctrl+Alt+Del`, `Alt+F4` is interceptable but awkward,
`Super` may be captured by the OS).

**The consequence of not fixing it.** Every ergonomics improvement S2/S4/S6
shipped (resolution presets, zoom, pointer mode, quality, stats) is
reachable only through the palette or a keyboard shortcut — exactly the two
paths that do not work in fullscreen or on a phone. Without this change,
the phone experience V2 (S4) built stays half-finished: a phone user can
zoom and use trackpad mode, but can't discover *how* without already
knowing the palette exists (which does not work well on a phone anyway).

**The approach, and why.** A floating, auto-hiding session toolbar pill —
top-center of the tile, shown on tap (coarse) or on hover near the top edge
(fullscreen), auto-hidden after 3 s of inactivity — mirroring every palette
row that matters in these two contexts (V-D10): `− fit + ⌖ Trackpad ◐
Balanced ⌨ ∿ ⤢` (zoom out/fit/in, pointer-mode indicator+toggle, quality
indicator+toggle, key-bar toggle, stats toggle, exit fullscreen). Every one
of its controls is *also* a palette row (Constitution V — the palette is
the complete action registry); the pill is described explicitly as "the
coarse and fullscreen mirror," never a third, separate action surface — a
fine-pointer, non-fullscreen viewer never sees it at all. HiDPI is opt-in
(V-D12): `GUI: HiDPI on/off`, default off, renders the canvas at
`devicePixelRatio` for crisp 1:1 text on a Retina laptop, changing only
client-side rendering, never the desktop's actual pixel size (which stays
whatever S2's `gui.geometry` says). `GUI: Send key…` (V-D13) is a small
prompt with five suggested chords (`Ctrl+Alt+Del`, `Ctrl+Alt+T`, `Alt+F4`,
`Super`, `Print`) plus free typing, sent viewer-side through noVNC
`sendKey` — no server round trip, so it works even on the macOS
view-only mirror backend (where it is refused with the existing mirror
view-only message, matching every other input-sending action on that
backend).

Alternatives the plan rejected and this intake does not reopen: a
persistent (non-auto-hiding) toolbar on every viewer (V-D10 explicitly
scopes it to coarse/fullscreen only — a fine-pointer non-fullscreen viewer
already has the palette); server-side key injection for Send key (viewer-
side only, so it degrades gracefully on the mirror backend); HiDPI on by
default (V-D12 — opt-in, because it doubles bytes on exactly the links C5
flagged).

## What Changes

Frontend only (`app/frontend/`).

### 1. `GuiToolbar` pill — `src/components/gui-toolbar.tsx` (new)

Rendered only when: the viewer is coarse-pointer, **or** the tile is
currently in fullscreen (fine or coarse) — never for a fine-pointer,
non-fullscreen viewer. Auto-hide after 3 s of no interaction; reappears on
tap (coarse) or on pointer movement near the top edge (fullscreen, mirrors
how the existing fullscreen exit affordance already behaves, if one
exists — reuse that pattern rather than inventing a second hover-reveal
mechanism).

Six controls, each wired to the **same callback** the corresponding palette
row already calls (no duplicated logic — the pill and the palette are two
UIs over one action):

| Pill control | Wired to (existing palette action) |
|---|---|
| `−` / `fit` / `+` | `GUI: Zoom out` / `GUI: Zoom to fit` / `GUI: Zoom in` (S4) |
| `⌖ Trackpad` (or `⌖ Touch`, showing current mode) | `GUI: Pointer → Trackpad` / `→ Touch` (S4) — tapping toggles between the two |
| `◐ Balanced` (or the current quality name) | `GUI: Quality →` (S6) — tapping cycles Sharp → Balanced → Smooth → Sharp |
| `⌨` | toggles the S4 key bar's visibility (the key bar itself already exists; this is a visibility toggle, not a new key-sending mechanism) |
| `∿` | `GUI: Show stats` / `Hide stats` toggle (S6) |
| `⤢` | exit fullscreen (existing fullscreen mechanism) |

If S6 has not merged when this change's apply runs (see the dependency note
in § Origin), the `◐`/`∿` controls degrade to a disabled/hidden state rather
than throwing — a defensive check against the posture/action existing,
matching how the codebase already guards against optional features. Once
S6 is on `main` (which the combined-execution queue guarantees by the time
S7 actually starts, since S6 precedes S7 in queue order regardless of being
marked "optional" for dependency purposes — the "optional" marking is about
whether S7 could in principle start without S6, not about the actual queue
order), the controls are simply live.

Test (vitest): the pill renders in exactly the two contexts (coarse;
fullscreen) and not in the third (fine, non-fullscreen); each control's tap
calls the identical handler its palette-row counterpart calls (assert via a
shared handler spy, proving no duplicated logic); auto-hide timing (fake
timers); reveal on tap/hover-near-edge.

### 2. HiDPI posture — `src/lib/gui-posture.ts` + `src/components/gui-surface.tsx`

`rk-gui-hidpi`: boolean, default `false`. When true, the canvas backing
store renders at `window.devicePixelRatio` instead of `1` — a rendering-only
change (canvas resolution, CSS size unchanged) that never touches
`gui.geometry` or any server-side desktop size. `GUI: HiDPI on` / `off`
palette rows.

Test: vitest for the posture default/round-trip; the canvas backing-store
dimension calculation at `devicePixelRatio` values `1`, `2`, and a
fractional value (e.g. `1.5`) each rendering the expected pixel dimensions
for a known CSS size.

### 3. `GUI: Send key…` — `src/components/gui-send-key.tsx` (new, or reusing the existing one-field prompt component per S2's precedent)

A tiny prompt offering five suggested chords as quick-pick options
(`Ctrl+Alt+Del`, `Ctrl+Alt+T`, `Alt+F4`, `Super`, `Print`) plus a free-typed
chord field, reusing the same one-field prompt shape S2's `Custom…`
resolution prompt already established (consistency, not a new prompt
component). Selecting or typing a chord sends it through noVNC `sendKey` —
purely client-side, no server round trip — so on the macOS mirror backend
(view-only) it is refused with the **existing** mirror view-only message
(the same refusal every other input-sending action already shows there,
reused verbatim, not a new message string).

Test: vitest for each of the five suggested chords producing the correct
`sendKey` call sequence (multi-key chords send the correct key-down/key-up
ordering); free-typed chord parsing (`Ctrl+Alt+F4`-style syntax); the
view-only refusal on the mirror backend.

### 4. Tests (summary; see per-item detail above)

Vitest per surface (pill, HiDPI, Send key). Playwright mobile: the pill's
show/hide behavior and one tap-through (e.g. tapping `+` zooms in,
verifiable via the RFB mock or the resulting `rk-gui-zoom` posture value).
Intent comments per Constitution § Test Intent Comments.

### 5. Docs

`docs/specs/gui.md` § The tile: the toolbar pill's two-context rule, HiDPI's
opt-in rendering-only behavior, Send key's chord list and viewer-side
mechanism. Memory via hydrate (§ Affected Memory below). This change also
updates both parent plans' final bookkeeping per the combined-execution
plan's § Done means: `docs/specs/gui.md` § Resize policy reads the
`auto`-value form of D7 (already done by S2 — this change verifies it is
still accurate, no re-edit expected) and § Switching desktops exists
(already done by S1/S3/S5); the parent plan's D7 row carries the pointer to
the ergonomics plan (already done by S2); the skill page is ≤150 lines
(verified, not re-trimmed, since S1 did the trim and every later stage added
at most one line per the combined-execution plan's rule).

### Acceptance (from the plan, binding)

- In fullscreen on the laptop, hovering the top edge shows the pill and `⤢`
  exits.
- On the phone the pill's `−`/`+` step zoom.
- `Alt+F4` from Send key closes the focused IceWM window.
- HiDPI on a `devicePixelRatio` 2 laptop renders 1:1 text crisp and doubles
  the overlay's Mbit/s (expected, documented — not a bug, since HiDPI
  doubles the actual pixel data sent).

## Affected Memory

- `run-kit/gui`: (modify) the `GuiToolbar` pill and its two-context
  visibility rule (coarse or fullscreen, never both-absent conditions);
  `rk-gui-hidpi` posture and its rendering-only scope; `GUI: Send key…` and
  its viewer-side `sendKey` mechanism, including the mirror-backend
  refusal; new Design Decision (every pill control mirrors an existing
  palette action — no toolbar-only functionality)
- `run-kit/ui/lenses-and-layout`: (modify) § GUI Surface — the toolbar pill
  as the final piece of tile-level chrome, its auto-hide timing
- `run-kit/ui/keyboard-and-palette`: (modify) § The `GUI:` palette family —
  `GUI: HiDPI on/off` and `GUI: Send key…` rows

## Impact

**Code (`app/frontend/src/`)**: `components/gui-toolbar.tsx` (new),
`lib/gui-posture.ts` (`rk-gui-hidpi`), `components/gui-surface.tsx` (HiDPI
canvas backing-store scaling), `components/gui-send-key.tsx` (new or
reused prompt), `lib/palette/gui.ts` + `app.tsx` (two new rows: `HiDPI
on/off`, `Send key…`) — each with vitest coverage; one Playwright mobile
spec.

**Contracts that change shape**: none — entirely frontend/client-state,
wiring existing actions into a new visual surface plus two genuinely new
client-only features (HiDPI rendering, Send key). No API route, stream
field, or Go type changes.

**Behavior change for existing hosts**: none by default — the pill only
appears in the two contexts it is scoped to and every control it exposes
already exists as a palette row (opt-in discovery, not new behavior); HiDPI
is opt-in and off by default; Send key is a net-new capability with no
default-behavior change to anything else.

**Tests**: vitest throughout; one Playwright mobile spec (pill show/hide,
one tap-through). Gates: frontend vitest + Playwright suites, `just test`.

**Dependencies**: none new.

## Open Questions

None as intake-blocking. Whether the fullscreen hover-reveal mechanism
reuses an existing pattern or needs a small new implementation is left as a
Confident assumption below, resolved at apply time by reading the current
fullscreen-exit affordance's code rather than asked here.

## Assumptions

| # | Grade | Decision | Rationale | Scores |
|---|-------|----------|-----------|--------|
| 1 | Certain | The toolbar pill renders only when coarse-pointer OR fullscreen, never for a fine-pointer non-fullscreen viewer; every control mirrors an existing palette row's callback rather than duplicating logic | Plan V-D10 verbatim: "the pill is the coarse and fullscreen mirror … Fine-pointer non-fullscreen viewers never see it" | S:90 R:80 A:90 D:90 |
| 2 | Certain | HiDPI (`rk-gui-hidpi`, default off) changes only the client-side canvas rendering resolution, never `gui.geometry` or any server-side desktop size | Plan V-D12 verbatim | S:90 R:85 A:90 D:90 |
| 3 | Certain | `GUI: Send key…` sends viewer-side via noVNC `sendKey`, no server round trip, refused on the mirror backend with the existing view-only message | Plan V-D13 verbatim; reuses the existing mirror refusal rather than introducing new copy | S:85 R:80 A:90 D:85 |
| 4 | Confident | If S6 has not yet merged when this change's apply begins, the pill's quality/stats controls degrade to a disabled/hidden state rather than blocking the rest of the change | The combined-execution plan marks S6 "optional" for S7's dependency, implying S7 must tolerate S6's absence gracefully; the queue order in practice guarantees S6 precedes S7, so this is a defensive-only code path | S:55 R:75 A:65 D:55 |
| 5 | Certain | Every pill control is wired to the identical callback its palette-row counterpart already calls — verified by test via a shared handler spy | Constitution V: palette is the complete action registry; a toolbar-only action would violate this, so the implementation must share, not duplicate, logic | S:85 R:80 A:90 D:85 |
| 6 | Confident | `GUI: Send key…`'s prompt reuses S2's existing one-field prompt component shape rather than a wholly new dialog | Plan V4 Do: "reusing the one-field prompt shape with the five chords as suggestions" — names the pattern, not the exact component file | S:70 R:85 A:80 D:70 |
| 7 | Confident | The fullscreen hover-reveal mechanism for the pill reuses whatever pattern the existing fullscreen-exit affordance already uses, rather than a new hover-detection implementation | Consistency with existing fullscreen chrome; the plan does not specify the exact reveal mechanism, only the behavior ("shown on tap or on hover in fullscreen") | S:60 R:80 A:70 D:60 |
| 8 | Certain | This is the queue's final stage — its docs task verifies (not re-performs) the combined-execution plan's § Done means bookkeeping that earlier stages (S1's skill-page trim, S2's D7 pointer, S1/S3/S5's § Switching desktops) already completed | Combined-execution plan § Done means lists these as conditions of the whole queue being finished, and each is explicitly assigned to an earlier stage | S:75 R:85 A:80 D:75 |

8 assumptions (4 certain, 4 confident, 0 tentative, 0 unresolved).
