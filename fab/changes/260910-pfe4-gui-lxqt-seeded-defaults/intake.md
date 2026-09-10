# Intake: LXQt seeded defaults (S5 / L2)

**Change**: 260910-pfe4-gui-lxqt-seeded-defaults
**Created**: 2026-09-10

## Origin

> Drafted via `/fab-draft` as stage S5 of the combined GUI execution queue
> (`fab/plans/sahil/26-09-10-gui-combined-execution.md`). Implements § L2
> ("LXQt seeded defaults") of
> `fab/plans/sahil/26-09-10-gui-lxqt-desktop.md`. One-shot, no conversational
> discussion — the design record is the lxqt plan's § Decision log (L-D4,
> L-D5, L-D6, binding — L-D4 partially Likely, see below) and its § UX /
> § L2 Do / § L2 Acceptance sections, reproduced here in full.

**Load-bearing gate — read before starting apply.** This change is
**blocked on a prerequisite that does not yet exist on `origin/main` at
draft time**: the lxqt plan's § L0 spike verdict. § L0 ("Spike") is *not* a
fab change — it is a side task the operator runs directly on the VM,
recording numbers (idle RSS, relay Mbit/s, first-start screenshots, SIGTERM
behavior, and critically **whether LXQt 0.17 honors `XDG_CONFIG_DIRS` for
`panel.conf`/`session.conf`/`lxqt.conf`**) as a new `## L0 verdict` section
appended directly to `fab/plans/sahil/26-09-10-gui-lxqt-desktop.md` on
`origin/main`, per the plan's own instructions ("commits that edit directly
to `main` — the convention for plan documents in `fab/plans/sahil/`"). This
intake is drafted **before** that verdict exists, so it necessarily encodes
L-D4's `XDG_CONFIG_DIRS` mechanism as the plan's stated default with its
documented fallback, rather than the verified answer.

**The combined-execution plan enforces this gate operationally, not this
intake**: per its § Rules, "before spawning S5, check that
`26-09-10-gui-lxqt-desktop.md` contains a `## L0 verdict` section on
`origin/main`. Absent ⇒ pause, notify, and resume when it lands." This
intake being drafted and `ready` does **not** mean S5 may start — the
operator's spawn-time gate is the actual enforcement point, separate from
and later than this drafting step. Anyone picking up this change (human or
agent) MUST verify `## L0 verdict` exists on `origin/main` before beginning
apply, and MUST read it first: if it says LXQt 0.17 does **not** honor
`XDG_CONFIG_DIRS` for one or more of the three files, this change's design
in § What Changes below (the `XDG_CONFIG_DIRS` seed mechanism) is
**superseded** by the L-D4 fallback (a dedicated `XDG_CONFIG_HOME` plus
re-exporting the user's real config home as `RK_USER_CONFIG_HOME`) — the
plan states this fallback is adopted "verbatim, no re-discussion needed."

Waits for S1 (`gui-session-starters-and-wm-verb`) merged — this change needs
the session-starter D-Bus wrap (a bare `startlxqt` cannot reliably seed and
run its own panel without a session bus) and reuses S1's exclusion-set and
supervisor-log conventions.

## Why

**The pain.** After S1 merges, `rk gui wm lxqt --restart` starts a
functioning but **unseeded** LXQt session — its own default panel, its own
default (likely light, icon-heavy) theme, a screensaver/locker module that
would lock a passwordless display, and a seconds-precision clock that
repaints once a second (60 rects/minute of pure relay cost for nothing, per
the C5 bandwidth-per-changed-rect budget). None of this matches rk's
existing IceWM look (dark, minimal, no locker) or its bandwidth discipline.

**The consequence of not fixing it.** LXQt without a seed is a worse first
impression than staying on IceWM — the whole point of L1+L3 (S1, S3) is to
make LXQt a real, chosen alternative, and an unseeded LXQt undermines that
by looking and behaving like a default Linux desktop nobody curated.

**The approach, and why.** Seed LXQt through `XDG_CONFIG_DIRS`, not
`XDG_CONFIG_HOME` (L-D4 — **Likely, pending § L0 verdict**, see the gate
above): the supervisor prepends `<state>/run-kit/gui/lxqt/etc` to
`XDG_CONFIG_DIRS` for the session; seeded files live under
`…/etc/lxqt/{session.conf,panel.conf,lxqt.conf}` and
`…/etc/pcmanfm-qt/lxqt/settings.conf`, written when absent (write-once,
delete-to-re-seed — the same convention G1's IceWM `preferences` file
established). `XDG_CONFIG_DIRS` (system-default layering) rather than
`XDG_CONFIG_HOME` (the user's own config home) is chosen specifically so a
user who also runs LXQt locally keeps their own preferences as overrides,
and so rk's seed never gets confused with — or silently relocates — the
user's own chromium/editor profiles the way pointing `XDG_CONFIG_HOME` at
rk's state dir would. Seed content (L-D5): `session.conf` picks `openbox` as
the window manager with power-management and screensaver/locker modules
disabled; `panel.conf` is one bottom panel (menu, two quick-launch buttons
reusing the G1 launcher's resolved terminal/browser, task bar, tray, a
minutes-only clock — no seconds); `lxqt.conf` picks a dark theme (exact name
from the L0 verdict); `pcmanfm-qt/lxqt/settings.conf` sets a solid-color
desktop (`#3b4252`, rk's existing ground color) with icons off.

Alternatives the plan rejected and this intake does not reopen: seeding
XFCE (L-D8 — LXQt is the only seeded DE); a locker/screensaver of any kind
on this passwordless display; a seconds-precision clock; desktop icons.

## What Changes

Backend (`app/backend/`) plus docs.

### 0. Prerequisite check (apply-entry, before any other task)

The apply-entry agent MUST first confirm `## L0 verdict` exists in
`fab/plans/sahil/26-09-10-gui-lxqt-desktop.md` on `origin/main` (`git fetch
origin && git show origin/main:fab/plans/sahil/26-09-10-gui-lxqt-desktop.md
| grep -q '^## L0 verdict'`). If absent, **do not proceed** — this is the
operator's queue-pause condition (combined-execution plan § Rules), and an
agent picking this change up out-of-band (outside the queue) must escalate
to the user rather than guess the verdict. If present, read it in full and
apply any correction it states to items 1–2 below before implementing them
— the verdict is authoritative over this intake's L-D4-as-written content.

### 1. The seed content — `internal/gui/seed_lxqt.go` (new)

Per L-D5, `go:embed` four files (exact key/value content taken from the §
L0 verdict — this intake reproduces the plan's stated *intent* for each
file; the verdict supplies the exact config keys that worked on 0.17):

- **`session.conf`**: window manager `openbox`; no `lxqt-powermanagement`
  module; no screensaver/locker module (no `xscreensaver`).
- **`panel.conf`**: one bottom panel — main menu · quick-launch (terminal,
  browser, via the G1 launcher's resolved binaries, regenerated on every
  supervise start exactly like G1's IceWM `toolbar`/`menu` files — the
  `toolbar` rule from G-D3) · task bar · tray · clock **without seconds**
  (one repaint a minute).
- **`lxqt.conf`**: a dark theme from `lxqt-themes` (exact name from the L0
  verdict — recorded Confident below, pending that verdict).
- **`pcmanfm-qt/lxqt/settings.conf`**: desktop wallpaper mode `color`, color
  `#3b4252` (matching rk's existing `xsetroot` ground), desktop icons off.

```go
// SeedLXQtDefaults writes the four seed files under
// <state>/run-kit/gui/lxqt/etc/{lxqt/{session,panel,lxqt}.conf,
// pcmanfm-qt/lxqt/settings.conf} when absent (write-once — a user's own
// edit to any file persists across restarts; delete the file to re-seed).
// The panel's quick-launch entries are regenerated on every call from the
// resolved launcher argv[0] names (rows omitted for a role that did not
// resolve), matching the IceWM toolbar/menu convention (G-D3). resolved
// carries the terminal/browser names from gui.ResolveApp (G1/S1's launcher).
func SeedLXQtDefaults(dir string, resolved LaunchResolution) (seeded bool, err error)
```

Tests on a `t.TempDir()`: first call seeds all four files with correct
permissions and content (dir `0700`, files `0600`, matching G1's
`internal/gui/seed.go` convention for the IceWM profile); a user edit to
`session.conf`/`lxqt.conf` survives a second call byte-identical
(`seeded=false` on that call, or a per-file write-once semantics if the
plan's exact granularity differs — this intake assumes whole-file write-once
per the G1 precedent, recorded as an assumption below); `panel.conf`'s
quick-launch entries track a changed launcher resolution across calls
(browser absent → present); deleting any one file re-seeds only that file.

### 2. Supervisor wiring — `cmd/rk/gui_supervise.go`

When the resolved WM (post S1's `ResolveWM`) is `startlxqt` (or
`lxqt-session`): call `SeedLXQtDefaults` (item 1) before starting the WM,
then set `XDG_CONFIG_DIRS=<dir>:${XDG_CONFIG_DIRS:-/etc/xdg}` in the child
env passed to the session starter — the same pattern S1's `WMArgv`/env
wiring already establishes for `dbus-run-session`, and directly analogous
to the existing `ICEWM_PRIVCFG` env-var precedent from G1 (the env builder
is already rung-specific, per the plan). Log line on first start: `gui:
window manager startlxqt (session under dbus-run-session; defaults
<dir>, seeded)` — extending S1's plainer `(session under dbus-run-session)`
line (S1 shipped without this segment since seeding did not exist yet; this
change adds the segment conditionally, when seeding actually occurred or the
seed dir already exists from a prior run).

**If the § L0 verdict (item 0) says `XDG_CONFIG_DIRS` does *not* work for
one or more files**, this item is replaced per L-D4's stated fallback: a
dedicated `XDG_CONFIG_HOME` for the LXQt session plus re-exporting the
user's real `XDG_CONFIG_HOME` (if any) as `RK_USER_CONFIG_HOME` and
documenting the substitution — the apply-entry agent implements whichever
branch the verdict selects and records the choice as a plan Design
Decision, not a re-opened intake question.

Test: the env var is set exactly for the LXQt rung (both `startlxqt` and
`lxqt-session` names), absent for every other rung; the seed call happens
before the WM starts; the log-line variants (seeded vs. already-present).

### 3. Integration test — `cmd/rk/gui_supervise_integration_test.go` (extends the existing file, per S1's naming precedent) or a new file if the existing one does not accept an LXQt-specific case cleanly

Capability-gated: skip unless `Xtigervnc`, `startlxqt` (or `lxqt-session`),
and `dbus-run-session` all resolve on PATH. Body (mirroring G1's IceWM
integration test structure): supervise on a temp state dir with `gui.wm`
pinned to the LXQt name; assert the `@rk_gui_wm` stamp equals the resolved
LXQt binary name; assert the four seed files exist with `0600`; assert
`gui.RunningApps` on a captured `/proc` fixture (or, if run live, the real
`/proc`) lists none of S1's widened `wmHelperComms` LXQt/dbus names; assert
a screenshot's pixel at the desktop center is `#3b4256` (or whatever the
exact hex from L-D5 renders as, allowing for VNC color-depth rounding).
Never touches the live `rk-gui` session (temp state dir, high display
number via `gui.FreeDisplay`).

### 4. Docs

- `docs/specs/gui.md` § Switching desktops (existing section from S1) gains
  the seed paragraph: what gets seeded, the write-once/regenerated split,
  and the "delete the dir to re-seed" rule. If the L0 verdict triggered the
  `XDG_CONFIG_HOME`/`RK_USER_CONFIG_HOME` fallback, this section documents
  that mechanism instead (per L-D4's stated fallback plan).
- Memory via hydrate (§ Affected Memory below).

### Acceptance (from the plan, binding)

- Fresh `<state>/gui/lxqt`, `rk gui wm lxqt --restart` → the tile shows one
  bottom panel with menu, two quick-launch buttons, task bar, tray, and a
  minutes-only clock over a solid `#3b4256` desktop with no icons.
- Idle relay traffic over 60 s stays within 2× the IceWM idle figure from
  the § L0 verdict.
- `rk gui off` confirm lists no `lxqt-*` process (relies on S1's widened
  exclusion set).
- A hand-edited `…/etc/lxqt/panel.conf` survives `rk gui restart`.
- `go test ./...` and `just test` green.

## Affected Memory

- `run-kit/gui`: (modify) the LXQt seed mechanism (`SeedLXQtDefaults`, the
  four files, write-once vs. regenerated split), the `XDG_CONFIG_DIRS`
  wiring (or its `XDG_CONFIG_HOME`/`RK_USER_CONFIG_HOME` fallback, per the
  verdict), the seeded-defaults segment of the supervisor's WM-found log
  line; new Design Decision (which of L-D4's two mechanisms the verdict
  selected, with the rejected alternative noted per the SRAD apply-records
  rule)
- `run-kit/daemon-lifecycle`: (modify) the supervisor's start-order gains a
  seed step before starting a session-starter WM

## Impact

**Code (`app/backend/`)**: `internal/gui/seed_lxqt.go` (new, `go:embed`),
`cmd/rk/gui_supervise.go` (seed call + env wiring + log line),
`cmd/rk/gui_supervise_integration_test.go` (or equivalent, capability-gated)
— each with tests.

**Contracts that change shape**: none — purely additive (a new seed
function, a new env var conditionally set, an extended log line). No route,
stream field, or exported signature changes.

**Behavior change for existing hosts**: only hosts that pin `gui.wm` to an
LXQt session-starter name (via S1's `rk gui wm lxqt` or S3's picker) see any
change — a bare `startlxqt` session becomes seeded on its first start after
this change merges (write-once — a prior manual LXQt session's own
`~/.config/lxqt` files, if any, are untouched since the seed lives under
`XDG_CONFIG_DIRS`, not `XDG_CONFIG_HOME`). IceWM-pinned or auto-resolved
hosts are entirely unaffected.

**Tests**: Go unit tests for the seed function and the supervisor wiring;
one capability-gated integration test. Gates: `cd app/backend && go test
./...`, `just test`, `just build`.

**Dependencies**: none new. `lxqt-core` is a host package the user installs
(per S1's `rk gui wm lxqt` refusal path when absent); this change's unit
tests run without it (temp-dir seeding needs no LXQt binary), only the
integration test and the live-VM acceptance need it present.

## Open Questions

None as intake-blocking questions for the *design* — L-D5's seed content is
fully specified. The one genuine unknown, **whether `XDG_CONFIG_DIRS` works
for LXQt 0.17's three config files**, is not an open question to ask a
human about — it is a gate this change cannot even start apply against until
the § L0 verdict exists on `origin/main` (see the load-bearing note in
§ Origin above), and once it exists the answer is mechanical: implement
whichever of L-D4's two branches the verdict names, verbatim, per the plan's
own instruction that no re-discussion is needed.

## Assumptions

| # | Grade | Decision | Rationale | Scores |
|---|-------|----------|-----------|--------|
| 1 | Unresolved | Whether LXQt 0.17 honors `XDG_CONFIG_DIRS` for `panel.conf`/`session.conf`/`lxqt.conf` (deciding whether L-D4's primary mechanism or its `XDG_CONFIG_HOME`/`RK_USER_CONFIG_HOME` fallback applies) | Deferred — the plan's own § L0 spike has not yet run/landed at draft time; this is a hard external-verification gate, not something an agent can infer from the codebase or the constitution. The combined-execution plan's operator enforces the gate at spawn time (pause until `## L0 verdict` exists on `origin/main`); this intake documents both branches so apply proceeds mechanically once the verdict lands | S:30 R:20 A:10 D:15 |
| 2 | Certain | Seed via `<state>/run-kit/gui/lxqt/etc` prepended to `XDG_CONFIG_DIRS` (assuming the verdict confirms this branch), write-once per file, regenerated panel quick-launch entries — mirrors the G1 IceWM seed precedent exactly | Plan L-D4 (primary branch) and L-D5; the code-server `settings.json` write-once precedent G1 already established | S:85 R:70 A:85 D:80 |
| 3 | Certain | Seed content: `session.conf` → `openbox` WM, no powermanagement/locker; `panel.conf` → one bottom panel (menu, 2 quick-launch, taskbar, tray, minutes-only clock); `lxqt.conf` → a dark theme; `pcmanfm-qt/lxqt/settings.conf` → solid `#3b4256`, icons off | Plan L-D5 verbatim; the relay pays per changed rect, a seconds clock/locker/icons are all documented anti-patterns for this use | S:90 R:75 A:85 D:90 |
| 4 | Confident | The exact dark theme name from `lxqt-themes` is whatever the § L0 verdict records — this intake cannot name it in advance | Plan L-D5: "a dark theme from `lxqt-themes` (L0 picks)" — explicitly deferred to the verdict, but the *category* of decision (a dark theme, from the stock package) is settled | S:60 R:80 A:55 D:65 |
| 5 | Certain | Seeding runs only when the resolved WM is a member of the LXQt session-starter names (`startlxqt`, `lxqt-session`) — never for IceWM, bare WMs, or XFCE | Plan L-D1/L-D8 — LXQt is the one seeded DE; XFCE stays reachable but unseeded | S:90 R:85 A:90 D:90 |
| 6 | Certain | File permissions and write-once granularity mirror G1's IceWM seed exactly (dir `0700`, files `0600`, whole-file write-once, delete-to-re-seed) | No LXQt-specific permission requirement is stated in the plan; the existing G1 precedent is the established pattern this change extends | S:75 R:80 A:85 D:80 |
| 7 | Confident | The supervisor's env-var wiring (`XDG_CONFIG_DIRS=<dir>:${XDG_CONFIG_DIRS:-/etc/xdg}`) is set only for the LXQt rung, using the same "env builder is already rung-specific" mechanism the `ICEWM_PRIVCFG` precedent established | Plan L2 Do item 2 states this explicitly; mirrors the existing rung-specific env pattern | S:70 R:80 A:80 D:70 |
| 8 | Confident | The integration test extends/parallels the S1-era `cmd/rk/gui_supervise_integration_test.go` structure (capability-gated on Xtigervnc + startlxqt + dbus-run-session) rather than living in `internal/gui/xvnc_integration_test.go` as the plan's original text suggests | Follows S1's own established deviation from the plan's original file placement (the stamp and `runGuiSuperviseLinux` are `package main`) — same reasoning applies here | S:60 R:80 A:70 D:60 |

8 assumptions (5 certain, 2 confident, 0 tentative, 1 unresolved).
