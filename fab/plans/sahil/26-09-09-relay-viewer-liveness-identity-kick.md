# Relay Viewers — Liveness Deadline, Derived Identity, Kick

**Drafted**: 2026-09-09 · against `e5d799d8` · triggered by ghost attach clients (phone/iPad tabs) clamping tmux window width under `window-size smallest`, reconnecting no matter what
**Shape**: 3 phases, one repo (run-kit). **Execute Phase 1 and Phase 2. Phase 3 is optional — build it only if ghosts persist with 1+2 live.**
**Rule of record**: `docs/memory/run-kit/tmux-sessions.md` § Attached-Client Enumeration + § Terminal Relay · `docs/memory/run-kit/ui/terminal.md` § Hidden-page stream suspension · `docs/memory/run-kit/configuration.md` § Multi-viewer sizing guard

## Diagnosis

Every browser terminal tile is one real sized `tmux attach-session` client forked by the daemon on a PTY (`api/terminals_ws.go` `attachStream`, `pty.StartWithSize`). tmux sizes a window to the narrowest sized client currently viewing it (`window-size smallest` + `aggressive-resize on`) — a deliberate guard for the tmux ≤3.7c pane-status redraw wedge, **not** negotiable. So a stale attach client with a phone-sized grid clamps every co-viewer of that window until the attach dies.

Why they linger (verified at `e5d799d8`):

- **The relay heartbeat is one-directional.** The client sends `{op:"ping"}` every 30 s (`relay-mux.ts` `HEARTBEAT_INTERVAL_MS`) and gives up on a silent server after 60 s (`LIVENESS_TIMEOUT_MS`). The server pongs (`terminals_ws.go:271`) but sets **no read deadline** outside teardown (`:645`, `terminalsCleanupWait` only). A socket whose peer stops talking lives until the kernel or a reverse proxy tears down TCP — behind a proxy/tunnel the daemon's TCP peer is the proxy, which stays healthy, so the ghost lives as long as the proxy's idle timeout.
- **Hidden-page suspension (260903-xj0w) needs JS to run.** A phone/iPad tab that the OS freezes before the 60 s grace fires never sends its `close` ops; the attach clients survive as a frozen tab's sockets.
- **"They reconnect no matter what" is by design.** The mux backoff-reconnects (`RECONNECT_BASE_MS`→`RECONNECT_CAP_MS`) and re-opens every live stream in `ws.onopen`; a stream-level `closed` with a non-4004 code makes `terminal-client.tsx` probe ONE fresh re-open (`:1137-1175`). A bare `tmux detach-client` against a browser client is therefore undone within seconds — which is exactly the reported symptom.
- **Nothing identifies a client.** `ClientInfo` (`internal/tmux/tmux.go:1142`) carries tty/grid/session/flags; `Viewer` (`internal/sessions/sessions.go:24`) is `{width,height}`; the relay stores nothing about the browser per connection (`terminalsConn`, `:183`) and nothing per stream (`stream`, `:169`) beyond the `*exec.Cmd`. The viewer chip and card line (`session-row.tsx:162-190`, `:412-425`) render grids only.

### Facts that shape the plan

- `list-clients` on the user's tmux (3.7c) exposes `client_pid`, `client_created`, `client_activity`, `client_termname` (verified live). `client_pid` is the attach process — the pid of the `*exec.Cmd` the daemon forked, so an rk-owned attach joins onto its stream with no new tmux state.
- `resumeSuspended()` calls `connect()` (`relay-mux.ts:379-395`), so a socket the server closed while every stream was suspended (heartbeat stopped — it is stream-gated) reconnects on the `visible` transition. Phase 1's deadline can close idle-suspended sockets without stranding resume.
- Constitution §II/§X: identity must be **derived** (peer address, User-Agent, pid, times the daemon already has), never self-reported. An in-memory map keyed by attach pid describes processes the daemon itself owns and dies with them — a fact about live children, not a state store; write that sentence into the design decision.
- Constitution §IV/§V: the viewer chip stays display-only; actions live in the session card + the command palette.
- Constitution §IX: any kick endpoint is `POST`.

---

## Phase 1 — Server-side liveness deadline (SMALL, ships first, closes the root cause)

### `api/terminals_ws.go`
- Add `terminalsLivenessTimeout = 90 * time.Second` (3 missed client heartbeats; longer than the client's own 60 s give-up so a live client always disconnects itself first).
- After `Upgrade` set `conn.SetReadDeadline(now + terminalsLivenessTimeout)`; refresh it after every successful `ReadMessage()` in the read loop (`:243`). Any inbound frame — data, control, ping — counts as liveness. Expiry surfaces as a read error → existing `teardown()` path → every stream's attach client + PTY reaped (`killAndReapAttach`).
- Keep the existing `terminalsCleanupWait` deadline in teardown (shorter, unchanged).
- `slog.Info` on deadline expiry with stream count + peer address (the first place the daemon names a ghost).

### `lib/relay-mux.ts`
- No protocol change. Verify (unit) that a server close with zero live streams stays closed (`scheduleReconnect` zero-check) and that `resumeSuspended` reconnects onto it. Export the server timeout as a documented invariant next to `HEARTBEAT_INTERVAL_MS` (comment only) so the two constants cannot drift silently.

### Tests
- `terminals_ws_test.go`: a connection that opens a stream then sends nothing is torn down at the deadline (clock-injected or a short test override of the constant) and its attach `cmd` is reaped; a connection that pings keeps its stream indefinitely past the deadline; refresh happens on data frames, not only pings.
- `relay-mux.test.ts`: suspended-all → server closes socket → `visible` resumes via `connect()` (guards the interaction above).

### Manual verification
- Open a tile on a phone, put the phone to sleep. `tmux -L <srv> list-clients -F '#{client_pid} #{client_width}x#{client_height} #{t:client_activity}'` shows the phone client gone within ~90 s; the desktop tile's width recovers on the next redraw. Wake the phone: the tile reconnects and repaints.

### Memory / docs
- `tmux-sessions.md` § Terminal Relay + `architecture.md` § Client heartbeat: heartbeat is now bidirectional-by-deadline; record the 30 s / 60 s / 90 s ladder and why 90.

---

## Phase 2 — Derived viewer identity (joins relay facts onto `list-clients`)

### `api/terminals_ws.go` — record what the daemon already knows
- On upgrade, capture onto `terminalsConn`: `peer` (`X-Forwarded-For` first hop when present, else `r.RemoteAddr` host), `userAgent`, `connectedAt`, `lastInbound` (updated where Phase 1 refreshes the deadline).
- On `attachStream` success, register `cmd.Process.Pid → *attachMeta{conn *terminalsConn, streamID, server, windowID, openedAt}` in a `Server`-level `attachRegistry` (mutex-guarded map). Unregister in `stream.teardown()`. Expose `Lookup(pid) (AttachMeta, bool)` and a read-only snapshot for tests.

### `internal/tmux/tmux.go` — widen the client format
- `clientFormat` gains `client_pid`, `client_created`, `client_activity`, `client_termname` (11 fields). `ClientInfo` gains `PID int`, `Created`, `Activity time.Time`, `TermName string`. `parseClients` tolerates the extra fields; the two exclusion classes are unchanged.

### `internal/sessions/sessions.go` — enrich `Viewer`
- `Viewer` gains `omitempty` fields: `kind` (`"rk"` when the pid is in the attach registry, `"tty"` otherwise — an ssh/local `tmux attach` stays visible with no browser identity), `pid`, `device` (coarse UA class: `phone`/`tablet`/`desktop`/`desktop-shell`/`unknown`, derived once at upgrade — no UA string on the wire), `peer`, `ageSec` (since `client_created`), `idleSec` (since the conn's `lastInbound` for `rk`, `client_activity` for `tty`). `foldViewers` takes a `func(pid) (AttachMeta, bool)` resolver so `internal/sessions` does not import `api` (the existing `api → sessions` direction holds).
- Same marshal, same `/api/sessions` + state-socket `sessions` event — no new endpoint, no poll. Old-frontend readers ignore the extra keys; old-backend payloads still render (all new fields optional in the TS type).

### Frontend — `session-row.tsx`, `api/client.ts`
- Card viewers line becomes one row per viewer: `<device glyph> <W>×<H> · <peer> · <age> · idle <n>s` (`data-testid="row-flyout-viewer"` per row; the existing `row-flyout-viewers` container stays). The clamping client (narrowest width) gets a `narrowest` marker so the culprit is legible. Chip unchanged (display-only, ≥2 gate).
- Mobile: rows are display-only text; no new tappable floor needed.

### Tests
- `parseClients` 11-field fixtures incl. missing-`client_pid` (older tmux → 0 → `tty` kind); registry register/unregister lifecycle; `foldViewers` join + `narrowest` selection; `session-row.test.tsx` per-viewer rows + old-payload fallback.

### Manual verification
- Two devices on one window: the card names the phone row as `phone · 100.x.x.x · narrowest`. An ssh `tmux attach` appears as `tty`.

### Memory / docs
- `tmux-sessions.md` § Attached-Client Enumeration (new fields, the pid join, the registry-is-not-a-store decision); `api-and-sockets.md` `/api/sessions` row; `ui/sidebar.md` session card viewers rows.

---

## Phase 3 — Kick (OPTIONAL — only if ghosts survive Phases 1+2)

Build only when a stale viewer is still observed after Phase 1 is live and Phase 2 has named it. If it is built, the no-reopen close code is mandatory — without it a kick is undone in seconds.

### Backend
- `POST /api/sessions/{session}/viewers/{pid}/detach` (`?server=`): for an `rk` viewer, `closeStream(id, closeKicked, "kicked by <peer>")` on its conn with new `closeKicked = 4005`, then reap; for a `tty` viewer, `tmux detach-client -t <client_tty>`. Validate `{session}` and `pid` (pid must be in the registry or in `list-clients` for that session). Wake the state hub on success.

### Frontend
- `relay-mux.ts`: a `closed` 4005 retires the stream and marks it `kicked`; `terminal-client.tsx` does **not** probe re-open on 4005 — it renders a static "Detached by another viewer — tap to reconnect" overlay; reconnect happens on the overlay tap, a manual refresh, or the next `visible` transition (the user's stated expectation: come back to the device, refresh, it reconnects).
- Session card: a `Detach` action per `rk`/`tty` viewer row (hidden on the viewer's own connection — the card knows its own conn via a per-socket id echoed in the `sessions` payload, else hide the row for `narrowest === self`). Palette: `Session: Detach other viewers` (all-but-self) per Constitution §V.

### Tests
- `terminals_ws_test.go`: detach endpoint emits 4005 and reaps; unknown pid → 404; pid of another session → 400. `relay-mux.test.ts`: 4005 retires without re-open; `visible` after a kick reconnects. `terminal-client.test.tsx`: overlay + tap reconnect. e2e `sidebar-*.spec.ts`: two contexts, kick one, its attach disappears from `list-clients`, it does not return until reload.

### Memory / docs
- `api-and-sockets.md` new route; `ui/terminal.md` 4005 semantics beside 4004/4001/1000; `ui/sidebar.md` card action row.

---

## Sequencing & risk

1. Phase 1 is independent and small — ship alone, observe for a day before Phase 2.
2. Phase 2 is additive payload + one widened format; the only cross-package seam is the pid resolver injected into `foldViewers`.
3. Phase 3 depends on Phase 2's pid identity and adds a route + a new close code; deferred by default.

**Risks**: (a) a legitimate visible tab whose main thread is starved >90 s gets dropped and transparently reconnects (accepted — the mux already handles socket drops flicker-free); (b) `X-Forwarded-For` is trusted only for display, never for authorization — note in code; (c) `client_pid` absent on very old tmux → every viewer is `tty`, identity degrades, nothing breaks.
