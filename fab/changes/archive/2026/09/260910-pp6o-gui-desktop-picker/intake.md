# Intake: The desktop picker — Settings row + `GUI: Desktop…` (S3 / L3)

**Change**: 260910-pp6o-gui-desktop-picker
**Created**: 2026-09-10

## Origin

> Drafted via `/fab-draft` as stage S3 of the combined GUI execution queue
> (`fab/plans/sahil/26-09-10-gui-combined-execution.md`). Implements § L3
> ("The desktop picker (Settings row + `GUI: Desktop…`)") of
> `fab/plans/sahil/26-09-10-gui-lxqt-desktop.md`. One-shot, no conversational
> discussion — the design record is the lxqt plan's § Decision log (L-D9
> primarily, plus L-D1–L-D8 as context, binding) and its § UX / § L3 Do /
> § L3 Acceptance sections, reproduced here in full.

Waits for S1 (`gui-session-starters-and-wm-verb`) merged — this change needs
`IsSessionStarter` and the candidate label map S1 introduces. Per the
combined-execution queue, S3 runs third (after S1, alongside where S2 would
otherwise sit in dependency terms, but the queue is strictly serial: S3
follows S2 in merge order even though it depends only on S1). S4 and S7
reuse the control this change builds.

**Coordination point 1 is binding on this change, and this change is its
primary owner**: the settings-dialog **choice control** — a "string with
server-supplied choices" select, distinct from the registry's static `enum`
kind — belongs here. S2 (`gui-fixed-geometry-and-resize`, already merged at
this point in the queue) deliberately shipped `gui.geometry` as a plain text
field and left the richer control model to this change. This change builds
the control S4/S7 may later reuse for other server-discovered choices, and
also builds the `Restart the desktop now?` confirm-dialog reuse (the
existing `gui-off-dialog` shell, given new copy and the running-apps list
from the status document).

## Why

**The pain.** `gui.wm`'s only user-facing surfaces today (after S1) are a
free-text Settings field and `rk gui wm`. A person who wants "LXQt" has to
know the exact binary name (`startlxqt`), type it correctly, and separately
discover that a restart is required and which apps will die. There is no
discoverable list of what is actually installed on this host, and no
confirm-before-destroy step in the Settings UI (the CLI verb's `--restart`
flag is at least an explicit ask; the free-text field's restart-required
implication is invisible).

**The consequence of not fixing it.** Constitution V (Keyboard-First) says
the palette is the complete action registry and the primary discovery
mechanism — a free-text binary name for a choice with three real answers
(auto/IceWM/LXQt, plus "other") is a poor control and a poor palette entry.
Every later stage that wants to offer a discoverable choice (there is
exactly one other candidate in this queue set — none currently planned, but
the control is deliberately shaped as reusable) would otherwise have to
build its own select-with-server-data machinery from scratch.

**The approach, and why.** One desktop picker, two doors (L-D9): the
`gui.wm` Settings row becomes a select — `Auto (ladder)` first, then every
**installed** candidate the server detected (labeled `IceWM`, `LXQt`, `XFCE`,
or the bare binary name), then `Other…` revealing the existing free-text
field for a typed binary. The palette gains `GUI: Desktop…` opening the same
picker as a sub-list. Installed candidates are derived server-side
(`GET /api/gui/{id}` gains `wm_candidates`, via `LookPath` over the ladder ∪
session-starter set, `IsSessionStarter` from S1 supplying `kind`) — never
stored (Constitution II), and computed on the existing status read (no new
stream field: it only changes when packages change). Choosing a value
writes through `POST /api/settings` as always, then asks
`Restart the desktop now? Running apps will close: <apps>` reusing the
off-confirm dialog shell, with `Restart` / `Later`. `Later` leaves the pin
set for the next `rk gui on`/`restart` — no forced immediacy.

Alternatives the plan rejected and this intake does not reopen: a picker
inside the gui tile itself (L-D8 — the picker lives in Settings/palette
only); seeding XFCE or any DE beyond LXQt (L-D1/L-D8 — XFCE stays reachable,
unseeded, one documentation line); storing the candidate list anywhere
(Constitution II — always derived at request time from `LookPath`).

## What Changes

Backend (`app/backend/`) and frontend (`app/frontend/`) plus docs.

### 1. `wm_candidates` on the status document — `api/gui.go`

Per L-D9:

```go
// WMCandidate is one entry in the desktop picker's list.
type WMCandidate struct {
    Name      string `json:"name"`      // binary name, e.g. "startlxqt"
    Label     string `json:"label"`     // "LXQt", "IceWM", "XFCE", or the raw name for an unlabeled bare WM
    Kind      string `json:"kind"`      // "wm" | "session" — from gui.IsSessionStarter (S1)
    Installed bool   `json:"installed"` // always true for a listed entry (see below)
}
```

`gui.Status` gains `WMCandidates []WMCandidate` — iterate the ladder ∪
session-starter set (both from `internal/gui/backend.go`, already exported
after S1) through the existing `LookPath` seam; a name that does not
resolve is **omitted entirely**, not listed with `installed:false` — L-D9's
UX explicitly wants "candidates not installed are not listed" (the footer
hint carries the install line instead). Label map: `icewm-session` → `IceWM`,
`startlxqt` → `LXQt`, `startxfce4` → `XFCE`, anything else → the raw binary
name. `Kind` from `gui.IsSessionStarter(name)` → `"session"` when true,
`"wm"` otherwise. `gui.Status` also gains `WMHint string` reusing the
existing `wm_hint`-style server text (S1's `DEInstallHint` for LXQt when it
is not installed) as the picker's footer line — computed once alongside
`WMCandidates`, distinct from the existing bare-WM `wm_hint` (S1's field,
present when the *resolved* WM is empty) even though both may render
similar apt lines; this field is present whenever the LXQt candidate is
absent from the list, regardless of what actually resolved.

Handler test: a stubbed `LookPath` returning hits for `icewm-session` and
`startlxqt` only produces exactly those two candidates with correct
labels/kinds, in ladder-then-session-starter-set order (or another
documented stable order); a `LookPath` returning no hits produces an empty
list plus the LXQt-install footer hint.

### 2. The Settings-dialog select control — `app/frontend/src/components/settings-dialog*.tsx`

The `gui.wm` row renders as a **select** once the gui status has been
fetched (falling back to today's free-text field before the first fetch
resolves, or if the fetch fails — never blocking the dialog on a slow/failed
status call): options are `Auto (ladder)` first, then each entry from
`wm_candidates` (label as the visible text, name as the value), then
`Other…` which reveals the existing text input for a typed binary name (the
S1/S2-era free-text field, now demoted to the `Other…` sub-state rather than
removed). On change: `POST /api/settings {"gui.wm": <name>}` (unchanged
write path), then the restart confirm (item 3) before any restart call is
made — selecting a value alone does not restart.

This is the **new, reusable control** Coordination point 1 assigns here: a
select whose option list comes from a server field rather than the static
registry `enum` kind. It is scoped narrowly to `gui.wm` in this change; nothing
in this change modifies the registry schema to add a generic "server-
discovered options" kind — that generalization, if ever needed, is a
separate decision.

Tests (vitest): renders `Auto`, each candidate label, and `Other…`; selecting
`Other…` reveals the text field; selecting a candidate does not itself call
restart; the free-text fallback path when `wm_candidates` is absent/empty
(no candidates ⇒ only `Auto` and `Other…`, still functional).

### 3. The restart confirm — reusing the off-confirm shell

`Restart the desktop now? Running apps will close: <apps>` — `<apps>` is the
existing running-apps list the `gui-off-dialog` shell already renders (S1's
apps exclusion set applies, so DE daemons never appear in this list).
Two actions: `Restart` → `POST /api/gui/host/restart` (existing endpoint,
unchanged); `Later` → no further call, the setting stays as posted in item 2,
takes effect on the next `rk gui on`/`restart`.

Test (vitest): the confirm renders with the status document's current apps
list; `Restart` calls the restart endpoint; `Later` does not.

### 4. Palette row — `src/lib/palette/gui.ts`

`GUI: Desktop…` — gated `enabled` (not also `reachable`; the picker should
work even when the gui host is off, mirroring how `gui.wm` can be set while
off per S1's bare-set rule) — opens the same picker as a **palette
sub-list**: `Auto` first (marked current when selected), then each
`wm_candidates` entry (current one marked), reusing item 2's data and
selection logic rather than a separate implementation. Selecting an entry
here goes through the identical write → confirm → restart/later flow as the
Settings dialog.

Test (vitest): the palette row's presence/absence per `enabled`; the
sub-list's entries match `wm_candidates`; selecting an entry triggers the
same confirm flow as the dialog.

### 5. Footer hint line

Below the picker (both Settings-dialog and palette sub-list contexts, or at
minimum the Settings dialog per § UX — the palette sub-list may omit it if a
palette sub-list has no natural footer slot, in which case the hint is
Settings-dialog-only): `Install more: sudo apt install --no-install-
recommends lxqt-core` from `wm_hint` (item 1) — the L-D7 hint for the one
supported DE, shown only when LXQt is absent from `wm_candidates`.

### 6. Tests

- Go: item 1's handler test (candidate derivation, label/kind mapping,
  omission of non-installed names, footer hint presence).
- Vitest: item 2's select states (candidates present / none but Auto+Other
  / Other… reveal), item 3's confirm → restart/later chain, item 4's
  palette gating and sub-list parity with the dialog.
- Playwright, with a stubbed status document listing two candidates
  (`IceWM`, `LXQt`): pick `LXQt` → confirm dialog names the running apps →
  `Later` leaves the setting posted and issues no restart call; a second
  pass picking `LXQt` → `Restart` posts the restart call. Intent comments
  per Constitution § Test Intent Comments.

### 7. Docs

- `docs/specs/gui.md` § Switching desktops (new short section, or extending
  whatever S1 already added there): the three CLI commands (from S1's § UX),
  plus one line for the picker (Settings → GUI → Desktop, and `Cmd+K` →
  `GUI: Desktop…`), then the existing XFCE-unseeded/compositor-caveat line
  from the lxqt plan's § UX → Docs.
- Memory via hydrate (§ Affected Memory below).

### Acceptance (from the plan, binding)

- On this VM with `lxqt-core` installed: Settings → GUI → Desktop shows
  `Auto (ladder)`, `IceWM`, `LXQt`, `Other…`.
- Choosing `LXQt` and `Restart` brings up the LXQt desktop in the tile
  (default look — S5/L2 has not necessarily merged yet in isolated testing
  of this change, so "default look" not "seeded look" is the acceptance
  bar for this change alone; once S5 has merged the desktop will additionally
  be seeded, which this change's acceptance does not depend on).
- `Cmd+K` → `GUI: Desktop…` → `IceWM` → `Restart` brings IceWM back.
- With `lxqt-core` removed, the LXQt row is absent and the footer shows the
  apt line.
- `just test` green.

## Affected Memory

- `run-kit/gui`: (modify) `wm_candidates` on `GET /api/gui/{id}` (derivation
  via `LookPath` over ladder ∪ session-starter set, never stored); the
  Settings-dialog select control and its `Other…` free-text fallback; the
  `GUI: Desktop…` palette row and its sub-list; the restart-confirm reuse of
  the off-confirm shell for a WM change; new Design Decision (desktop
  candidates are always derived at request time, never persisted, per
  Constitution II)
- `run-kit/ui/keyboard-and-palette`: (modify) § The `GUI:` palette family —
  the new `GUI: Desktop…` row and its sub-list behavior
- `run-kit/ui/dialogs-and-state`: (modify) the restart-confirm reuse pattern
  (a second consumer of the off-confirm shell beyond `rk gui off`)
- `run-kit/api-and-sockets`: (modify) `GET /api/gui/{id}` document gains
  `wm_candidates` and (conditionally) `wm_hint`'s picker-footer usage

## Impact

**Code (`app/backend/`)**: `api/gui.go` (`WMCandidate` type, `wm_candidates`
+ footer-hint derivation on the status document) — with handler test.

**Code (`app/frontend/src/`)**: `components/settings-dialog*.tsx` (select
control + `Other…` reveal + restart confirm), `lib/palette/gui.ts` +
`app.tsx` (`GUI: Desktop…` row + sub-list) — with vitest coverage; one
Playwright spec (stubbed status document, two candidates).

**Contracts that change shape** (additive only): `gui.Status` gains
`wm_candidates` and reuses/extends `wm_hint`'s presence conditions. No
existing route, stream field, or exported function signature changes.

**Behavior change for existing hosts**: none for hosts that never open
Settings → GUI → Desktop or the palette row — the underlying `gui.wm`
write path, restart endpoint, and running-apps list are all unchanged from
S1. A host with LXQt installed sees a richer picker instead of a free-text
field; a host without any DE installed beyond the ladder's bare WMs sees
`Auto` + `Other…` only, functionally identical to today's free-text field.

**Tests**: Go handler test; vitest for the select/palette/confirm; one
Playwright spec with a stubbed status document (two candidates). Gates:
`cd app/backend && go test ./...`, `just test`, `just build`, frontend
vitest + Playwright suites.

**Dependencies**: none new. Depends on S1's `IsSessionStarter`,
`sessionStarters`/ladder exports, and `DEInstallHint` being on `main`.

## Open Questions

None. The lxqt plan's § Decision log (L-D9) resolves every design point for
L3 and its § UX fixes the exact copy for both the picker and the confirm
dialog. Whether the footer hint line renders inside the palette sub-list or
Settings-dialog-only is left to the implementer as a Confident assumption
(below), not a question, since the plan's § UX shows it in the dialog
context and is silent on the palette sub-list's exact layout.

## Assumptions

| # | Grade | Decision | Rationale | Scores |
|---|-------|----------|-----------|--------|
| 1 | Certain | `wm_candidates` derived via `LookPath` over ladder ∪ session-starter set on every status read; never stored; a non-installed name is omitted entirely, not flagged `installed:false` | Plan L-D9 verbatim; Constitution II — no database, derive at request time | S:90 R:80 A:90 D:90 |
| 2 | Certain | Label map `icewm-session`→`IceWM`, `startlxqt`→`LXQt`, `startxfce4`→`XFCE`, else the raw binary name; `kind` from `IsSessionStarter` | Plan L-D9 verbatim, reusing S1's exported predicate | S:90 R:85 A:90 D:90 |
| 3 | Certain | The Settings-dialog `gui.wm` row becomes a select (`Auto` → installed candidates → `Other…` revealing the existing free-text field), falling back to the free-text field before the status fetch resolves or on fetch failure | Plan L-D9; a dialog must never block on a slow/failed status call | S:80 R:80 A:85 D:80 |
| 4 | Certain | `GUI: Desktop…` palette row opens the identical picker as a sub-list, sharing selection/write/confirm logic with the dialog rather than a separate implementation | Plan L-D9; Constitution V — palette is the complete action registry, and code reuse avoids drift between the two doors | S:85 R:80 A:85 D:85 |
| 5 | Certain | Choosing a candidate writes `gui.wm` via `POST /api/settings` immediately, then shows `Restart the desktop now? Running apps will close: <apps>` reusing the `gui-off-dialog` shell; `Restart` calls the existing restart endpoint, `Later` leaves the pin for next restart | Plan L-D9 verbatim; matches the existing off-confirm precedent for a destructive action | S:90 R:80 A:90 D:90 |
| 6 | Certain | The footer hint (`Install more: sudo apt install --no-install-recommends lxqt-core`) shows only when LXQt is absent from `wm_candidates`, using S1's `DEInstallHint` | Plan L-D9's footer-line UX; reuses S1's hint function rather than duplicating wording | S:85 R:90 A:90 D:85 |
| 7 | Confident | The footer hint line is shown in the Settings dialog; whether the palette sub-list also renders it (vs. omitting for lack of a natural footer slot) is an implementation choice, not specified by the plan's UX block | Plan's § UX shows the hint only in the Settings-dialog picker context | S:55 R:90 A:70 D:60 |
| 8 | Certain | No registry schema change — this is a component-level select scoped to `gui.wm`, not a new generic "server-discovered options" registry kind | Coordination point 1 scopes this narrowly; a generalized control kind is out of scope for this change | S:85 R:85 A:85 D:85 |
| 9 | Confident | Candidate ordering is ladder-then-session-starter-set (or another documented stable order) — the plan does not specify an exact sort, only that `Auto` comes first and `Other…` comes last | Plan § UX gives the example order `IceWM, LXQt, Other…` consistent with ladder-then-starters, but does not state a formal sort rule | S:55 R:85 A:70 D:55 |
| 10 | Certain | This change's own acceptance does not require S5 (LXQt seeded defaults) to have merged — "default look" is sufficient for this change's acceptance bar; the seeded look is S5's separate acceptance | Combined-execution queue places S3 before S5; each stage's acceptance must stand alone against main at the point it merges | S:80 R:85 A:85 D:80 |

10 assumptions (7 certain, 3 confident, 0 tentative, 0 unresolved).
