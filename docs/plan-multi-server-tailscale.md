# Plan: Multi-Server Support via Tailscale

## Goal

Enable multiple Claude Code instances running on different remote servers to send
notifications to a single cc-g2 Hub, which relays them to Even G2 glasses. All
devices are connected via Tailscale.

## Target Architecture

```
Server A (Hub host)              Server B (remote)          Server C (remote)
├─ Hub:8787  <──── Tailscale ──── Claude Code ──────────── Claude Code
├─ Vite:5173                      (hooks → Hub's TS IP)    (hooks → Hub's TS IP)
├─ Claude Code (local, optional)
├─ tmux sessions
│
└─── iPhone + G2 glasses (via Tailscale)
```

## What Changed

### Phase 1: Configurable HUB_URL [DONE]

Replaced all hardcoded `127.0.0.1` references with `CC_G2_HUB_URL` env var.

Files modified:
- `scripts/cc-g2.sh` — reads `CC_G2_HUB_URL`, uses it for health checks, auth
  checks, and hook URL injection. Passes it to tmux sessions as env var.
- `scripts/cc-g2-stop-notify.sh` — uses `CC_G2_HUB_URL` for Hub URL
- `scripts/cc-g2-statusline.sh` — uses `CC_G2_HUB_URL` for Hub URL
- `.env.example` — documents `CC_G2_HUB_URL`

Default: `http://127.0.0.1:8787` (backward compatible).

### Phase 2: Remote session identification [DONE]

Added `CC_G2_HOSTNAME` (defaults to `hostname -s`) throughout the pipeline:

- `scripts/cc-g2.sh` — resolves hostname, sends as `X-CC-G2-Hostname` header in
  PermissionRequest hook, passes to tmux sessions as env var
- `server/notification-hub/index.mjs` — reads header, stores in notification metadata
- `scripts/cc-g2-stop-notify.sh` — includes hostname in stop notification metadata
- `src/glasses-ui.ts` — prepends hostname to notification prefix
  (e.g., `server-b:myproject#1:`)
- `.env.example` — documents `CC_G2_HOSTNAME`

### Phase 3: Remote reply-relay [DONE — Option A]

Key finding: PermissionRequest approvals already work fully for remote sessions.
The Hub code (`shouldRelay = false`) skips the tmux relay for resolved approvals;
the HTTP long-poll response is the only channel. No changes needed for approvals.

Changes for non-approval replies (stop notification comments):
- `server/notification-hub/reply-relay.sh` — extracts `hostname` from notification
  metadata, compares with local hostname, gracefully skips relay for remote hosts
  (exit 0 + log) instead of failing with exit 1

Limitation: Voice comments on stop notifications cannot be relayed to remote
sessions. Revisit with Option C (pull-based agent) if needed.

### Phase 4: Documentation and testing

- [ ] Update README with multi-server setup instructions
- [ ] Add Tailscale network configuration examples
- [ ] Test with 2+ remote servers

## Remote Server Quick Setup

On a remote server with Tailscale:

```bash
git clone https://github.com/inutano/cc-g2.git
cd cc-g2
export CC_G2_HUB_URL=http://<hub-tailscale-ip>:8787
export CC_G2_HOSTNAME=server-b        # optional, defaults to hostname -s
export HUB_AUTH_TOKEN=<token-from-hub> # from tmp/notification-hub/hub-auth-token
cc-g2
```

## Future Considerations

- Alternative STT provider (replace Groq with OpenAI Whisper API, local Whisper,
  or other OpenAI-compatible endpoint) — revisit after multi-server is stable
- Pull-based relay agent (Option C) for voice comment relay to remote sessions

## Non-Goals

- Public internet deployment (Tailscale only)
- WebSocket/SSE push (polling is fine for now)
- Changes to the G2 UI or Even Hub SDK
