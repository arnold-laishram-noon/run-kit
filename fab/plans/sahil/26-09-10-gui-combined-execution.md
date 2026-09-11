# GUI combined execution — one merge-auto queue over the LXQt and viewer-ergonomics plans

> Operator handover — written 2026-09-10 from the azure-eagle thread,
> **executed 2026-09-10 16:34Z → 2026-09-11 04:20Z**, closed out 2026-09-11.
> This file sequenced two sibling plans into one autopilot queue:
> [`26-09-10-gui-lxqt-desktop.md`](26-09-10-gui-lxqt-desktop.md) (L0–L3: the
> supported desktop environment and the `rk gui wm` switch) and
> [`26-09-10-gui-viewer-ergonomics.md`](26-09-10-gui-viewer-ergonomics.md)
> (V1–V4: fixed geometry, zoom, touch pointer modes, quality, toolbar). It
> decided nothing about either plan's design — every stage points at the
> section that owns it. It decided only **order, gates, and the three
> coordination points** the two plans shared. § Execution record and
> § Deviations and lessons are the post-run additions; the design sections
> are kept as written so the reasoning stays legible next to what happened.

**Status (2026-09-11)**: **Queue complete.** S0 verdict on main; S1–S7 merged
as PRs #916, #919 (+ follow-up #920), #929, #931, #932, #933, #934; the seven
change folders archived (`63540359`); both child plans' breakdown tables read
Done. Wall clock 11 h 46 min from the first PR opening to the last merge.
One deviation worth a rule change (§ Deviations → the arm-before-idle race).

**Strategy**: `merge-auto` — one serial queue; the operator arms each PR with
GitHub auto-merge on completion and spawns the next stage only on the tick
that verifies the merge, after `git fetch origin` and a rebase onto
`origin/main`. Implicit `--base` chaining is off in this mode, so the queue
declares **no `depends_on`** — the order below *is* the dependency graph, and
every stage starts from a main that already contains its predecessors.

**Why serial, not two lanes**: `merge-auto` is a single lane by construction.
The two plans could have run as two concurrent lanes (their code is nearly
disjoint), but the price would have been a second operator or the
`cherry-pick-ladder` mode, plus rebasing over each other's edits to
`cmd/rk/gui.go`, `gui_supervise.go`, the palette file, the spec, and the
skill page. Serial-with-good-ordering cost one stage-length of latency per
pair and removed every mid-flight conflict, because each stage was created
after the previous one merged. The only genuinely concurrent work was the L0
spike (§ S0), which is not a fab change and ran beside the queue. *Held: no
stage hit a rebase conflict.*

---

## Stages, in queue order

| # | Stage | Plan section | Slug | Size | Waits for | Why here |
|---|-------|--------------|------|------|-----------|----------|
| S0 | **LXQt spike** *(not queued — side task)* | lxqt § L0 | — | S | `lxqt-core` installed by the user (sudo) | Produces the § L0 verdict that S5 needs; runs on throwaway displays, so it can start on day one beside the queue |
| S1 | **Session starters + `rk gui wm`** | lxqt § L1 | `gui-session-starters-and-wm-verb` | S | — | Smallest backend change; lands the `guiOnSummary` `(session)` suffix and **owns the skill-page trim** (§ Coordination point 2) so S2 rebases over the smaller diff |
| S2 | **Fixed geometry + `rk gui resize`** | ergonomics § V1 | `gui-fixed-geometry-and-resize` | M | S1 merged | Amends D7; the one stage that changes a parent decision. Ships `gui.geometry` as a validated text field in Settings and defers the select control to S3 (§ Coordination point 1) |
| S3 | **Desktop picker** | lxqt § L3 | `gui-desktop-picker` | S | S1 merged | **Owns the "string with server-supplied choices" settings control** and the restart-confirm reuse; S4/S7 reuse it |
| S4 | **Zoom + touch pointer modes + key bar** | ergonomics § V2 | `gui-zoom-and-touch-pointer` | M | S2 merged | Frontend only; adopts the S3 control if the geometry row wants it, otherwise touches S3 only in the palette file |
| S5 | **LXQt seeded defaults** | lxqt § L2 | `gui-lxqt-seeded-defaults` | M | S1 merged **and** § L0 verdict appended to the lxqt plan | Backend only; placed here so S0 has four stages of wall-clock to finish. If the verdict is not in by the time S4 merges, the operator **pauses** before S5 rather than skipping it (§ Rules) |
| S6 | **Quality presets + stats overlay** | ergonomics § V3 | `gui-quality-presets-and-stats` | S | S4 merged | Shares the posture module and the toolbar seam with S4 |
| S7 | **Session toolbar + HiDPI + Send key** | ergonomics § V4 | `gui-toolbar-keybar-hidpi-sendkey` | M | S4 merged (S6 optional) | Last: pure polish; the pill mirrors rows every earlier stage created |

Dependency graph (arrows = "must be on main first"):

```
S0 (spike, side task) ─────────────────────────────┐
                                                   ▼
S1 ──▶ S2 ──▶ S4 ──▶ S6 ──▶ S7          S1 ──▶ S5 (needs S0 verdict)
 └───▶ S3 (after S1; before S4 so S4 can reuse its control)
```

Queue order honors every arrow: `S1, S2, S3, S4, S5, S6, S7`. *Ran in exactly this order.*

The LXQt plan's own § Execution order (L0 ∥ L1 first, then L2 ∥ L3, L2 gated on the L0 verdict) is satisfied by this queue: S1 = L1, S3 = L3, S5 = L2, with S0 = L0 running alongside. Its "L2 ∥ L3" parallelism collapses to S3-then-S5 because `merge-auto` is a single lane; nothing waits longer than one stage for it.

---

## Execution record

### Timeline

| # | Change | PR | Opened (Z) | Merged (Z) | Gap from previous merge | Size |
|---|--------|----|-----------|-----------|-------------------------|------|
| S0 | *(spike)* | commit `0b4be7f3` | — | 2026-09-10 (before S1 opened) | — | L0 verdict, ~150 lines |
| S1 | `260910-zsui-gui-session-starters-and-wm-verb` | #916 | 09-10 16:34 | 09-10 16:46 | — | +1473 / −105 |
| S2 | `260910-zuci-gui-fixed-geometry-and-resize` | #919 | 17:50 | 18:34 | 1 h 04 to open | +3089 / −185 |
| S2′ | *(follow-up, not queued)* Copilot review fixes for #919 | #920 | 18:38 | 18:45 | — | +41 / −11 |
| S3 | `260910-pp6o-gui-desktop-picker` | #929 | 20:12 | 20:44 | 1 h 27 to open | +2472 / −108 |
| S4 | `260910-0aur-gui-zoom-and-touch-pointer` | #931 | 23:17 | 23:25 | 2 h 33 to open | +3360 / −299 |
| S5 | `260910-pfe4-gui-lxqt-seeded-defaults` | #932 | 09-11 00:21 | 00:33 | 56 min to open | +1785 / −51 |
| S6 | `260910-5psw-gui-quality-presets-and-stats` | #933 | 01:16 | 01:50 | 43 min to open | +1404 / −68 |
| S7 | `260910-t2lv-gui-toolbar-keybar-hidpi-sendkey` | #934 | 03:17 | 04:20 | 1 h 27 to open | +2054 / −70 |

"Gap from previous merge" is the pipeline's whole per-stage cost under this
mode: spawn, intake gate, apply, review, hydrate, ship. Sizes are the PR's
squash diff. Total: 7 stages, ~15.6 k lines added, 11 h 46 min end to end,
S1 opening to S7 merging.

### Close-out already on main

- `eb6ddce1` — the seven intakes drafted (`/fab-draft`), IDs recorded here and in the child plans.
- `0b4be7f3` — § L0 verdict appended to the LXQt plan: L-D4 **confirmed** (LXQt 0.17 reads every seeded file from a prepended `XDG_CONFIG_DIRS` entry; no fallback needed); idle numbers recorded; teardown must signal the **process group** (bound into S1's supervisor kill).
- `66c6f05f` — S5's intake clarified with the verdict before spawn (the § Rules S5 gate, honored).
- `63540359` — S1–S7 change folders archived under `fab/changes/archive/2026/09/`.
- `f51bb9cf` — § Queue below filled; both child plans' § Status and breakdown rows set to Done.

### Done means — verified 2026-09-11

| Criterion | Result | Evidence |
|---|---|---|
| All seven PRs merged and squashed on main | ✓ | #916 #919 #929 #931 #932 #933 #934 all `MERGED`; #920 follow-up also merged |
| Both child plans' breakdown tables show Done with PR links | ✓ | lxqt rows L0–L3, ergonomics rows V1–V4 |
| `docs/specs/gui.md` § Resize policy reads the `auto`-value form of D7 | ✓ | section rewritten: `gui.geometry` default `1920x1080`, D7 survives as `auto`, pins inert under a fixed value (V-D4) |
| `docs/specs/gui.md` § Switching desktops exists | ✓ | present, plus § The tile from G2 |
| Parent plan's D7 row carries the pointer to the ergonomics plan | ✓ | `26-09-09-gui-surface.md` line 40 links `26-09-10-gui-viewer-ergonomics.md`; breakdown row `V1–V4` also present |
| Skill page ≤ 150 lines | ✓ | `docs/site/skill/gui.md` is 139 lines (was 145 before S1's trim; net −6 after adding `wm` and `resize`); `go test ./cmd/rk -run Skill` passes |
| `rk gui wm lxqt --restart` shows the seeded LXQt desktop at the geometry `rk gui resize` last set | ✓ (live host) | installed `rk v3.19.47` lists `wm` and `resize`; `rk gui status` → `on (Xtigervnc, :18, 1280x720 fixed, 0 viewers, startlxqt (session))`; `rk gui wm` → `startlxqt (pinned; running)` |
| The phone drives it in trackpad mode | **not verified here** | needs a hands-on phone; S4's Playwright coarse-pointer tests passed in the pipeline, which is the automated proxy |

---

## Deviations and lessons

1. **The arm-before-idle race (S2).** With `--stop-stage ship`, the operator armed #919's auto-merge as soon as the PR existed. The S2 agent then kept working: Copilot's review landed at 18:16 (inside the 10-minute poll for once), the agent fixed two findings and committed them — but #919 had already squash-merged at 18:34 without those commits on the branch. The fixes reached main as the separate follow-up PR #920 (cherry-picked, 18:45). **Rule for next time:** under `merge-auto`, arm only when the spawned agent's pane is idle *and* its `review-pr` stage is `done`, `failed`, or `active`-with-timeout (not `active`-and-working). Equivalently: keep `--stop-stage ship` for the *queue's* completion delta, but gate the *arm* on agent idleness, not on ship. This is a one-line addition to § Rules → Arming; it is written there now.
2. **Copilot mostly never came.** Only #919 received a Copilot review before merge; the other six PRs were merged 7–63 minutes after opening with zero reviews, and Copilot does not review merged PRs. So the planned post-queue "sweep with `/git-pr-review`" had nothing to process. Under `merge-auto` the Copilot review is effectively opt-out; if it is wanted, the arm must wait for it (which re-imports the 15–90 minute latency the `--stop-stage ship` rule was chosen to avoid). Decision for future queues is the user's: speed (as run) or a "wait for one review or 60 minutes" arm gate.
3. **The 30-minute stage flag is mis-sized for M changes.** Every M stage took 56 min to 2 h 33 from the previous merge to its PR opening (full lane: dispatched apply with the e2e phase, dispatched review, dispatched hydrate). The operator's default `> 30 min ⇒ flag` would have fired on five of seven stages. Suggest 90 minutes per stage for full-lane M changes in the operator's per-queue budget; total wall clock (11 h 46) was inside the "a working day" estimate once S0 ran alongside.
4. **Coordination points all held.** S3 built the settings choice control and S2 shipped `gui.geometry` as a text field (point 1); S1's skill-page trim left 11 lines of headroom after both verbs landed (point 2); no live-display collision was reported (point 3). Shared files rebased mechanically every time, as predicted.
5. **The S5 gate fired correctly.** S0's verdict landed before S4 merged, and S5's intake was clarified against it (`66c6f05f`) before spawn — the rule's happy path.

## Follow-ups

- **Stale remote branch** `origin/260910-vu4p-gui-desktop-tile-strip-and-palette` still exists (recreated by a post-merge status push during G2; its tip `b581ff67` is bookkeeping only). Safe to delete.
- **Phone trackpad acceptance** for S4 has not been done by hand on a real device; the coarse-pointer Playwright tests are the only evidence.
- **Copilot opt-in decision** (§ Deviations 2) for the next `merge-auto` queue.
- **Geometry row control**: S2 shipped `gui.geometry` as a text field; coordination point 1 left switching it to S3's choice control to "S7 or a later micro fix". S7 did not take it. Candidate micro fix if the text field grates.

---

## Before the queue starts (operator checklist — as run)

1. **User installs `lxqt-core`** on the host (`sudo apt install --no-install-recommends lxqt-core`, 38 packages here). rk never installs packages; S0, S1's acceptance, and S5 all need it present. *Done before S0.*
2. **Draft the seven intakes** with `/fab-draft` (create-without-activate), one per stage, each pointing at its plan section and following that plan's § Pickup protocol. Record the 4-char IDs in the § Change breakdown of the owning plan and in § Queue below. The intake gate (confidence ≥ 3.0) runs at spawn. *Done — `eb6ddce1`; no draft fell below the gate.*
3. **Spawn S0** as an enrolled ad-hoc agent (not autopilot) with the lxqt plan's § L0 instructions; its deliverable is a `docs:` commit to main appending § L0 verdict. Throwaway displays only, never the live `rk-gui` session. *Done — `0b4be7f3`, displays `:94 :96 :97 :98 :99`.*
4. **Start the queue**: `fab operator autopilot start --queue <S1,S2,S3,S4,S5,S6,S7> --mode merge-auto`. No `depends_on` on any entry. *Done; queue exhausted and stopped (`autopilot: null` in the operator state).*
5. **Enroll each spawn with `--stop-stage ship`** so Copilot's 15–90 minute latency never gates the queue. *Done — and the cause of § Deviations 1 and 2.*

## Queue

| # | Change ID | Folder | PR | Merged |
|---|-----------|--------|----|--------|
| S1 | zsui | 260910-zsui-gui-session-starters-and-wm-verb | #916 | ✓ 2026-09-10 16:46Z |
| S2 | zuci | 260910-zuci-gui-fixed-geometry-and-resize | #919 (+ #920) | ✓ 18:34Z (+ 18:45Z) |
| S3 | pp6o | 260910-pp6o-gui-desktop-picker | #929 | ✓ 20:44Z |
| S4 | 0aur | 260910-0aur-gui-zoom-and-touch-pointer | #931 | ✓ 23:25Z |
| S5 | pfe4 | 260910-pfe4-gui-lxqt-seeded-defaults | #932 | ✓ 2026-09-11 00:33Z |
| S6 | 5psw | 260910-5psw-gui-quality-presets-and-stats | #933 | ✓ 01:50Z |
| S7 | t2lv | 260910-t2lv-gui-toolbar-keybar-hidpi-sendkey | #934 | ✓ 04:20Z |

---

## Coordination points (decided here so no agent re-decides them)

1. **The settings-dialog choice control belongs to S3.** The registry's `enum` kind has static options only; S3's picker needs choices the server discovers (`wm_candidates` on the status document) and S2's geometry field wants presets plus a free `WxH`. S3 builds the "string with suggestions" control and the `Restart the desktop now?` confirm; **S2 ships `gui.geometry` as a validated text field** plus its palette rows and does not touch the dialog's control model. S7 (or a later micro fix) may switch the geometry row to S3's control. *Held; the switch is still open (§ Follow-ups).*
2. **The skill page has five lines of budget.** `docs/site/skill/gui.md` was at 145 of the 150-line cap the `skill_test.go` guard enforces. S1 adds `rk gui wm` and one gotcha; S2 adds `rk gui resize` and the note that a fixed desktop is what keeps `shot`/`click` coordinates stable. **S1 does the trim** as part of its docs task. *Held; the page ended at 139 lines.*
3. **The live display is a shared fixture.** Playwright real-rig tests are isolated per worktree (own tmux socket family, own state home), so they never collide. **Manual acceptance is not**: S1 restarts the host desktop into LXQt and back, S2 resizes it, S5 restarts into seeded LXQt. The operator serializes manual acceptance on the live `rk-gui` session with the queue order and never runs it while an S0 spike display is being measured. *Held; the live host now runs `startlxqt` at `1280x720 fixed`.*

Shared files, for the rebase step's awareness (all resolved mechanically because each stage started after its predecessor merged): `app/backend/cmd/rk/gui.go` (S1 and S2 each add a verb and a summary segment), `cmd/rk/gui_supervise.go` (S1, S2, S5 each add a log line or an env entry), `internal/gui/apps_linux.go` (S1 only), `api/gui.go` (S2 adds `resize`, S3 adds `wm_candidates`), `app/frontend/src/lib/palette/gui.ts` (S2, S3, S4, S6, S7 each add rows), `docs/specs/gui.md` (every stage, different sections), `docs/memory/run-kit/gui.md` and `ui/*` (hydrate, every stage).

---

## Rules for the operator on this queue

- **Skip is not the default for a base stage.** `merge-auto`'s failure policy skips a change on review exhaustion or a rebase conflict. S1 and S2 are bases for everything after them: if either is skipped, run `fab operator autopilot pause` immediately and escalate — do not let S3–S7 start on a main that lacks their base. S3, S5, S6, S7 may be skipped and re-queued individually; a skipped S4 also pauses (S6/S7 depend on it). *Never triggered.*
- **S5 gate.** Before spawning S5, check that `26-09-10-gui-lxqt-desktop.md` contains a `## L0 verdict` section on `origin/main`. Absent ⇒ `pause`, notify, and resume when it lands. *Triggered on the happy path.*
- **Arming.** Every autopilot PR is a draft (`/git-pr` creates drafts): `gh pr ready` then `gh pr merge --auto --squash`. One armed PR at a time (single repo, single sequence). Verify the merge on a later tick before `git fetch origin`, rebase, and the next spawn. Record the sequence in a `kind: coordination` note per § 6 Auto-Merge Choreography. **Added after the run (§ Deviations 1): arm only once the spawned agent's pane is idle and its `review-pr` stage is no longer actively working** — `ship: done` alone is the queue's completion delta, not the arm trigger.
- **Copilot.** A `review-pr` left `active` after `/fab-fff` is the expected Copilot-timeout outcome, not a stall; with `--stop-stage ship` it does not gate the queue. *Outcome: only one of seven PRs was reviewed before merge (§ Deviations 2); the post-queue sweep had nothing to process.*
- **Decision authority.** Each stage's intake treats its owning plan's § Decision log as Certain except the rows those plans mark Likely (lxqt L-D4 and L-D7's non-apt names; ergonomics V-D3's xrandr incantation and V-D6's zoom mechanism). Agents do not cross-edit the other plan's decisions; a conflict between the two plans is an escalation to the user, not a local fix. *L-D4 resolved by S0 to Certain; V-D3 and V-D6 resolved inside S2 and S4.*
- **Bookkeeping per stage.** Fill the § Change breakdown row in the owning plan when the change is created, mark Done when merged, and fill § Queue above. The final stage's PR updates both plans' § Status lines. *Done in `f51bb9cf`.*

## Timeouts and escalation

Per `fab-operator` § Autopilot: stage > 30 min ⇒ flag; total > 2 h ⇒ flag. Pre-run estimate: S2 and S4 at 45–60 min each, the rest 20–40 min, the whole queue a working day. *Actual: M stages 56 min to 2 h 33 each, S stages 43 min to 1 h 27, total 11 h 46 — see § Deviations 3 for the budget recommendation.*
