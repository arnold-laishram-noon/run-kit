# Intake: Quality presets and the stats overlay (S6 / V3)

**Change**: 260910-5psw-gui-quality-presets-and-stats
**Created**: 2026-09-10

## Origin

> Drafted via `/fab-draft` as stage S6 of the combined GUI execution queue
> (`fab/plans/sahil/26-09-10-gui-combined-execution.md`). Implements § V3
> ("Quality presets and the stats overlay") of
> `fab/plans/sahil/26-09-10-gui-viewer-ergonomics.md`. One-shot, no
> conversational discussion — the design record is the ergonomics plan's
> § Decision log (V-D9, V-D11, binding) and its § UX / § V3 Do / § V3
> Acceptance sections, reproduced here in full.

Waits for S4 (`gui-zoom-and-touch-pointer`) merged — per the combined-
execution plan, S6 "shares the posture module and the toolbar seam with
S4": `src/lib/gui-posture.ts` already carries `rk-gui-zoom`/`rk-gui-pointer`
from S4, and this change adds `rk-gui-quality` beside them using the
identical read/write helper shape S4 established, rather than
re-implementing posture plumbing from scratch.

## Why

**The pain.** `applyRfbProps` today hardcodes two quality presets —
`qualityLevel`/`compressionLevel` = `(6,2)` for fine pointers, `(4,6)` for
coarse — with no way for a user to change them. There is also no visibility
into what the connection is actually doing: no fps counter, no bandwidth
reading, no RTT, so a person on a slow Tailscale link has no way to know
*why* the tile feels laggy or to trade sharpness for smoothness.

**The consequence of not fixing it.** The C5 verdict (parent surface plan)
already established that bytes-per-frame is what caps fps on the user's
link (≤ 40 Mbit/s observed), and that the coarse preset already ships ~40%
fewer bytes than fine — but that knowledge is invisible to the end user and
inaccessible to them as a control. C6 (a later, separate change) is
expected to swap in a better-tuned encoder behind these same three named
presets; without this change there is no user-facing seam for C6 to sit
behind, and no way to validate C6's claims against a live number.

**The approach, and why.** Quality becomes a user choice with three names
(V-D9): `GUI: Quality → Sharp / Balanced / Smooth` mapping to
`(qualityLevel, compressionLevel)` = `(8,1)` / `(6,2)` / `(3,7)` — default
`Balanced` on fine, `Smooth` on coarse (today's hardcoded presets, simply
renamed and now user-overridable). The names, not the numbers, are the
durable contract: C6 can later swap what `Smooth` maps to internally
(different encoder, different tuning) without touching this change's rows.
A stats overlay, off by default (V-D11): fps (framebuffer updates/sec),
relay Mbit/s, RTT (via a periodic tiny ping round trip or the relay's WS
ping frame if it grows one), desktop size, zoom — the same three probes the
Playwright perf spec already computes for test purposes, lifted into a
product-facing overlay behind a `stats` seam on `GuiSurface`.

Alternatives the plan rejected and this intake does not reopen: a full
encoder swap (C6 — a separate, later change); making quality a host-wide
setting rather than per-viewer (V-D9 explicitly keeps it per-viewer, like
zoom/pointer mode); a persistent always-on overlay (V-D11 — off by default).

## What Changes

Frontend only (`app/frontend/`).

### 1. Quality posture — `src/lib/gui-posture.ts`

`rk-gui-quality`: `"sharp" | "balanced" | "smooth"` — per-viewer, using the
identical get/set/notify shape S4's `rk-gui-zoom`/`rk-gui-pointer` postures
established (module already carries that pattern after S4 merges). Default
resolution: `balanced` on fine pointers, `smooth` on coarse — matching
today's hardcoded split exactly, so a viewer that never touches the row
sees byte-for-byte the same behavior as before this change.

Mapping table (constant, not user-editable): `sharp` → `(8,1)`,
`balanced` → `(6,2)`, `smooth` → `(3,7)` — applied to noVNC's
`qualityLevel`/`compressionLevel` RFB properties the same place
`applyRfbProps` already sets them, replacing the hardcoded fine/coarse
branch with a posture-driven lookup.

Test: default resolution per pointer type matches today's values exactly
(regression guard); the three-entry mapping table; round-trip get/set.

### 2. Palette row — `src/lib/palette/gui.ts`

`GUI: Quality → Sharp | Balanced | Smooth`, each option's description per §
UX: `more detail, more bytes` / `default` / `fewer bytes, smoother motion on
slow links`. Gated `enabled && reachable`, hidden on the mirror backend —
matching the existing `GUI:` row conventions.

Test: the three rows render with the correct descriptions; selecting one
updates the posture and is reflected in `applyRfbProps`'s next apply.

### 3. Stats seam — `src/components/gui-surface.tsx`

`GuiSurface` exposes a `stats` object/hook to the overlay component:
- **fps**: framebuffer-update counter, via the same `Display.flip` hook the
  Playwright perf spec (`tests/e2e/gui-perf.spec.ts`) already wraps for test
  purposes — reused here as a product-facing counter rather than
  reimplemented.
- **Mbit/s**: bytes over the RFB's WebSocket, via the same wrap the perf
  spec uses (a `WebSocket` send/receive byte counter), sampled and
  reported as a rolling rate.
- **RTT**: a periodic (5 s) tiny no-op `POST /api/gui/{id}/ping` round trip,
  timed client-side — or the relay's own WS ping frame if a future relay
  change adds one (this change does not add a relay-level ping; it uses the
  HTTP round trip as the baseline mechanism, matching the plan's stated
  "or" — the WS-ping alternative is explicitly deferred to whichever change
  grows that capability, not built here).
- **Desktop size**: the existing `width`/`height` stream fields, already
  available to `GuiSurface`.
- **Zoom**: the `rk-gui-zoom` posture from S4, already available.

The perf spec keeps its own independent wraps (this change does not remove
or refactor `gui-perf.spec.ts`'s existing instrumentation) — the product
seam and the test seam are two independent consumers of the same
underlying noVNC hooks, deliberately not unified into one code path (the
perf spec's wraps predate this change and are test-only scaffolding; this
change adds a separate, minimal product-facing seam rather than risk
destabilizing the existing perf test by sharing implementation).

Test: vitest for the stats hook's counter math (fps from flip events over a
time window, Mbit/s from byte deltas, a fake-timer-driven RTT ping cycle).

### 4. Stats overlay — `src/components/gui-stats-overlay.tsx` (new, or inline in `gui-surface.tsx` if the codebase's existing overlay convention favors a co-located component — see Assumption below)

A monospace corner overlay (top-right per § UX) rendering:
`59 fps · 41 Mbit/s · 262 ms · 1920×1080 · fit` — reading the item 3 seam.
Toggled by `GUI: Show stats` / `GUI: Hide stats` palette rows, posture
`rk-gui-stats-visible` (boolean, default off — V-D11).

Test: vitest for the overlay's render given a stats snapshot; the toggle
rows' presence and the default-off posture.

### 5. Tests (summary; see per-item detail above)

Vitest throughout (posture, mapping table, palette rows, stats math, overlay
render). Playwright, real-rig (Xtigervnc-gated, since it needs an actual RFB
connection to produce real fps/byte counts): the overlay renders and its fps
counter increments over a short observation window against the real rig.
Intent comments per Constitution § Test Intent Comments.

### 6. Docs

`docs/specs/gui.md` § Smoothness: the three named presets as the user-facing
lever C6 later sits behind (explicitly framed as the seam C6 will reuse,
per the plan's "the names are what C6 later swaps a Kasm or Tight-tuned
encoder behind — the row survives the backend"). Memory via hydrate
(§ Affected Memory below).

### Acceptance (from the plan, binding)

- Switching to `Smooth` on the 260 ms/40 Mbit netem link (the C5 recipe from
  the parent surface plan) raises fps measurably over `Balanced`.
- The overlay's Mbit/s agrees with the perf spec's own measurement within
  10%.
- `just test` green.

## Affected Memory

- `run-kit/gui`: (modify) `rk-gui-quality` posture and its three-name
  mapping to `(qualityLevel, compressionLevel)`; the `stats` seam on
  `GuiSurface` and its four/five counters; the stats overlay and its
  `rk-gui-stats-visible` posture; new Design Decision (quality is named,
  not numeric, so C6 can swap the mapping behind stable names)
- `run-kit/ui/lenses-and-layout`: (modify) § GUI Surface — the stats overlay
  as tile-level chrome, off by default
- `run-kit/ui/keyboard-and-palette`: (modify) § The `GUI:` palette family —
  `GUI: Quality →` and `GUI: Show/Hide stats` rows

## Impact

**Code (`app/frontend/src/`)**: `lib/gui-posture.ts` (`rk-gui-quality`,
`rk-gui-stats-visible`), `components/gui-surface.tsx` (mapping-driven
`applyRfbProps`, the `stats` seam), `components/gui-stats-overlay.tsx` (new
or co-located), `lib/palette/gui.ts` + `app.tsx` (four new rows) — each with
vitest coverage; one Playwright real-rig spec (Xtigervnc-gated).

**Contracts that change shape**: none server-side beyond the existing
`POST /api/gui/{id}/ping` usage this change is the first consumer of (a
route the plan describes as possibly new — see Assumption below regarding
whether `/ping` already exists or is introduced here). No stream field or
Go type changes otherwise.

**Behavior change for existing hosts**: none by default — the quality
mapping's default resolution matches today's hardcoded fine/coarse split
exactly; the stats overlay is off by default (V-D11). A viewer who opts
into `Sharp`/`Smooth` or turns on stats gets the new behavior explicitly.

**Tests**: vitest throughout; one Playwright real-rig spec. Gates: frontend
vitest + Playwright suites, `just test`.

**Dependencies**: none new — reuses the existing noVNC `Display.flip`/
WebSocket wrap pattern the perf spec already established; if
`POST /api/gui/{id}/ping` does not yet exist, this change adds it as a
minimal no-op timing endpoint (see Assumption below).

## Open Questions

None as intake-blocking. Whether `POST /api/gui/{id}/ping` already exists
elsewhere in the codebase or must be added by this change is recorded as a
Confident assumption below (the plan's § UX describes it as "a periodic tiny
… round trip," consistent with either reading) rather than a question — the
apply-entry agent checks the current router and adds the endpoint only if
absent.

## Assumptions

| # | Grade | Decision | Rationale | Scores |
|---|-------|----------|-----------|--------|
| 1 | Certain | Three named quality presets (`Sharp`/`Balanced`/`Smooth`) mapping to fixed `(qualityLevel, compressionLevel)` tuples `(8,1)`/`(6,2)`/`(3,7)`; default resolution matches today's hardcoded fine/coarse split exactly | Plan V-D9 verbatim; the default-matches-today rule is what makes this change a pure addition, not a behavior change, for a viewer that never touches the row | S:90 R:80 A:90 D:90 |
| 2 | Certain | Quality is per-viewer (posture, not host setting), consistent with zoom/pointer mode from S4 | Plan V-D9: "Posture `rk-gui-quality`, per viewer" | S:85 R:85 A:90 D:85 |
| 3 | Certain | Stats overlay is off by default, toggled by `GUI: Show/Hide stats`, and shows fps/Mbit/RTT/size/zoom in a monospace top-right corner overlay | Plan V-D11 verbatim, § UX example line | S:85 R:85 A:85 D:85 |
| 4 | Confident | RTT is measured via a periodic `POST /api/gui/{id}/ping` HTTP round trip (not a relay WS ping frame) — this change adds the endpoint if it does not already exist as a minimal no-op | Plan states "or the WS ping frame if the relay grows one" as an explicit alternative; the HTTP round trip is the baseline this change can build without depending on an unscoped relay change | S:60 R:75 A:65 D:60 |
| 5 | Certain | The stats seam reuses the same `Display.flip` and WebSocket byte-counting hooks the Playwright perf spec already wraps, as a second independent consumer — the perf spec's own instrumentation is untouched | Plan V-D11: "the counters are the perf spec's three probes lifted into the tile … the spec keeps its own wraps" | S:85 R:80 A:85 D:85 |
| 6 | Confident | The overlay component is a new file (`gui-stats-overlay.tsx`) rather than inlined in `gui-surface.tsx` | Follows the existing pattern of extracting tile-level chrome into its own component file (e.g. the anticipated `gui-keybar.tsx` from S4); not explicitly mandated by the plan | S:55 R:85 A:70 D:60 |
| 7 | Certain | This change shares `src/lib/gui-posture.ts`'s existing helper shape (introduced by S4) for its own new postures rather than reimplementing get/set/notify plumbing | Combined-execution plan's explicit note: S6 "shares the posture module and the toolbar seam with S4" | S:80 R:85 A:85 D:80 |
| 8 | Confident | The 10%-agreement acceptance bar (overlay Mbit/s vs. perf spec Mbit/s) is validated once, manually or via the real-rig Playwright spec's assertions, not as a continuously enforced CI gate beyond that one test | Plan's acceptance line states the criterion but not a CI enforcement mechanism beyond "just test green" | S:55 R:80 A:65 D:55 |

8 assumptions (5 certain, 3 confident, 0 tentative, 0 unresolved).
