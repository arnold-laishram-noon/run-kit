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

**Gate resolved.** `fab/plans/sahil/26-09-10-gui-lxqt-desktop.md` now carries
a `## L0 verdict` section on `main` (committed 2026-09-10). It **confirms
L-D4 as written, no fallback needed**: LXQt 0.17.1 keeps a caller-supplied
`XDG_CONFIG_DIRS` and only *appends* `/etc`, `/etc/xdg`, `/usr/share` when
missing, so a directory rk prepends stays first in the search order — all
four seed files (`session.conf`, `panel.conf`, `lxqt.conf`,
`pcmanfm-qt/lxqt/settings.conf`) were read from the prepended dir with an
empty `XDG_CONFIG_HOME`. This intake now encodes the verdict's mechanism and
exact config keys directly (§ What Changes below), not the plan's prior
"stated default with documented fallback" language. The apply-entry agent
still confirms the section exists on `origin/main` before starting (§ What
Changes item 0), as a mechanical check, not because the answer is in doubt.

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
`XDG_CONFIG_HOME` (L-D4 — **Certain, confirmed by the § L0 verdict**): the
supervisor prepends `<state>/run-kit/gui/lxqt/etc` to `XDG_CONFIG_DIRS` for
the session; seeded files live under
`…/etc/lxqt/{session.conf,panel.conf,lxqt.conf}`,
`…/etc/pcmanfm-qt/lxqt/settings.conf`, and `…/etc/autostart/`, written when
absent (write-once, delete-to-re-seed — the same convention G1's IceWM
`preferences` file established). `XDG_CONFIG_DIRS` (system-default layering)
rather than `XDG_CONFIG_HOME` (the user's own config home) is chosen
specifically so a user who also runs LXQt locally keeps their own
preferences as overrides, and so rk's seed never gets confused with — or
silently relocates — the user's own chromium/editor profiles the way
pointing `XDG_CONFIG_HOME` at rk's state dir would; the verdict confirms
this is exactly the mechanism Ubuntu's own `/usr/share` defaults already use.
Seed content (L-D5, corrected by the verdict): `session.conf` picks
`openbox` as the window manager with `leave_confirmation=false`,
`lock_screen_before_power_actions=false` (no power-management or
screensaver/locker module); `panel.conf` is one bottom panel (menu, one
quick-launch button seeded as a `.desktop` path reusing the G1 launcher's
resolved terminal/browser — not a raw `exec=`, which renders no button when
the binary isn't first resolved — task bar, tray, a minutes-only clock via
`formatType=custom, useAdvancedManualFormat=true, customFormat=HH:mm` — the
`timeShowSeconds`/`short-timeonly` combination the plan originally described
still shows seconds, since "short" is the locale's short time format, not a
manual one); `lxqt.conf` picks theme `dark` with `icon_theme` probed from
installed icon themes (`breeze-dark` → `Papirus-Dark` → `Adwaita`, else
omitted — `lxqt-core` ships none); `pcmanfm-qt/lxqt/settings.conf` sets
`WallpaperMode=none` (not `color`, which pcmanfm-qt does not recognize as a
mode name — `none` is the solid-`BgColor` mode) with `BgColor=#3b4252` (rk's
existing ground color) and icons off (`HideItems=true`,
`DesktopShortcuts=`); an `autostart/lxqt-xscreensaver-autostart.desktop`
seed with `Hidden=true` shadows the system one so a user who installs
`xscreensaver` doesn't get it autostarted under LXQt.

Alternatives the plan rejected and this intake does not reopen: seeding
XFCE (L-D8 — LXQt is the only seeded DE); a locker/screensaver of any kind
on this passwordless display; a seconds-precision clock; desktop icons.

## What Changes

Backend (`app/backend/`) plus docs.

### 0. Prerequisite check (apply-entry, before any other task)

The apply-entry agent confirms `## L0 verdict` exists in
`fab/plans/sahil/26-09-10-gui-lxqt-desktop.md` on `origin/main` (`git fetch
origin && git show origin/main:fab/plans/sahil/26-09-10-gui-lxqt-desktop.md
| grep -q '^## L0 verdict'`) — a mechanical sanity check, since this intake
was clarified after the verdict landed and already encodes its answer
(L-D4 confirmed, no fallback) and its exact config keys directly into items
1–2 below. If the check somehow fails (e.g. a stale `origin` fetch), stop
and re-sync rather than guess; the verdict content quoted below is
authoritative and should not drift from what's on `main`.

### 1. The seed content — `internal/gui/seed_lxqt.go` (new)

Per L-D5 as corrected by the § L0 verdict, `go:embed` five files (exact
key/value content verified on LXQt 0.17.1 and checked for drift against
lxqt master by the verdict):

- **`etc/lxqt/session.conf`**: `[General] window_manager=openbox
  leave_confirmation=false lock_screen_before_power_actions=false`;
  `[Environment] GTK_CSD=0 GTK_OVERLAY_SCROLLING=0`. No
  `lxqt-powermanagement` module (not in `lxqt-core`); no screensaver/locker
  module — handled instead by the autostart override below, since
  `lxqt-xscreensaver-autostart.desktop` ships with `lxqt-session` itself,
  not as a separate module.
- **`etc/lxqt/panel.conf`**: one bottom panel (`panels=panel1`,
  `position=Bottom`, `panelSize=32`, `iconSize=22`, `lineCount=1`) with
  plugins `mainmenu,quicklaunch,taskbar,tray,statusnotifier,worldclock` —
  main menu · quick-launch (terminal, browser, seeded as `.desktop` paths
  reusing the G1 launcher's resolved binaries — `apps\N\desktop=`, regenerated
  on every supervise start exactly like G1's IceWM `toolbar`/`menu` files,
  the `toolbar` rule from G-D3; a raw `exec=` entry whose binary isn't first
  resolved renders no button, per the verdict, so only resolved roles get an
  entry) · task bar · tray · status-notifier · clock **without seconds**
  (`formatType=custom, useAdvancedManualFormat=true, customFormat=HH:mm` —
  one repaint a minute; `formatType=short-timeonly` +
  `timeShowSeconds=false` does *not* suppress seconds, per the verdict).
- **`etc/lxqt/lxqt.conf`**: `[General] theme=dark
  icon_theme=<probed>` — theme name `dark` (from `lxqt-themes`, confirmed);
  icon theme probed at seed time from `/usr/share/icons`, first hit of
  `breeze-dark`, `Papirus-Dark`, `Adwaita`, key omitted if none present
  (`lxqt-core` installs no icon theme itself — without one the panel falls
  back to text labels, which is an acceptable degrade, not a failure);
  `[Qt] style=Fusion`.
- **`etc/pcmanfm-qt/lxqt/settings.conf`**: `[Desktop] WallpaperMode=none
  BgColor=#3b4252 FgColor=#ffffff HideItems=true DesktopShortcuts=
  ShowHidden=false` — `none` is pcmanfm-qt's solid-`BgColor` mode; `color` is
  not a value it recognizes (the plan's original text was wrong on this key;
  the verdict corrects it).
- **`etc/autostart/lxqt-xscreensaver-autostart.desktop`**: `Type=Application
  Name=XScreenSaver (disabled by rk) Exec=xscreensaver -no-splash
  Hidden=true OnlyShowIn=LXQt;` — an `XDG_CONFIG_DIRS`-seeded autostart entry
  shadows the system one (`Hidden=true` suppresses it), which is the actual
  mechanism that keeps a user-installed `xscreensaver` from autostarting
  under LXQt (there is no separate "locker module" to disable in
  `session.conf` — the verdict found this was the real mechanism).

```go
// SeedLXQtDefaults writes the five seed files under
// <state>/run-kit/gui/lxqt/etc/{lxqt/{session,panel,lxqt}.conf,
// pcmanfm-qt/lxqt/settings.conf, autostart/lxqt-xscreensaver-autostart.desktop}
// when absent (write-once — a user's own edit to any file persists across
// restarts; delete the file to re-seed). The panel's quick-launch entries
// are regenerated on every call as .desktop-path entries from the resolved
// launcher (rows omitted for a role that did not resolve), matching the
// IceWM toolbar/menu convention (G-D3). resolved carries the terminal/
// browser names from gui.ResolveApp (G1/S1's launcher); icon theme is
// probed from /usr/share/icons at seed time, not passed in.
func SeedLXQtDefaults(dir string, resolved LaunchResolution) (seeded bool, err error)
```

Tests on a `t.TempDir()`: first call seeds all five files with correct
permissions and content (dir `0700`, files `0600`, matching G1's
`internal/gui/seed.go` convention for the IceWM profile); a user edit to
`session.conf`/`lxqt.conf` survives a second call byte-identical
(`seeded=false` on that call, or a per-file write-once semantics if the
plan's exact granularity differs — this intake assumes whole-file write-once
per the G1 precedent, recorded as an assumption below); `panel.conf`'s
quick-launch entries track a changed launcher resolution across calls
(browser absent → present, seeded as a `.desktop` path); `lxqt.conf`'s
`icon_theme` reflects whatever the probe finds on the test's fake
`/usr/share/icons` (present → named, absent → key omitted); deleting any one
file re-seeds only that file.

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

No fallback branch — the § L0 verdict confirmed `XDG_CONFIG_DIRS` works for
all seeded files, so `XDG_CONFIG_HOME`/`RK_USER_CONFIG_HOME` (L-D4's
fallback) is not implemented by this change.

Test: the env var is set exactly for the LXQt rung (both `startlxqt` and
`lxqt-session` names), absent for every other rung; the seed call happens
before the WM starts; the log-line variants (seeded vs. already-present).

### 3. Integration test — `cmd/rk/gui_supervise_integration_test.go` (extends the existing file, per S1's naming precedent) or a new file if the existing one does not accept an LXQt-specific case cleanly

Capability-gated: skip unless `Xtigervnc`, `startlxqt` (or `lxqt-session`),
and `dbus-run-session` all resolve on PATH. Body (mirroring G1's IceWM
integration test structure): supervise on a temp state dir with `gui.wm`
pinned to the LXQt name; assert the `@rk_gui_wm` stamp equals the resolved
LXQt binary name; assert the five seed files exist with `0600`; assert
`gui.RunningApps` on a captured `/proc` fixture (or, if run live, the real
`/proc`) lists none of S1's widened `wmHelperComms` LXQt/dbus names; assert
a screenshot's pixel at the desktop center is `#3b4252` (the verdict's own
measurement sampled `srgb(59,66,82)` = `0x3b4252`, matching rk's ground
color; allow for VNC color-depth rounding). Never touches the live
`rk-gui` session (temp state dir, high display number via `gui.FreeDisplay`).

### 4. Docs

- `docs/specs/gui.md` § Switching desktops (existing section from S1) gains
  the seed paragraph: what gets seeded (the five files), the
  write-once/regenerated split, and the "delete the dir to re-seed" rule.
  No `XDG_CONFIG_HOME`/`RK_USER_CONFIG_HOME` fallback to document — the
  verdict confirmed the primary mechanism works.
- Memory via hydrate (§ Affected Memory below).

### Acceptance (from the plan, binding)

- Fresh `<state>/gui/lxqt`, `rk gui wm lxqt --restart` → the tile shows one
  bottom panel with menu, two quick-launch buttons (terminal and browser,
  each present only if its role resolved), task bar, tray, and a
  minutes-only clock over a solid `#3b4252` desktop with no icons.
- Idle relay traffic over 60 s stays within 2× the IceWM idle figure from
  the § L0 verdict.
- `rk gui off` confirm lists no `lxqt-*` process (relies on S1's widened
  exclusion set).
- A hand-edited `…/etc/lxqt/panel.conf` survives `rk gui restart`.
- `go test ./...` and `just test` green.

## Affected Memory

- `run-kit/gui`: (modify) the LXQt seed mechanism (`SeedLXQtDefaults`, the
  five files, write-once vs. regenerated split), the `XDG_CONFIG_DIRS`
  wiring (confirmed working, no `XDG_CONFIG_HOME` fallback needed), the
  seeded-defaults segment of the supervisor's WM-found log line; new Design
  Decision recording L-D4 confirmed as written (the rejected
  `XDG_CONFIG_HOME`/`RK_USER_CONFIG_HOME` fallback noted per the SRAD
  apply-records rule), plus the L-D5 corrections (clock format keys,
  wallpaper mode, quick-launch `.desktop` paths, icon-theme probe, the
  autostart-shadow mechanism for the locker)
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

None. The one genuine unknown at draft time — **whether `XDG_CONFIG_DIRS`
works for LXQt 0.17's config files** — is resolved by the § L0 verdict now
committed to `main`: it does, for all five seeded files, with no fallback
needed. The verdict also supplied and corrected the exact config keys
(§ What Changes item 1), closing the design gaps this intake previously
deferred (clock format, wallpaper mode key, quick-launch entry shape, icon
theme).

## Clarifications

### Session 2026-09-10 (fab-clarify pfe4)

The lxqt plan's § L0 verdict (`fab/plans/sahil/26-09-10-gui-lxqt-desktop.md`)
landed on `main` after this intake was drafted. This session folds its
findings in, closing the one Unresolved row and correcting four Confident/
Certain rows the plan itself amended after the spike.

| # | Action | Detail |
|---|--------|--------|
| 1 | Resolved (Unresolved → Certain) | `XDG_CONFIG_DIRS` confirmed to work for all seeded files; no `XDG_CONFIG_HOME`/`RK_USER_CONFIG_HOME` fallback implemented |
| 3 | Corrected | Clock format keys (`formatType=custom, useAdvancedManualFormat=true, customFormat=HH:mm`, not `timeShowSeconds`); wallpaper mode `none` + `BgColor` (not `color`, which pcmanfm-qt doesn't recognize) |
| 4 | Resolved (Confident → Certain) | Theme name `dark`; `icon_theme` set by a probe over `/usr/share/icons` (`breeze-dark` → `Papirus-Dark` → `Adwaita` → omitted), since `lxqt-core` ships no icon theme |
| 9 | Added | Quick-launch entries seeded as `.desktop` paths, not raw `exec=` (a verdict finding — an unresolved raw `exec=` renders no button) |
| 10 | Added | Locker suppression is an `autostart/…Hidden=true` seed shadowing the system autostart entry, not a `session.conf` module — the verdict's actual mechanism |

Also updated (non-Assumptions-table): § Origin's load-bearing gate note (now
resolved, not blocking); § Why's L-D4 grading (Likely → Certain); § What
Changes items 0–4 (prerequisite check simplified to a mechanical re-verify,
seed content and Go doc comment carry the five verified files and corrected
keys, the supervisor-wiring fallback branch removed, the integration test's
hex typo `#3b4256` corrected to the verdict's measured `#3b4252`, docs
paragraph drops the fallback-documentation branch); § Open Questions closed
out.

## Assumptions

| # | Grade | Decision | Rationale | Scores |
|---|-------|----------|-----------|--------|
| 1 | Certain | LXQt 0.17 honors `XDG_CONFIG_DIRS` for all five seeded files (`session.conf`, `panel.conf`, `lxqt.conf`, `pcmanfm-qt/lxqt/settings.conf`, the autostart override) — L-D4's primary mechanism, no `XDG_CONFIG_HOME`/`RK_USER_CONFIG_HOME` fallback needed | Clarified — resolved by the § L0 verdict now committed to `main`: `startlxqt` only *appends* system dirs to a caller-supplied `XDG_CONFIG_DIRS`, so a prepended seed dir stays first; verified live with an empty `XDG_CONFIG_HOME` and all four original files (plus the autostart shadow) honored | S:95 R:90 A:95 D:95 |
| 2 | Certain | Seed via `<state>/run-kit/gui/lxqt/etc` prepended to `XDG_CONFIG_DIRS`, write-once per file, regenerated panel quick-launch entries — mirrors the G1 IceWM seed precedent exactly | Plan L-D4 (confirmed, no longer conditional) and L-D5; the code-server `settings.json` write-once precedent G1 already established; verdict confirms the mechanism is exactly the one Ubuntu's own `/usr/share` defaults use | S:90 R:75 A:90 D:85 |
| 3 | Certain | Seed content, corrected per the verdict: `session.conf` → `openbox` WM + `leave_confirmation=false` + `lock_screen_before_power_actions=false`, no powermanagement module; `panel.conf` → one bottom panel (menu, 2 quick-launch, taskbar, tray, statusnotifier, minutes-only clock via `formatType=custom, useAdvancedManualFormat=true, customFormat=HH:mm`); `lxqt.conf` → `theme=dark`; `pcmanfm-qt/lxqt/settings.conf` → `WallpaperMode=none` + `BgColor=#3b4252`, icons off | Plan L-D5 as corrected by the § L0 verdict: `color` is not a valid `WallpaperMode` (use `none`, which is the solid-`BgColor` mode); the plan's `timeShowSeconds`/`short-timeonly` combination does not suppress seconds — `formatType=custom` does; every key verified live on 0.17.1 and checked for drift against lxqt master | S:95 R:80 A:90 D:90 |
| 4 | Certain | Theme name `dark` (from `lxqt-themes`); `icon_theme` set by probing `/usr/share/icons` at seed time for the first of `breeze-dark`, `Papirus-Dark`, `Adwaita`, key omitted if none present (`lxqt-core` ships no icon theme) | Clarified — the § L0 verdict names `dark` as the theme used and confirms `icon_theme` is read from the seed layer (probe: removing the key drops the quick-launch icon to a text label); `lxqt-core`'s lack of a bundled icon theme means a probe, not a fixed name, is the correct mechanism | S:90 R:85 A:90 D:85 |
| 5 | Certain | Seeding runs only when the resolved WM is a member of the LXQt session-starter names (`startlxqt`, `lxqt-session`) — never for IceWM, bare WMs, or XFCE | Plan L-D1/L-D8 — LXQt is the one seeded DE; XFCE stays reachable but unseeded | S:90 R:85 A:90 D:90 |
| 6 | Certain | File permissions and write-once granularity mirror G1's IceWM seed exactly (dir `0700`, files `0600`, whole-file write-once, delete-to-re-seed) | No LXQt-specific permission requirement is stated in the plan; the existing G1 precedent is the established pattern this change extends | S:75 R:80 A:85 D:80 |
| 7 | Certain | The supervisor's env-var wiring (`XDG_CONFIG_DIRS=<dir>:${XDG_CONFIG_DIRS:-/etc/xdg}`) is set only for the LXQt rung, using the same "env builder is already rung-specific" mechanism the `ICEWM_PRIVCFG` precedent established, with no conditional fallback branch to implement | Plan L2 Do item 2 states this explicitly; mirrors the existing rung-specific env pattern; the verdict removes the need for a second (fallback) code path, simplifying the decision | S:85 R:85 A:85 D:85 |
| 8 | Confident | The integration test extends/parallels the S1-era `cmd/rk/gui_supervise_integration_test.go` structure (capability-gated on Xtigervnc + startlxqt + dbus-run-session) rather than living in `internal/gui/xvnc_integration_test.go` as the plan's original text suggests | Follows S1's own established deviation from the plan's original file placement (the stamp and `runGuiSuperviseLinux` are `package main`) — same reasoning applies here | S:60 R:80 A:70 D:60 |
| 9 | Certain | Quick-launch entries are seeded as `.desktop` paths (`apps\N\desktop=/usr/share/applications/<name>.desktop`), not raw `exec=` lines, and only for a role that actually resolved | Clarified — the § L0 verdict found a raw `exec=` entry whose binary wasn't on PATH (`x-www-browser` in its test env) rendered no button at all; `.desktop` paths are the reliable form and match how `qterminal.desktop` was seeded successfully | S:90 R:85 A:90 D:85 |
| 10 | Certain | The "no locker" requirement is implemented as an `autostart/lxqt-xscreensaver-autostart.desktop` seed with `Hidden=true`, shadowing the system one — not a `session.conf` module toggle | Clarified — the § L0 verdict found `lxqt-xscreensaver-autostart.desktop` ships with `lxqt-session` itself (not a separate `lxqt-powermanagement`-style module), and confirmed the `XDG_CONFIG_DIRS`-seeded autostart entry with `Hidden=true` suppresses it (probe: hiding `lxqt-runner.desktop` the same way stopped that process from starting) | S:90 R:80 A:90 D:85 |

10 assumptions (9 certain, 1 confident, 0 tentative, 0 unresolved).
