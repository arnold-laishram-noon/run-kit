# Intake: GUI session starters and the `rk gui wm` verb (S1 / L1)

**Change**: 260910-zsui-gui-session-starters-and-wm-verb
**Created**: 2026-09-10

## Origin

> Drafted via `/fab-draft` as stage S1 of the combined GUI execution queue
> (`fab/plans/sahil/26-09-10-gui-combined-execution.md`). Implements § L1
> ("Session starters and the `wm` verb") of
> `fab/plans/sahil/26-09-10-gui-lxqt-desktop.md`. One-shot, no conversational
> discussion — the design record is the lxqt plan's § Decision log (L-D1
> through L-D9, binding except the Likely rows below) and its § UX / § L1 Do
> / § L1 Acceptance sections, which this intake reproduces in full so the
> apply-entry agent (a separate context) never has to re-read the plan to
> recover a value.

Per the combined-execution plan's Coordination point 2, this change **also**
owns the skill-page line-budget trim for `docs/site/skill/gui.md`
(`skill_test.go` enforces a 150-line cap; the file is at 145 today) — S1
frees at least 8 lines before adding its own gotcha, so S2 (which adds `rk
gui resize` documentation next in the queue) rebases over a smaller, already
tidied file. This is explicit scope, not an incidental docs touch.

Per the combined-execution plan's § Rules, S1 is a **base stage**: S2, S3,
S4, S5, S6, S7 all start from a main branch that contains this change merged.
If this change is skipped by the operator, the queue pauses — it does not
silently proceed.

## Why

**The pain.** Today's WM ladder resolves a bare window-manager binary
(`icewm-session, openbox, xfwm4, i3, kwin_x11, x-session-manager`); only
`x-session-manager` is wrapped in `dbus-run-session`. A user who pins
`gui.wm` to `startlxqt` or `startxfce4` (the only way to reach a full desktop
today — editing YAML or the Settings free-text field, then `rk gui restart`)
gets a session that starts with no D-Bus session bus, so its panel, tray, and
policy-agent daemons fail silently or behave unreliably — libdbus autolaunch
on a headless X display is not dependable. There is also no CLI verb that
writes `gui.wm`: today it is Settings-dialog-only or hand-edited YAML, with
no PATH check, no restart chaining, and no refusal message when the pinned
binary is missing.

**The consequence of not fixing it.** Every later stage in the queue depends
on this: S3 (the desktop picker) needs `IsSessionStarter` and the "session"
vs "bare WM" distinction to label candidates; S5 (LXQt seeded defaults) needs
the D-Bus wrap to exist before it can seed a working LXQt session at all — an
unwrapped `startlxqt` cannot reliably run its own panel. Without S1, "switch
to LXQt" stays a three-step manual dance that half-works.

**The approach, and why.** Generalize the existing `x-session-manager`
special case into a table (`sessionStarters`, L-D2): every session-starter
binary (`startlxqt`, `lxqt-session`, `startxfce4`, `xfce4-session`,
`startplasma-x11`, `x-session-manager`) gets wrapped in
`dbus-run-session -- <name>`; bare window managers stay unwrapped. Add
`rk gui wm [auto|icewm|lxqt|xfce|<binary>] [--restart] [--force]` as the CLI
face of the existing `gui.wm` registry key (L-D3), following the `rk gui
on|off` precedent (Constitution IV's single-settings-surface carve-out
already used by that pair). Add per-package-manager install hints (L-D7) so
a refusal is actionable. This stage does **not** implement the LXQt seed
itself (that is S5/L2) and does **not** implement the Settings-dialog picker
control (that is S3/L3) — it only makes a pinned session starter actually
work and gives it a command-line switch.

Alternatives the plan rejected and this intake does not reopen: XFCE/LXQt as
recommended (only LXQt gets a seed, in S5); a picker inside the gui tile
itself (L-D8/L-D9 — the picker lives in Settings/palette, S3's job);
Wayland sessions, KDE/GNOME, per-viewer DEs (L-D8, out of scope everywhere).

## What Changes

All backend (`app/backend/`) plus docs. Every user-facing string below is
plan § UX copy verbatim.

### 1. The session-starter table and the D-Bus wrap — `internal/gui/backend.go`

Per L-D2, add a `sessionStarters` set/table: `startlxqt`, `lxqt-session`,
`startxfce4`, `xfce4-session`, `startplasma-x11`, `x-session-manager`.
`WMArgv` gains the general rule: **any** name in `sessionStarters` returns
`["dbus-run-session", "--", <name>]` (today's `x-session-manager` case
generalizes into the table instead of being special-cased); every other
(bare WM) name is unwrapped, as today (e.g. the existing `icewm-session
--nobg --notray` flags from the G1 change are untouched — those flags stay
attached to the icewm rung specifically, not to the session-starter
mechanism).

```go
// IsSessionStarter reports whether name is a member of the session-starter
// set — used by the status summary suffix (below) and by S3's candidate
// labeling (kind: wm|session).
func IsSessionStarter(name string) bool
```

Table test: every session-starter name gets the `dbus-run-session --` wrap;
every non-member bare WM name (e.g. `openbox`, `icewm-session`) does not;
`IsSessionStarter` returns true/false correctly for members/non-members.

### 2. Per-DE install hints — `internal/gui/hint.go`

Per L-D7, extend the existing package-manager-aware hint machinery (from the
G1 change: `PackageManager`, `WMInstallHint`, `InstallHint`) with a
session-starter-aware hint:

```go
// DEInstallHint returns the install line for a session-starter binary name
// (startlxqt/startxfce4), package-manager-aware like WMInstallHint:
//   apt    "sudo apt install --no-install-recommends lxqt-core" / "xfce4"
//   dnf    "sudo dnf install lxqt-session lxqt-panel lxqt-config pcmanfm-qt qterminal" / <xfce4 dnf line, analogous>
//   pacman "sudo pacman -S lxqt" / <xfce4 pacman line, analogous>
//   ""     "install lxqt with your package manager" / "install xfce4 with your package manager"
func DEInstallHint(name string, lookPath func(string) (string, error)) string
```

The dnf group name is deliberately **not** `@lxqt-desktop` (L-D7 explicitly
rejects it) — dnf uses the explicit package list. Non-apt package names are
Confident, not Certain (see § Assumptions) — they are wording only, never
executed (Constitution I: rk never runs the package manager).

Tests: every hint string for `startlxqt` and `startxfce4` across all four
manager cases (apt/dnf/pacman/none).

### 3. `rk gui wm` — new `cmd/rk/gui_wm.go`

Per L-D3 and § UX:

```
$ rk gui wm
wm: auto → icewm-session (running)
$ rk gui wm lxqt
error: startlxqt not on PATH — sudo apt install --no-install-recommends lxqt-core (pass --force to pin anyway)
$ sudo apt install --no-install-recommends lxqt-core
$ rk gui wm lxqt --restart
set gui.wm=startlxqt
restarted (Xtigervnc :10) — window manager: startlxqt (session, seeded)
$ rk gui wm auto --restart
set gui.wm= (ladder)
restarted (Xtigervnc :10) — window manager: icewm-session
```

- **Alias map**: `auto` ⇒ `""`, `icewm` ⇒ `icewm-session`, `lxqt` ⇒
  `startlxqt`, `xfce` ⇒ `startxfce4`; anything else is taken as a literal
  binary name.
- **No argument**: print the current pin and the resolved rung —
  `wm: auto → icewm-session (running)` or `wm: startlxqt (pinned; running)`
  when a pin resolves, reading `settings.Load().GUIWM` and the live
  supervisor stamp (`@rk_gui_wm`) the same way `rk gui status` does.
- **PATH check with `--force`**: `LookPath` the resolved binary name; if not
  found, refuse: `error: <bin> not on PATH — <DEInstallHint or WMInstallHint>
  (pass --force to pin anyway)`, exit 1. With `--force`, pin anyway (the
  supervisor logs the pin-miss fallback line already added by S1 item 4
  below) and continue to the write/restart path.
- **Write path**: `settings.Load() → set GUIWM to the resolved binary name →
  Save()`, the identical `settings.Load → apply → Save` path `rk gui on`
  uses (no new settings machinery — `gui.wm` already exists from the parent
  G-D6 change).
- **Without `--restart`**: print `set gui.wm=<value> — takes effect on rk gui
  restart (kills apps on the display)` and exit 0.
- **With `--restart`**: after the write, chain into the existing
  `runGuiRestart` path exactly as if the user had typed `rk gui restart`
  next, then print the restart's own output lines.
- **Exit codes**: `0` on a successful set (with or without restart, and
  regardless of gui on/off state for a bare set); `1` on the PATH refusal,
  on a daemon-down error, or on gui-off **specifically when `--restart` was
  requested** (a bare set without `--restart` is allowed while gui is off
  and simply takes effect on the next `rk gui on`); `2` on usage errors (bad
  alias/flag combination — cobra's standard usage-error exit).

Tests: alias resolution table; PATH-check refusal (with and without
`--force`) using the settings-home and lookPath seams (no real X); the
write-then-restart chain using the existing restart-path test seams;
exit-code table above; the bare-set-while-off-succeeds case; `--restart`
while off fails with the off error.

### 4. Supervisor lines — `cmd/rk/gui_supervise.go`

- The existing pin-miss log line gains the DE-aware hint when the pin is a
  recognized session-starter name: `gui: gui.wm=startlxqt not on PATH;
  falling back to the ladder — sudo apt install --no-install-recommends
  lxqt-core` (uses `DEInstallHint` from item 2 when the pin is a
  session-starter name, `WMInstallHint` otherwise — unchanged for a bare-WM
  pin miss).
- The existing WM-found log line gains a `(session under dbus-run-session)`
  suffix when the resolved rung is a session starter:
  `gui: window manager startlxqt (session under dbus-run-session; defaults
  …/run-kit/gui/lxqt/etc, seeded)` — **but** the `defaults …, seeded` segment
  is S5's addition (LXQt seeding does not exist yet in this change); S1's
  version of this line is `gui: window manager startlxqt (session under
  dbus-run-session)` with no defaults segment. A bare-WM line
  (`gui: window manager openbox`) is unchanged.

Tests: the pin-miss line names the DE hint for a session-starter pin, the
plain WM hint for a bare-WM pin; the WM-found line carries the `(session
under dbus-run-session)` suffix exactly when `IsSessionStarter` is true.

### 5. Status summary — `cmd/rk/gui.go`

`guiStatusSummary` (the function backing `rk gui status`'s `wm` segment and
the doctor row) appends `(session)` when the reported WM name is a member of
`IsSessionStarter` — e.g. `on (Xtigervnc, :10, 1920x1080, 1 viewer,
startlxqt (session))` vs `on (Xtigervnc, :10, 1920x1080, 1 viewer,
icewm-session)` for a bare WM. `--json` is unaffected — the raw `wm` field
stays the plain binary name; only the human-readable summary line gains the
suffix, so a reader can tell a DE apart from a bare WM without a second
lookup. Per plan § UX: "nothing else changes shape."

Test: the summary line for a session-starter WM carries `(session)`; for a
bare WM it does not.

### 6. Running-apps exclusion set — `internal/gui/apps_linux.go`

Per L-D6, extend the existing `wmHelperComms` exclusion map (added by the
G1 IceWM change) with the LXQt and XFCE daemon names:

```go
// added to the existing wmHelperComms map:
"lxqt-session": true, "lxqt-panel": true, "lxqt-runner": true,
"lxqt-globalkeysd": true, "lxqt-notificationd": true,
"lxqt-policykit-agent": true, "pcmanfm-qt": true,
"xfce4-session": true, "xfce4-panel": true, "xfdesktop": true,
"xfsettingsd": true, "xfce4-notifyd": true, "xfce4-power-manager": true,
"dbus-daemon": true, "dbus-run-session": true,
```

Same mechanism as the parent G-D7 (filter on trimmed `comm`, after the pid
excludes, by name never ancestry — an app launched from an LXQt panel is a
child of `lxqt-panel` and must stay listed).

Test: a fake `procRoot` carrying `lxqt-panel`, `lxqt-session`, and an
ordinary app (e.g. `xterm`) all with the display's `DISPLAY` env → only the
ordinary app survives `RunningApps`.

### 7. Docs

- `docs/specs/gui.md`: § The supervisor gains the session-starter set and
  the `dbus-run-session` wrap rule, and a short § Switching desktops section
  (new) naming `rk gui wm` and the three example commands from § UX above
  (the fuller "seeded" / "picker" content is S5's and S3's additions — S1's
  version covers only the verb and the wrap). § Agent verbs is unaffected
  (`rk gui wm` is a human/CLI verb, not an agent verb).
- `docs/site/skill/gui.md` (currently 145 of the 150-line cap enforced by
  `skill_test.go`): **the trim required by Coordination point 2**, done as
  part of this change's docs task, before adding anything new:
  - Fold the existing `env`/`exec` gotcha lines into one line.
  - Merge the existing `shot` default-path note and the `--out` note into
    one line.
  - These two mergers must free **at least 8 lines** (verify with `wc -l`
    after editing — the number is a floor, not a target; trim further if a
    clean merge frees more).
  - Then add **one** new gotcha line: the desktop may be IceWM or a full
    desktop environment (LXQt/XFCE); the agent verbs (`rk gui env/exec/shot`
    etc.) work identically regardless; `rk gui windows` lists the DE's panel
    as a window on a full desktop — filter by `app` when looking for user
    apps.
  - Net effect: file length stays comfortably under 150 after the trim +
    1-line addition. The `skill_test.go` guard (which checks the synced copy
    at `cmd/rk/skill/gui.md` matches and is ≤150 lines) must pass; run
    `scripts/sync-skill.sh` after editing.
- `rk gui --help`: `wm` verb listed under the `display` group in
  `guiCmd.Long`'s `Subcommands:` list.
- Memory via hydrate (§ Affected Memory below).

### Acceptance (from the plan, binding)

- With `lxqt-core` installed on this VM, `rk gui wm lxqt --restart` shows an
  LXQt desktop in the tile (default look — unseeded until S5) with
  `rk gui status` reading `startlxqt (session)`.
- `rk gui wm auto --restart` returns to IceWM.
- `rk gui wm xfce` without XFCE installed exits 1 with the apt install line.
- `--force` pins anyway and the supervisor logs the pin-miss fallback line.
- `go test ./...` and `just test` green.
- Without `lxqt-core` installed, S1 still ships fully — its `rk gui wm xfce`
  (or `lxqt`) refusal path is testable via the PATH-check seam without real
  packages; only the live-VM LXQt-desktop-appears acceptance line is skipped
  and noted, per the combined-execution plan's operator checklist item 1.
- `docs/site/skill/gui.md` and its synced copy are ≤150 lines and the
  `skill_test.go` guard passes.

## Affected Memory

- `run-kit/gui`: (modify) the generalized session-starter table and the
  `dbus-run-session` wrap rule (superseding the special-cased
  `x-session-manager` note); `rk gui wm` verb (alias map, PATH refusal,
  `--force`, `--restart` chaining, exit codes); the `(session)` suffix on
  `guiStatusSummary`; the DE-aware install hint (`DEInstallHint`); the
  widened `wmHelperComms` exclusion set; new Design Decision (LXQt is the
  one supported full desktop; session starters generalize the D-Bus wrap
  rule rather than special-casing each DE)
- `run-kit/configuration`: (modify) a short note that `gui.wm` now has a
  first-class CLI setter (`rk gui wm`) alongside the Settings dialog and
  hand-edited YAML — no schema change, the key already exists

## Impact

**Code (`app/backend/`)**

- `internal/gui/backend.go`: `sessionStarters` table, `WMArgv` generalization,
  `IsSessionStarter` — with table test.
- `internal/gui/hint.go`: `DEInstallHint` — with tests for all manager cases
  × two DE names.
- `cmd/rk/gui_wm.go` (new): the verb, its alias map, PATH check, settings
  write, restart chaining — with tests via the settings-home/lookPath seams
  (no real X needed).
- `cmd/rk/gui_supervise.go`: pin-miss line gains the DE hint; WM-found line
  gains the session suffix — with tests.
- `cmd/rk/gui.go`: `guiStatusSummary` appends `(session)` — with test.
- `internal/gui/apps_linux.go`: widened exclusion map — with test.

**Docs**: `docs/specs/gui.md` (§ The supervisor, new § Switching desktops),
`docs/site/skill/gui.md` (trim + one new gotcha, synced copy), `rk gui
--help`, memory via hydrate.

**Dependencies**: none new. `lxqt-core`/`xfce4` are host packages the user
installs; the change works (refusal path testable) without them present on
CI or a dev box.

**Tests**: Go unit tests throughout, all seam-based (no real X server, no
tmux session required for this stage — the S1 acceptance's live-VM step is
manual, separate from `go test`/`just test`). Gates: `cd app/backend && go
test ./...`, `just test`, `just build`.

**Behavior change for existing hosts**: none by default — `gui.wm` stays
empty (ladder auto-resolve) unless a user explicitly pins a session-starter
name, which today already half-works via YAML/Settings; this change makes
that pin actually function (D-Bus wrap) and adds the CLI shortcut. No
existing pin's resolved binary changes.

## Open Questions

None. The lxqt plan's § Decision log resolves every design point in scope
for L1 and its § UX fixes the exact copy. The two rows the plan itself marks
Likely (L-D4's `XDG_CONFIG_DIRS` seed mechanism, and L-D7's non-apt package
names) are out of L1's scope for the first, and recorded as a Confident
assumption for the second — not open questions for this stage.

## Assumptions

| # | Grade | Decision | Rationale | Scores |
|---|-------|----------|-----------|--------|
| 1 | Certain | Session starters (`startlxqt`, `lxqt-session`, `startxfce4`, `xfce4-session`, `startplasma-x11`, `x-session-manager`) all wrap in `dbus-run-session -- <name>`; bare WMs stay unwrapped; `x-session-manager`'s existing wrap generalizes into the table rather than staying a special case | Plan L-D2; a pinned DE without a session bus is the first failure a user hits | S:95 R:80 A:95 D:95 |
| 2 | Certain | `rk gui wm [auto\|icewm\|lxqt\|xfce\|<binary>] [--restart] [--force]` exactly per § UX copy, including alias map, PATH refusal message, exit codes (0/1/2), and the `settings.Load → apply → Save` write path `rk gui on` already uses | Plan L-D3; the `on|off` precedent is the existing Constitution IV carve-out | S:95 R:75 A:95 D:95 |
| 3 | Certain | A name not on PATH refuses with the DE/WM install line unless `--force` (pins anyway; supervisor logs the pin-miss fallback) | Plan L-D3 verbatim | S:90 R:80 A:95 D:90 |
| 4 | Confident | Non-apt package names for LXQt/XFCE: dnf `lxqt-session lxqt-panel lxqt-config pcmanfm-qt qterminal` (not the `@lxqt-desktop` group) / analogous xfce4 line; pacman `lxqt` / analogous xfce4 line | Plan marks L-D7's non-apt names Likely — from packaging knowledge, unverified on a real dnf/pacman host; wording only, never executed | S:65 R:95 A:55 D:65 |
| 5 | Certain | `IsSessionStarter(name)` is exported for reuse by S3 (candidate `kind: wm|session` labeling) and by the status-summary `(session)` suffix here | Plan L1 Do item 1 names both consumers explicitly; one shared predicate avoids duplicating the set | S:85 R:90 A:90 D:90 |
| 6 | Certain | Running-apps exclusion set widens with the full LXQt/XFCE daemon name list from L-D6, applied by comm name after pid excludes, never by ancestry | Plan L-D6 verbatim; matches the existing G-D7 mechanism this change extends | S:95 R:90 A:95 D:95 |
| 7 | Certain | `guiStatusSummary` appends `(session)` only to the human-readable summary line; `--json`'s `wm` field stays the plain binary name | Plan § UX "Nothing else changes shape"; avoids a breaking JSON-contract change for S3/frontend consumers | S:85 R:85 A:90 D:85 |
| 8 | Certain | This stage does the docs/site/skill/gui.md trim (fold env/exec gotchas to one line, merge shot default-path/--out notes to one line), freeing ≥8 lines, before adding its own one-line gotcha | Combined-execution plan Coordination point 2 — explicitly assigned to S1 so S2 rebases over a smaller diff | S:90 R:85 A:80 D:90 |
| 9 | Certain | This change does NOT seed LXQt (`XDG_CONFIG_DIRS`, the four config files) and does NOT build the Settings-dialog picker control — those are S5/L2 and S3/L3 respectively | Plan's explicit L1/L2/L3 split; combined-execution queue orders S1 before S3 and S5 for exactly this reason | S:90 R:85 A:90 D:90 |
| 10 | Confident | The supervisor's WM-found log line in this change reads `(session under dbus-run-session)` with no `defaults …, seeded` segment (that segment is S5's addition once seeding exists) | Plan's L1 vs L2 Do lists show the seeded-defaults segment introduced in L2's supervisor edit, not L1's | S:60 R:85 A:75 D:65 |

10 assumptions (7 certain, 3 confident, 0 tentative, 0 unresolved).
