# HexoKit rebrand execution plan — run-kit becomes the product, `rk` stays the substrate

> Plan doc — written 2026-09-10 from the bright-pelican `/fab-discuss` thread.
> Cross-repo: this plan lives in run-kit because run-kit is the product being
> renamed, but its changes land in five repos (`run-kit`, `shll`, `shll.ai`,
> `homebrew-tap`, `fab-kit` + the satellite READMEs). Positioning authority:
> [`docs/wiki/competitive-landscape.md`](../../../docs/wiki/competitive-landscape.md)
> ("Cockpit for the agent era"). This doc owns the naming tiers, the decision
> log, the site shape, the change breakdown, and the cutover order.

**Goal**: make the product findable. "run-kit" is unsearchable — the first
page for `runkit` belongs to RunKit, the defunct YC-backed Node playground —
and shll.ai buries the dashboard as the 5th of "seven small CLIs" behind a
fab-kit-centred loop diagram. After this plan: the product is **HexoKit**
(hexokit.com, `brew install sahil87/tap/hexokit`, GitHub `sahil87/hexokit`),
the binary is still `rk`, every tmux/env/socket identifier is untouched, the
six companion CLIs keep their names and are presented as *the HexoKit
toolkit*, and shll.ai becomes a permanent redirect host.

**Status (2026-09-10)**: Phase 0 underway. Approach B chosen (product rename,
substrate kept). D1–D4 and D13 confirmed by Sahil in the thread; D5–D12 are
the plan's proposals and are open until marked confirmed. S1 done (operator,
direct git op, no fab change). S2 pipeline done, PR [hexokit-site#1](https://github.com/sahil87/hexokit-site/pull/1)
open draft, not yet merged (fab change `1ha7`, hexokit-site). Next pickup is
S3 ∥ S4 once S2 merges.

---

## Why this shape (the evidence)

- **The name is lost.** `runkit` search → RunKit (Node playground, "Try on
  RunKit" on npm, dead since ~2024, last commit 2023). `"run-kit" tmux agent`
  → zero hits for this project. `hexokit` → one 6-star abandoned 2020 repo
  (a Hexo blog-generator installer); GitHub org, brew formula, crates.io all
  free. Ownable in weeks.
- **The site contradicts the thesis.** shll.ai hero: "Seven small CLIs that
  force AI agents to plan before they code." Tool order in
  `src/lib/tool-slugs.ts`: idea, hop, fab-kit, wt, run-kit, tu, shll. The
  loop diagram centres fab-kit. Yet `shll`'s own roster (`tools.go`) already
  ranks run-kit first by importance, and the competitive-landscape doc is a
  single-product thesis. The market agrees: the adoption in this space goes
  to dashboards/multiplexers (guppi, herdr, cmux), not planning harnesses.
- **Coupling is to the substrate, not the name.** fab-kit references
  `@rk_pane_agent_state` 92×, `rk mux send` 39×, `rk mux await` 30×, all
  behind `exec.LookPath("rk")`. idea references run-kit 0×. Nothing in any
  repo depends on the *string* `run-kit` except brew, URLs, the config dir,
  and the README banner mandated by the `readme-extraction` standard.
- **The logo already says it.** `assets/logo.svg` is a cube inside a
  flat-top hexagon; shll.ai's logo is a verbatim copy of it. Six sides,
  six companions (fab-kit, wt, idea, tu, hop, desktop). The story is free.

---

## Decision log

| # | Decision | Status | Why |
|---|----------|--------|-----|
| D1 | **HexoKit names the dashboard product.** The family is "the HexoKit toolkit"; the companions keep their names (fab-kit, wt, idea, tu, hop, `shll`) | Confirmed | Approach B; umbrella-only (A) leaves the unsearchable product name in place; a second new name (D, "execute") teaches two names and is generic |
| D2 | **Binary stays `rk`.** `hexokit` becomes the long binary name, replacing `run-kit`, with `rk` the interchangeable alias — exactly today's `run-kit`/`rk` pair. Every `RK_*` env var, `@rk_*` tmux option (90 distinct), `rk-*` socket/session name (`rk-daemon`, `rk-gui`, `rk-test-e2e`, …), the Go module path `rk`, `rk-code-bridge` and its `rk.*` VS Code settings keys, and all `rk-*` CSS classes are **untouched** | Confirmed | ripgrep/`rg`, helix/`hx` precedent. Avoids a live-tmux-state migration that fab-kit also reads. Cross-repo contracts (`docs/specs/agent-state.md`, `cli-layering.md`) survive without a line changed |
| D3 | **Hexo association accepted.** "hexokit" reads as "Hexo kit" to Hexo users; exact-match SERP is empty so the term is ownable | Confirmed | — |
| D4 | **`shll` command stays** as the toolkit manager/installer name for now. Renaming it is out of scope (possible later phase) | Confirmed | Typed by few, near-zero brand exposure; its two network endpoints (`shll.ai/install`, `shll.ai/versions.json`) are baked into shipped binaries |
| D5 | **One website: hexokit.com.** shll.ai is folded, not kept beside it — two brands for one toolkit is the mistake being corrected. shll.ai stays registered forever as a 301 host, plus the two live endpoints in D4 | Proposed | — |
| D6 | **hexokit.com is a single-product site** in the guppi.sh / herdr.dev shape (hero → screenshot → install one-liner → features → toolkit → footer), *on top of* the existing Starlight docs layer, which moves under it unchanged. See § Site shape | Proposed | The landing page is the thing that was missing; the docs pipeline is the thing that works |
| D7 | **URL scheme.** HexoKit's own docs live at `/docs/…`; the companions keep the root-slug shape `/fab-kit/…`, `/wt/…` etc. that shll.ai already emits, so the sync pipeline and redirects are mechanical: `shll.ai/run-kit/* → hexokit.com/docs/*`, `shll.ai/<tool>/* → hexokit.com/<tool>/*`, `shll.ai/getting-started/* → hexokit.com/toolkit/*` | Proposed | Two redirect rules, zero pipeline changes for the six companions |
| D8 | **Electron**: `productName` "Run Kit" → "HexoKit", `artifactName` → `hexokit-desktop-…`; **`appId` `ai.shll.run-kit` is kept** so installed apps keep their identity/settings. Roster entry `rk-desktop` keeps its name (it's a CLI verb, `rk desktop`) | Proposed | appId is invisible to users; changing it makes the app a stranger to the OS |
| D9 | **On-disk homes migrate once, silently.** `~/.config/run-kit/` → `~/.config/hexokit/` (the `config-home` standard fixes the dir to the full tool name); `$XDG_STATE_HOME/run-kit/` → `…/hexokit/`; browser `runkit-*` localStorage keys → `hexokit-*`. Each is read-old-then-write-new on first run, old left in place for one release, then dropped | Proposed | Silent settings loss (theme, keybindings, macros, sidebar geometry) would be the one user-visible regression of the rename |
| D10 | **Install one-liner is product-first.** `curl -fsSL hexokit.com/install \| sh` installs `shll` + `hexokit` by default and prints the toolkit hint; the toolkit page offers the full install. Same script as today's `shll.ai/install` (it already takes tool args) with a changed default | Proposed | A product site whose install line pulls seven binaries reads as a bundle, not a product. Needs the `install-composition` standard's Policy B wording moved to hexokit.com |
| D11 | **Historical text is not renamed.** `fab/` change archives and plans, `docs/memory/` narrative, git history, old PR titles keep "run-kit". Only live surfaces (README, docs/site, specs' present-tense identity lines, code identifiers in the brand tier) change | Proposed | ~60 % of the 9,500 `run-kit` occurrences are archives; rewriting history is churn with no reader |
| D12 | **"kit" twice (HexoKit, fab-kit) is accepted.** If it ever grates, the fix is renaming fab-kit to `fab` (its binary already is), not touching HexoKit | Proposed | — |
| D13 | **Site repo is a copy, not a rename; shll.ai's repo is never renamed.** `git clone --mirror` of `sahil87/shll.ai` pushed to a new **`sahil87/hexokit-site`** (full history, all branches/tags). hexokit.com is built and published there while shll.ai stays live and untouched. At cutover, the `shll.ai` repo's *contents* are replaced in place with the redirect stub (CNAME `shll.ai`, redirect pages, byte copies of `/install` + `/versions.json`) — no new repo, no rename, GitHub redirects and history intact. No `hexokit.com`-named repo is ever created (avoids the `shll`/`shll.ai` twin-name confusion). Site is not folded into the product repo: its daily sync crons commit to `main` and would pollute the product's history and CI | Confirmed | Sahil's ordering: nothing irreversible happens until the site has been tested end to end on the real domain. One repo carries one Pages deployment + one custom domain, so two live domains need two repos regardless. Lost in the copy: only the site repo's issues/PRs/stars (near zero) |

---

## Naming tiers

| Tier | Identifiers | Action |
|------|-------------|--------|
| **Brand** (user reads it) | GitHub repo `sahil87/run-kit`; tap formula `run-kit`; long binary `run-kit`; README H1/tagline/banner; `docs/site/*` (10 pages); shll roster `Name`/`Formula`/`Repo`; shll.ai slug tables + sidebar; Electron `productName`/`artifactName`; `~/.config/run-kit/`; `$XDG_STATE_HOME/run-kit/`; `runkit-*` localStorage keys; `run-kit-frontend`/`run-kit-desktop` private npm names; `CamelCase RunKit` in user-visible strings; sahil87 profile README; homebrew-tap README (still points at stale `ai.shll.in`); Discord/OG/JSON-LD on the site | **Rename** |
| **Substrate** (machines read it) | `rk` binary; ~90 cobra verbs; `RK_*` (43 vars); `@rk_*` (90 options); `rk-daemon`/`rk-gui`/`rk-ctl`/`rk-jobs`/`rk-test-e2e-*` sockets & sessions; Go module `rk`; `rk-code-bridge` + `rk.*` settings; `rk-*` CSS; `rk agent setup` hook names; release asset names `rk-<os>-<arch>` | **Keep** |
| **Historical** | `fab/changes`, `fab/plans`, `docs/memory` prose, PR/commit history | **Leave** (D11) |

Rough live-surface size: 30–50 files across the five repos plus the three
standards documents. The 9,500-occurrence grep count is not the work.

---

## Site shape (hexokit.com)

Answering the open question from the thread: **yes, the homepage is a
star-tool page like guppi/herdr, and no, the companion docs do not go away —
they move one level down, pipeline intact.**

Three layers, one Astro site (the existing `sites/astro-starlight-terminal1`
fork, or a sibling variant per shll.ai's multi-site constitution):

1. **`/` — product landing.** Custom page, not the Starlight docs template.
   Carries run-kit's own visual vocabulary (monospace, hexagon mark, accent
   green, the CRT/typed-cursor treatments already in `globals.css`) so the
   site and the app look like one thing. Sections, in order:
   - Hero. Working tagline is already in the README: **"Your tmux, in the
     browser and on your phone."** Sub-line from the landscape doc: *Cockpit
     for the agent era.* Desktop + phone screenshot side by side (shll.ai
     already curates `public/screenshots/run-kit-*.webp`).
   - Install one-liner (D10) + `brew install sahil87/tap/hexokit`.
   - Features, five or six cards, each with a real screenshot: every pane a
     live terminal from any device · agents are just panes (state via hooks,
     agent-agnostic, "the agent is one of the things you run") · `rk riff`
     — one agent per worktree, watch the fleet · boards + status pyramid ·
     cron clock + operator · code / web / GUI tiles.
   - "Nothing wraps your agent" — the agent-agnostic paragraph from the
     README, verbatim-ish. This is the differentiator vs herdr/cmux.
   - **The toolkit hexagon.** The cube-in-hexagon mark with the six
     companions on the six edges — fab-kit, wt, idea, tu, hop, desktop —
     one line each, linking into `/toolkit/` and each tool's subtree. This
     is where the current shll.ai loop diagram's content survives, reframed
     around the product.
   - Desktop app card (today the app has no page anywhere).
   - Footer: Docs · Toolkit · GitHub · Discord · versions.json · llms.txt.
2. **`/docs/…` — HexoKit docs.** Starlight. Synced from run-kit's
   `docs/site/` exactly as today (install, tutorial, boards, notifications,
   agent-hooks, status-dot, customizing-tmux, cron-schedule-kinds, skill +
   topics) plus the help-dump command reference. First-class nav item.
3. **`/toolkit/` + `/<tool>/…` — the family.** `/toolkit/` absorbs today's
   Getting started (overview, philosophy, daily flow, "start a new change")
   rewritten from "seven CLIs" to "HexoKit and the six tools around it".
   Each companion keeps its synced Overview / Readme / Commands / docs-site
   pages at the root slug (D7). The daily README/help-dump crons keep
   running with one slug change (`run-kit` → `hexokit`, output to `/docs/`).

What is deliberately *not* on the homepage: the seven-tool table, the
install-everything-first flow, fab-kit's pipeline diagram (it lives at
`/fab-kit/` and on `/toolkit/`).

Navigation: `Docs · Toolkit · Desktop · GitHub`. shll.ai's `/llms.txt`,
`/llms-full.txt`, `/versions.json`, `/.well-known/security.txt` move as is.

---

## Change breakdown

Three phases. **Phase 0 builds the new site beside the old one with nothing
irreversible; Phase 1 does the renames; Phase 2 is the cutover flip.** One
row = one PR (S1 is a one-off git operation, not a PR). Rows in the same
block can run in parallel. Agents: fill folder/PR when you create the change.

### Phase 0 — build hexokit.com beside shll.ai (reversible; shll.ai untouched)

| # | Repo | Slug (suggested) | Depends on | Size | Scope | PR | Status |
|---|------|------------------|-----------|------|-------|----|--------|
| S1 | hexokit-site (new) | *(git op, no fab change)* | — | S | `git clone --mirror sahil87/shll.ai` → push to new `sahil87/hexokit-site` (create with `gh auth switch --user sahil87`). Disable the copied cron workflows until S3 lands so they don't scaffold a second run-kit tree | — | **done** — repo created private, 40 branches mirrored (PR hidden-refs rejected as expected), `Refresh: Help`/`Refresh: README` workflows disabled |
| S2 | hexokit-site | `hexokit-domain-and-deploy` | S1 | S | CNAME → `hexokit.com`; Namecheap DNS → Pages (A/AAAA + `www` CNAME); verify the domain on the `sahil87` account; `site:` in astro.config; deploy workflow green on the new repo. Site is live but unannounced | [hexokit-site#1](https://github.com/sahil87/hexokit-site/pull/1) (draft) | **pipeline done, PR open** — repo flipped public (free plan has no private Pages), custom domain `hexokit.com` set on Cloudflare (not Namecheap — DNS was already on Cloudflare), `hexokit.dev` 301s to `hexokit.com` via Cloudflare redirect, `site:`/CNAME/robots.txt flipped, first deploy green. Not yet merged. Remaining: HTTPS enforcement once GitHub issues the cert (`gh api -X PUT repos/sahil87/hexokit-site/pages -F https_enforced=true` as sahil87), account-level domain verification (GitHub UI TXT record), then merge #1 |
| S3 | hexokit-site | `hexokit-site-structure` | S2 | M | D7 URL scheme: slug table gets `hexokit` → output `/docs/`, **source repo still `sahil87/run-kit`** (one field flips in X2); companions unchanged at root slugs; `/toolkit/` from getting-started; re-enable both crons with the new map; nav `Docs · Toolkit · Desktop · GitHub`; llms.txt, JSON-LD, OG, favicon regen | | not started |
| S4 | hexokit-site | `hexokit-landing` | S2 | L | D6 landing page + design iteration (§ Site shape). Hand-written, so it is on-brand from day one even while `/docs/` still reads "run-kit" | | not started |
| S5 | hexokit-site | `hexokit-install-script` | S3 | S | D10: `/install` served from the same script with the product-first default; the `install-composition` Policy B text prepared for C1 | | not started |

### Phase 1 — renames (standards first; shll.ai still live and unchanged)

| # | Repo | Slug (suggested) | Depends on | Size | Scope | PR | Status |
|---|------|------------------|-----------|------|-------|----|--------|
| C1 | shll | `hexokit-standards-and-roster` | S4 accepted | M | Standards: `readme-extraction` banner → "Part of [HexoKit](https://hexokit.com)"; `install-composition` Policy B → hexokit.com; `config-home` example → `hexokit`; `update` naming clause. Roster: `run-kit` → `hexokit` (`Formula`, `Repo`, `Name`), `LegacyName` gains `run-kit` beside `rk`; `versions.json` URL constant → hexokit.com (shll.ai kept as fallback); `shll skill` bundle text | | not started |
| C2 | homebrew-tap | `hexokit-formula` | C1 | S | `Formula/hexokit.rb` installs `hexokit` + `rk` symlink; `formula_renames.json` adds `run-kit → hexokit` (precedent: `rk → run-kit`); README banner + drop stale `ai.shll.in` | | not started |
| C3 | run-kit | `hexokit-repo-rename` | C1 | L | README H1/tagline/badges; `docs/site/*` identity lines; `.github/formula-template.rb`, `scripts/release.sh` asset/formula names; Electron `productName`/`artifactName` (D8, appId kept); `run-kit` cobra alias verb → `hexokit` (keep `run-kit` hidden alias one release); specs' present-tense identity lines; `fab/project/config.yaml` name; `docs/site/skill.md` H1. **Last step: GitHub rename `run-kit` → `hexokit`** (old URL redirects; needs `gh auth switch --user sahil87`) | | not started |
| C4 | run-kit | `hexokit-home-migration` | C3 | M | D9: `~/.config/run-kit` → `~/.config/hexokit` (one-time move, dual-read one release); `$XDG_STATE_HOME/run-kit` → `hexokit` (cron entries + snapshots must move; droppable caches may cold-start); `runkit-*` localStorage → `hexokit-*` read-old/write-new | | not started |
| C7 | fab-kit + wt, idea, tu, hop, sahil87 | `hexokit-banner-sweep` | C1 | S each | README banner via the standard; present-tense "run-kit" product mentions → HexoKit (`rk` verbs untouched); sahil87 profile table row + fix `brew install sahil87/tap/all` (no such formula) | | not started |

### Phase 2 — cutover (the only irreversible steps; do in one sitting)

| # | Repo | Slug (suggested) | Depends on | Size | Scope | PR | Status |
|---|------|------------------|-----------|------|-------|----|--------|
| X1 | hexokit-site | `hexokit-site-source-flip` | C3 merged | S | Slug table source `sahil87/run-kit` → `sahil87/hexokit`; refresh crons run once; redirect map for the old shll.ai paths ready (D7) | | not started |
| X2 | shll.ai | `shll-ai-redirect-stub` | X1 live | S | Replace the repo's contents in place (D13): CNAME `shll.ai`, redirect pages → hexokit.com, **byte copies** of `/install` and `/versions.json`. They MUST be real files, not redirects: GitHub Pages redirects are meta-refresh HTML, and `curl -fsSL … \| sh` would feed that HTML to `sh` (curl's `-L` only helps against real 301s, which Pages cannot emit). Refreshed by the same CI copy step. Remove the cron workflows. Never lapses (D4) | | not started |
| X3 | run-kit | `hexokit-memory-hydrate` | X2 | S | Memory + specs identity sweep for present-truth lines only (D11); competitive-landscape one-liner; `context.md`; this plan's Status → Done | | not started |

Order: S1 → S2 → (S3 ∥ S4) → S5 → *[design accepted]* → C1 → (C2 ∥ C3 ∥ C7)
→ C4 → X1 → X2 → X3. Users on the old formula get brew's rename handling
from C2; users on old `shll` binaries keep working via X2's endpoints.

## Constitution mapping (run-kit)

- **II No Database** — C4 moves files under `$XDG_STATE_HOME`; every moved
  class is either a droppable cache (may cold-start) or the two bounded
  carve-outs (snapshots, seed caches). Nothing becomes authoritative.
- **IV Minimal Surface Area** — no new routes, no settings surface; the
  config key set is unchanged, only its directory (registry-driven).
- **VII Convention Over Configuration** — the config dir still derives from
  the tool name; `hexokit` is now the tool name.
- **Toolkit Standards** — this plan *changes* three standards (C1) before
  any repo edit; every later row is checked against the revised text.
- **X Hooks Carry Only the Underivable** — untouched; hook names are
  substrate tier.

---

## Risks

- **Silent state loss on rename** (D9). Mitigation: dual-read one release,
  e2e test that seeds old keys/dirs and asserts they are picked up.
- **Formula rename edge cases.** `brew upgrade` with `formula_renames.json`
  has worked once (`rk → run-kit`); verify on a clean box before C2 merges.
- **Electron identity.** Keeping `appId` while changing `productName` is
  supported by electron-builder but macOS may show the old name in some
  system dialogs until re-launch; acceptable.
- **Hexo confusion** (D3). Accepted; mitigate with a one-line "not related
  to the Hexo blog generator" in the GitHub repo description if it comes up.
- **fab-kit skill prose drift.** fab-kit's `_cli-agents.md`/`fab-operator.md`
  mention "run-kit" as the product ~30×; substrate verbs (`rk mux …`) are
  correct and stay. C7 needs a careful present-tense-only pass.
- **Two-site window.** During Phase 0/1 both sites are live: shll.ai
  unchanged, hexokit.com unannounced. Nobody is pointed at hexokit.com until
  X1, so the window is safe; its docs layer reads "run-kit" until then.
- **Cron double-scaffold.** hexokit-site's copied cron workflows must stay
  disabled until S3's slug map lands, or they commit a second `run-kit` tree.

---

## Pickup protocol (for the agent taking the next change)

1. Read this doc's Decision log; D1–D4 and D13 are Certain, D5–D12 become Certain
   when Sahil marks them confirmed here — do not re-open either set.
2. Phase 0 lives in the new `hexokit-site` repo (S1 creates it). Phase 1
   starts in `shll` (`hop where shll`) and is standards-first by design:
   run `shll standards` and read `readme-extraction`, `install-composition`,
   `config-home`, `update` before editing. Do not start Phase 1 until Sahil
   has accepted the S4 landing page.
3. Substrate identifiers (`rk`, `RK_*`, `@rk_*`, `rk-*`) are never renamed
   under this plan. If a change appears to need it, stop and add a row here.
4. Update the Status line and your row when you create/merge a change.
