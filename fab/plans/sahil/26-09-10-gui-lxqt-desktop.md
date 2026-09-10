# GUI full-desktop option — LXQt as the supported desktop environment, and a one-command switch

> Plan doc — written 2026-09-10 from the bronze-crane `/fab-discuss` thread.
> Child of [`26-09-10-gui-desktop.md`](26-09-10-gui-desktop.md) (IceWM as the
> default rung, the seeded profile, the `gui.wm` pin, the launcher, the agent
> verbs — G1–G3 all merged) and grandchild of
> [`26-09-09-gui-surface.md`](26-09-09-gui-surface.md). This doc owns the
> "I want a real desktop environment" path: which DE rk supports beyond the
> IceWM default, how a user switches to it from the command line, and how rk
> keeps its look and its bandwidth discipline when a DE is running. It does
> not reopen G-D1 (IceWM stays the default) or G-D9 (rk recommends no DE to
> users who have not asked for one).

**Goal**: a user who wants a full desktop in the gui tile installs one
package, runs one rk command, and gets a seeded LXQt desktop that behaves
under the relay like the IceWM one — solid ground, no locker, no compositor,
no idle screen churn — with IceWM one command away again.

**Status (2026-09-10)**: not started. L0 (spike) first; L1 and L2 follow.

---

## Where things stand today (as merged on `main`)

- The ladder is `icewm-session → openbox → xfwm4 → i3 → kwin_x11 → x-session-manager`;
  `gui.wm` pins a binary name ahead of it (G-D6).
- **A pinned session starter does not get a D-Bus session.** `WMArgv` wraps
  only `x-session-manager` in `dbus-run-session`; a pinned `startlxqt` or
  `startxfce4` runs bare and its session, panel, and policy agent then depend
  on libdbus autolaunch, which is unreliable on a headless X display.
- **There is no CLI verb that writes a settings key.** `gui.enabled` has
  `rk gui on|off`; `gui.wm` has only the Settings dialog and
  `~/.config/run-kit/config.yaml`.
- So the honest answer to "can we switch to LXQt from the command line today"
  is **not in one step**: install `lxqt-core`, set `gui.wm: "startlxqt"` in
  the dialog or the YAML, run `rk gui restart`, and hope D-Bus autolaunches.
  L1 makes it `rk gui wm lxqt --restart`.
- Neither DE has a seeded config. A fresh LXQt shows its default panel and
  wallpaper; a fresh XFCE additionally shows the first-run panel wizard and
  runs the xfwm4 compositor, which the relay pays for on every update.

---

## Why LXQt and not XFCE (decided here)

Both are ~40 packages without recommends on Ubuntu 22.04 and both idle far
above IceWM's 29 MB. The differences that matter to **this** use — a
headless Xvnc display, seeded by rk, watched over a bandwidth-bound relay,
shared with an agent — all favor LXQt:

| Concern | LXQt | XFCE |
|---|---|---|
| Config model | plain INI files under `lxqt/` — seedable by writing files | `xfconf` daemon + XML; first panel start shows a wizard unless pre-seeded; the compositor toggle lives inside xfconf |
| Compositor | none (openbox) | xfwm4 composites by default — must be switched off for VNC |
| Idle daemons in the minimal install | session, panel, runner, policykit agent, notifications, globalkeys; power management and a locker are **separate** packages | session, panel, desktop painter, settings daemon, power manager; a screensaver/locker usually comes along and would lock a passwordless display |
| Maturity / reach | smaller team; jammy ships 0.17, current is 2.x on Qt6 (config-key drift risk) | 4.16 everywhere, universally documented |
| Native apps | pcmanfm-qt, qterminal, featherpad, lximage-qt, screengrab — thinner set | Thunar, xfce4-terminal, mousepad, ristretto, taskmanager — fuller, more polished |

"Enough apps" is not the deciding axis: the gui surface exists so a human can
watch and steer an agent that mostly drives a browser and a terminal. Both
DEs have a file manager, a terminal, an image viewer, and an editor, and both
run any GTK/Qt app; apps show up in both menus through the same desktop-file
scan IceWM uses. What decides it is seeding and relay cost, and those favor
LXQt. XFCE stays **reachable** (`rk gui wm xfce` after L1) but **unseeded**
and documented with its compositor caveat only.

---

## Decision log (Certain — intakes do not re-open these)

| # | Decision | Why |
|---|----------|-----|
| L-D1 | **LXQt is the one supported desktop environment**; XFCE and others are reachable through the pin but get no seed, no test, and one documentation line each | § Why LXQt; G-D9 keeps DEs off the recommended path; supporting one is affordable, two is not |
| L-D2 | **Session starters run under `dbus-run-session` generically.** `WMArgv` gains a session-starter set — `startlxqt`, `lxqt-session`, `startxfce4`, `xfce4-session`, `startplasma-x11`, `x-session-manager` — every member wrapped as `dbus-run-session -- <name>`; bare window managers stay unwrapped. The set is one table beside the ladder | a pinned DE without a session bus is the failure the user hits first; `x-session-manager` already has the wrap, the rule just generalizes |
| L-D3 | **`rk gui wm [auto\|icewm\|lxqt\|xfce\|<binary>] [--restart] [--force]`** — the CLI face of the `gui.wm` key. No argument prints the current pin and the resolved rung (`wm: auto → icewm-session`). Aliases: `auto` ⇒ `""`, `icewm` ⇒ `icewm-session`, `lxqt` ⇒ `startlxqt`, `xfce` ⇒ `startxfce4`; anything else is a binary name. A name not on PATH **refuses** with the DE's install line unless `--force` (pins anyway; the supervisor logs the pin-miss fallback). Writes through the same `settings.Load → apply → Save` path `rk gui on` uses. Without `--restart` it prints `set gui.wm=startlxqt — takes effect on rk gui restart (kills apps on the display)`; with it, it runs the restart verb | the `rk gui on|off` precedent for a gui-scoped write of one registry key; Constitution IV's single settings *surface* is the dialog — a CLI setter for the key the gui verbs own is the same carve-out `on|off` already uses |
| L-D4 | **rk seeds LXQt through `XDG_CONFIG_DIRS`, not `XDG_CONFIG_HOME`.** The supervisor prepends `<state>/run-kit/gui/lxqt/etc` to `XDG_CONFIG_DIRS` for the session; the seeded files live under `…/etc/lxqt/` (`session.conf`, `panel.conf`, `lxqt.conf`) and `…/etc/pcmanfm-qt/lxqt/settings.conf`, written when absent (write-once, delete to re-seed). LXQt reads `XDG_CONFIG_DIRS/lxqt/*.conf` as system defaults and the user's `~/.config/lxqt` as overrides | `XDG_CONFIG_HOME` would be inherited by every app launched from the desktop and move the user's chromium/editor profiles under rk's state dir — surprising and hard to undo. Defaults-dir seeding gives rk its look on a fresh host and lets a user who also runs LXQt locally keep their own preferences. **Likely, not Certain**: L0 verifies that LXQt 0.17 honors `XDG_CONFIG_DIRS` for `panel.conf` and `session.conf`; if it does not, the fallback is a dedicated `XDG_CONFIG_HOME` **plus** re-exporting the user's real one as `RK_USER_CONFIG_HOME` and documenting it |
| L-D5 | **Seed content**: `session.conf` — window manager `openbox`, no `lxqt-powermanagement`, no screensaver/locker module, no `xscreensaver`; `panel.conf` — one bottom panel: main menu · quick-launch (terminal, browser via the G1 launcher's resolved binaries) · task bar · tray · clock **without seconds** (one repaint a minute, not one a second); `lxqt.conf` — a dark theme from `lxqt-themes` (L0 picks); `pcmanfm-qt/lxqt/settings.conf` — desktop wallpaper mode `color`, color `#3b4252`, desktop icons off. The supervisor's `xsetroot` paint still runs (the desktop window covers it; harmless) | the relay pays per changed rect (parent C5): a seconds clock is 60 rects a minute for nothing; a locker on a passwordless display locks the user out; icons on the desktop are churn on every resize |
| L-D6 | **Running-apps exclusion set grows** with the DE daemons: `lxqt-session, lxqt-panel, lxqt-runner, lxqt-globalkeysd, lxqt-notificationd, lxqt-policykit-agent, pcmanfm-qt` (desktop mode), `xfce4-session, xfce4-panel, xfdesktop, xfsettingsd, xfce4-notifyd, xfce4-power-manager`, `dbus-daemon`, `dbus-run-session`. Same mechanism as G-D7 | `rk gui status` and the off-confirm list must not read `lxqt-panel ×1` as a user app |
| L-D7 | **Install hints per DE**, package-manager-aware like G-D2: apt `sudo apt install --no-install-recommends lxqt-core` / `xfce4`; dnf `@lxqt-desktop` group is **not** used — `sudo dnf install lxqt-session lxqt-panel lxqt-config pcmanfm-qt qterminal`; pacman `sudo pacman -S lxqt`; the XFCE lines analogous. Non-apt names Likely | consistency with the backend and WM hints; the `rk gui wm` refusal and the supervisor's pin-miss line both print them |
| L-D8 | **Out of scope**: XFCE seeding; KDE/GNOME; Wayland sessions; installing anything; per-viewer DEs; a picker *inside the tile* (the picker lives in Settings and the palette — L-D9) | G-D9, the parent's D10; one supported DE |
| L-D9 | **One desktop picker, two doors.** The `gui.wm` row in the Settings dialog becomes a picker instead of a free-text field: `Auto (ladder)` first, then every **installed** candidate the server detected (`IceWM`, `LXQt`, `XFCE`, plus bare WMs by name), then `Other…` for a typed binary. The palette gains `GUI: Desktop…` opening the same picker. Installed candidates come from the server — `GET /api/gui/{id}` gains `wm_candidates: [{name, label, kind: wm\|session, installed: true}]`, derived by `LookPath` over the ladder ∪ session-starter set on the status read (no stream field: it changes only when packages change). Choosing a value writes `gui.wm` through `POST /api/settings` and then asks `Restart the desktop now? Running apps will close: <apps>` — the off-confirm dialog's list — with `Restart` / `Later`. `Later` leaves the pin set for the next `rk gui on`/`restart`. Candidates not installed are not listed; a footer line says `Install more: sudo apt install --no-install-recommends lxqt-core` (the L-D7 hint for the supported DE) | Constitution V: a UI control needs a palette row, and a free-text binary name is a poor control for a choice with three real answers; Constitution II/X: which desktops are installed is derivable server-side and never stored; the restart is destructive, so it confirms like `off` does |

---

## UX (final)

### `rk gui wm`

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

Exit `0` on set; `1` on the PATH refusal, daemon down, or gui off (`--restart`
only — a bare set is allowed while off and takes effect on `rk gui on`); `2`
on usage.

### Supervisor pane

```
gui: window manager startlxqt (session under dbus-run-session; defaults …/run-kit/gui/lxqt/etc, seeded)
gui: gui.wm=startlxqt not on PATH; falling back to the ladder — sudo apt install --no-install-recommends lxqt-core
```

### `rk gui status` / doctor / stream

`wm` carries the binary name as today (`startlxqt`); the status summary
appends `(session)` for a member of the session-starter set so a reader can
tell a DE from a bare WM. Nothing else changes shape.

### Skill page (`rk skill gui`)

One new paragraph under § Gotchas: the desktop may be IceWM or LXQt; the
agent verbs (G3) work identically; `rk gui windows` lists the DE's panel as a
window on LXQt — filter by `app` when looking for user apps.

### Docs

`docs/specs/gui.md` § The supervisor: the session-starter set, the
`XDG_CONFIG_DIRS` seed, the `wm` verb. § Switching desktops (new, short): the
three commands above, then one line for XFCE (`rk gui wm xfce`, unseeded,
"turn off xfwm4 compositing in Settings → Window Manager Tweaks or the relay
pays for it").

---

## Change breakdown

| # | Slug (suggested) | Depends on | Size | Change folder | PR | Status |
|---|------------------|-----------|------|---------------|----|--------|
| L0 | *(spike — no fab change; § L0 verdict appended here)* | — | S | — | — | not started |
| L1 | `gui-session-starters-and-wm-verb` | parent G3 merged (it is) | S | | | not started |
| L2 | `gui-lxqt-seeded-defaults` | L0 verdict, L1 merged | M | | | not started |
| L3 | `gui-desktop-picker` | L1 merged (∥ L2) | S | | | not started |

L1 is useful alone: it makes any pinned DE actually start (D-Bus) and gives
the switch a command. L2 is the LXQt-specific seed and depends on what L0
learns about `XDG_CONFIG_DIRS` and the 0.17 config keys. L3 is the frontend
picker (L-D9) and needs only L1's candidate list; it can run beside L2.
Three changes, not one, so a wrong L-D4 guess costs only L2 and the
frontend review stays separate, as with the parent's G2.

---

### L0 — Spike (half a day, this VM, nothing merged)

**Do**: `sudo apt install --no-install-recommends lxqt-core` (38 packages
here). On a throwaway display (`Xtigervnc :9x` with the production argv, a
short socket path under `$XDG_RUNTIME_DIR`): `dbus-run-session -- startlxqt`
with the seeded files placed under a temp `XDG_CONFIG_DIRS` entry. Record:

1. Does 0.17 read `panel.conf` / `session.conf` / `lxqt.conf` from
   `XDG_CONFIG_DIRS` when `~/.config/lxqt` has no such file? (decides L-D4)
2. Idle RSS of the session's process tree; idle relay Mbit/s over 60 s with
   no viewer input, measured with the C5 harness (`scripts/gui-perf-link.sh`
   is not needed — loopback is enough), clock with and without seconds.
3. First-start screenshot at 1280×800 and 536×799 (does the panel reflow;
   does pcmanfm-qt's desktop honor color mode and icons-off).
4. SIGTERM to `dbus-run-session` — do all `lxqt-*` processes exit; does the
   launched `qterminal` survive as the IceWM case did.
5. `startxfce4` under `dbus-run-session` once, for the documentation line
   only: confirm the compositor is on by default and where the toggle lives.

Append the results as § L0 verdict below (numbers, screenshots viewed not
committed, the exact config keys that worked).

### L1 — Session starters and the `wm` verb

**Do** (`app/backend/`, `docs/`):
1. `internal/gui/backend.go` — `sessionStarters` table (L-D2);
   `WMArgv` wraps members in `dbus-run-session --`; `IsSessionStarter(name)`
   for the status suffix. Table test.
2. `internal/gui/hint.go` — `DEInstallHint(name)` for `startlxqt`/`startxfce4`
   per package manager (L-D7).
3. `cmd/rk/gui_wm.go` (new) — the verb per § UX: alias map, PATH check with
   `--force`, `settings.Load → GUIWM → Save`, `--restart` chaining into
   `runGuiRestart`. Tests with the settings-home and lookPath seams; no X.
4. `cmd/rk/gui_supervise.go` — pin-miss line carries the DE hint when the
   pin is a known DE name; the WM line adds `(session under dbus-run-session)`.
5. `cmd/rk/gui.go` — `guiStatusSummary` appends `(session)`.
6. `internal/gui/apps_linux.go` — L-D6 exclusion names. Test.
7. Docs: spec § The supervisor + § Switching desktops; skill page gotcha;
   `rk gui --help` lists `wm` under the `display` group; memory via hydrate.

**Acceptance**: with `lxqt-core` installed on this VM, `rk gui wm lxqt
--restart` shows an LXQt desktop in the tile (default look — unseeded until
L2) with `rk gui status` reading `startlxqt (session)`; `rk gui wm auto
--restart` returns to IceWM; `rk gui wm xfce` without XFCE installed exits 1
with the apt line; `--force` pins and the supervisor logs the fallback;
`go test ./...` and `just test` green.

### L3 — The desktop picker (Settings row + `GUI: Desktop…`)

**Do**:
1. `api/gui.go` — `wm_candidates` on the status document (L-D9): iterate the
   ladder ∪ session-starter set through the `LookPath` seam, label map
   (`icewm-session` → `IceWM`, `startlxqt` → `LXQt`, `startxfce4` → `XFCE`,
   others by name), `kind` from `IsSessionStarter`. Handler test.
2. `app/frontend/src/components/settings-dialog*.tsx` — the `gui.wm` row
   renders as a select when the gui status has been fetched: `Auto (ladder)`,
   the installed candidates, `Other…` (reveals the text field). On change →
   `POST /api/settings {"gui.wm": …}` → the restart confirm (reusing the
   `gui-off-dialog` shell with new copy and the running-apps list from the
   status doc) → `POST /api/gui/host/restart` on `Restart`.
3. `src/lib/palette/gui.ts` — `gui-desktop` row `GUI: Desktop…` when
   `enabled`, opening the same picker as a palette sub-list (candidates as
   rows, `Auto` first, current marked).
4. Footer hint line from `wm_hint`-style server text (the L-D7 apt line for
   LXQt when it is not installed).
5. Tests: vitest for the select's states (candidates present / none but
   auto / Other…), the confirm → restart chain, the palette row gating;
   Playwright step with a stubbed status doc listing two candidates: pick
   `LXQt` → confirm dialog names the running apps → `Later` leaves the
   setting posted and no restart call; `Restart` posts restart.

**Acceptance**: on this VM with `lxqt-core` installed, Settings → GUI →
Desktop shows `Auto (ladder)`, `IceWM`, `LXQt`, `Other…`; choosing `LXQt`
and `Restart` brings up the LXQt desktop in the tile; `Cmd+K` → `GUI:
Desktop…` → `IceWM` → `Restart` brings IceWM back; with `lxqt-core` removed
the LXQt row is absent and the footer shows the apt line. `just test` green.

### L2 — LXQt seeded defaults

**Do**:
1. `internal/gui/seed_lxqt.go` (new) — `go:embed` the four files (keys from
   the L0 verdict); `SeedLXQtDefaults(dir, resolved)` writes them when
   absent under `<state>/run-kit/gui/lxqt/etc/…`, regenerates only the
   panel's quick-launch entries from the G1 launcher resolution (the
   `toolbar` rule from G-D3). Tests on a temp dir.
2. `cmd/rk/gui_supervise.go` — when the resolved WM is `startlxqt` (or
   `lxqt-session`), seed, then set `XDG_CONFIG_DIRS=<dir>:${XDG_CONFIG_DIRS:-/etc/xdg}`
   in the WM env (the icewm rung's `ICEWM_PRIVCFG` precedent — the env
   builder is already rung-specific). Log `defaults <dir>, seeded` on first
   start.
3. Integration test (`xvnc_integration_test.go`, gated on `Xtigervnc` +
   `startlxqt` + `dbus-run-session`): supervise on a temp state dir, assert
   the `@rk_gui_wm` stamp is `startlxqt`, the four files exist, a
   `RunningApps` scan lists none of the L-D6 names, and a screenshot's pixel
   at the desktop center is `#3b4252`.
4. Docs: spec § Switching desktops gains the seed paragraph and the
   "delete the dir to re-seed" rule; memory via hydrate.

**Acceptance**: fresh `<state>/gui/lxqt`, `rk gui wm lxqt --restart` → the
tile shows one bottom panel with menu, two quick-launch buttons, task bar,
tray, and a minutes-only clock over a solid `#3b4252` desktop with no icons;
idle relay traffic over 60 s stays within 2× the IceWM idle figure from L0;
`rk gui off` confirm lists no `lxqt-*` process; a hand-edited
`…/etc/lxqt/panel.conf` survives `rk gui restart`.

---

## Constitution mapping

| Principle | How this plan honors it |
|---|---|
| I Security First | `dbus-run-session -- <name>` and every seed path are argv slices; the verb's PATH check uses `LookPath` |
| II No Database | the pin is the existing registry key; the seed dir is a defaults artifact (delete = reset); nothing new is read as truth at request time |
| III Wrap, Don't Reinvent | LXQt draws the desktop; rk writes four INI files and an env var |
| IV Minimal Surface Area | one CLI verb over an existing key (the `on|off` carve-out); no new routes; the picker replaces the existing `gui.wm` field in the one settings surface rather than adding a second; the palette row is the V-mandated door to it |
| V Keyboard-First | `GUI: Desktop…` in the palette reaches the same picker the dialog shows |
| VII Convention Over Configuration | `auto` remains the default; the aliases are conveniences over binary names |
| Toolkit `install-composition` | probe, degrade, hint; rk installs nothing; the hint is per package manager |

## Risks

| # | Risk | Mitigation |
|---|---|---|
| 1 | LXQt 0.17 (jammy) ignores `XDG_CONFIG_DIRS` for some of the four files | L0 decides; the L-D4 fallback is a dedicated `XDG_CONFIG_HOME` with the user's real one re-exported and documented |
| 2 | Config-key drift between 0.17 and 2.x breaks the seed on newer hosts | seed only keys present in both (L0 checks the 2.x docs); unknown keys are ignored by QSettings, not fatal |
| 3 | A DE's autostart pulls in a locker or power manager from a user's own `~/.config/autostart` | rk's `session.conf` disables the modules it knows; a user's autostart is theirs — documented, not fought |
| 4 | `dbus-run-session` absent on a minimal host | it ships with `dbus`, which every DE depends on; the resolver logs and runs the starter bare if it is missing, matching today |
| 5 | `rk gui wm … --restart` kills the agent's apps mid-task | the verb prints the warning; the G3 human-input guard does not apply to the human's own CLI; the operator is told the restart is destructive in the skill page |
| 6 | Idle relay traffic from the panel exceeds the IceWM baseline noticeably | L-D5's minutes-only clock and no desktop icons; L0 measures before L2 commits |

## Pickup protocol

1. Read this file, the parent plan's § Decision log and § Spike verdict, the
   grandparent's § C5 verdict, `docs/specs/gui.md`, the constitution, and
   the memory files `gui`, `configuration`, `daemon-lifecycle`.
2. Treat § Decision log here as Certain except L-D4 (Likely until L0) and
   the non-apt package names in L-D7 (Likely).
3. L0 runs on throwaway displays only — never the live `rk-gui` session.
4. Fill your row in § Change breakdown; mark Done when merged; add a pointer
   row to the parent plan in the same PR.
