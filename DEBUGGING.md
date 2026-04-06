# Claude Peers — Long-Running Session Bug

Tracking the investigation and fixes for sessions that lose peer registration after running for hours/days.

## The Problem

Long-running Claude Code sessions (the atlas main session runs 24/7) periodically become invisible to other peers. Cron jobs that need to relay results via `send_message` can't find the main session, causing missed morning briefs, job hunt results, and evening wraps.

Symptoms:
- Cron session calls `list_peers` and gets zero results (or doesn't find the main session)
- Main session's MCP server is still running but broker has stale/missing registration
- Cron falls back to writing results to `crons/results/` files (which nobody reads)
- Rob doesn't get Telegram alerts because the cron can't reach the orchestrator

## Timeline of Investigation

### 2026-03-31: First noticed
- Job hunt cron ran but Rob didn't see results
- Morning brief cron couldn't find the main session
- Workaround: restart the main session to re-register

### 2026-04-01: Peers relay worked after restart
- After restart, cron found the session and delivered results successfully
- Re-engage and discover agents both relayed via peers correctly
- Confirmed: fresh sessions work, old sessions don't

### 2026-04-02: Morning brief missed again
- Cron log showed: "Results written to crons/results/ for heartbeat pickup (main session not running)"
- But the main session WAS running — just not discoverable via peers
- Main session had been running since previous day's restart

### 2026-04-03: Session restart fixed it temporarily
- Rob requested restart at ~7:50 AM to refresh peers
- After restart, peers test worked: test session found main session and sent message successfully
- But by next morning, same issue recurred

### 2026-04-04: Peers MCP server disconnected mid-session
- System notification: "claude-peers MCP server disconnected"
- Broker was still running (PID alive) but the per-session MCP server connection dropped
- Investigated broker DB directly: `sqlite3 ~/.claude-peers.db "SELECT * FROM peers"` showed stale entries
- Rob made code changes to fix the root issue, restarted, peers worked again
- Test session confirmed: spawned a test session, it found the main session, sent a message successfully

### 2026-04-04: Rob's initial code fix
- Rob made changes to the peers code (details in commit history)
- After restart with new code, peers worked for the rest of the day
- Cron job hunt results relayed successfully via peers

### 2026-04-05: Morning brief lost again
- Brief ran at 6:05 AM, sent to peer `wlmnf55e` (old session ID)
- Session was restarted at 6:49 AM (Rob requested), got new peer ID `6atdd0g5`
- The brief was delivered to a session that was killed 44 minutes later
- This is a timing issue, not the long-running bug — but highlights fragility

### 2026-04-05: Root cause analysis completed
- Explored agent analyzed server.ts and broker.ts in detail
- Identified 3 bugs causing the registration loss
- Swarm session dispatched to fix all 3

## Root Cause Analysis

### Bug 1 (CRITICAL): Silent heartbeat failure without re-registration

**File:** `server.ts` ~line 536-552

When the heartbeat POST to the broker fails (broker restart, network hiccup, process issue), the catch block silently ignores the error:

```typescript
const heartbeatTimer = setInterval(async () => {
  if (myId) {
    try {
      await brokerFetch("/heartbeat", { id: myId });
    } catch {
      // Non-critical  <-- BUG: error silently ignored, no re-registration
    }
  }
}, HEARTBEAT_INTERVAL_MS);  // 15 seconds
```

The server keeps using a stale `myId` forever. All subsequent peer operations fail silently. In a 24-hour session, at least one broker restart is likely (the broker runs as a KeepAlive LaunchAgent).

**Fix (2026-04-05):** Added `needsReregistration` flag. On heartbeat failure, logs the error and flags for re-registration. On next tick, calls `ensureBroker()` + `/register` to get a new peer ID. Backs off — one attempt per interval.

### Bug 2 (MAJOR): No timeout-based stale peer cleanup

**File:** `broker.ts` ~line 88-120

`cleanStalePeers()` only checks if the PID exists via `process.kill(pid, 0)`, never checks the `last_seen` timestamp:

```typescript
function cleanStalePeers() {
  const peers = db.query("SELECT id, pid, ppid FROM peers").all();
  for (const peer of peers) {
    let shouldRemove = false;
    try {
      process.kill(peer.pid, 0);  // Only checks if process exists
    } catch {
      shouldRemove = true;
    }
    // NO LAST_SEEN CHECK!  <-- BUG
    if (shouldRemove) {
      db.run("DELETE FROM peers WHERE id = ?", [peer.id]);
    }
  }
}
```

No TTL defined anywhere. A peer with `last_seen` from 4 hours ago stays registered indefinitely as long as the PID exists (which it does — the Claude process is still running, just the MCP server connection dropped).

**Fix (2026-04-05):** Added two thresholds:
- `HARD_STALE_THRESHOLD_MS` (10 minutes): remove peer regardless of PID status
- `STALE_THRESHOLD_MS` (5 minutes): remove only if PID is also dead
- Recent entries (<5 min): keep existing PID+PPID liveness check

### Bug 3 (MINOR): process.kill(pid, 0) treats EPERM as "dead"

**File:** `broker.ts` ~line 228-243 (listPeers) and ~line 88-120 (cleanStalePeers)

`process.kill(pid, 0)` can throw `EPERM` on macOS when the process exists but belongs to a different user. The code treats ANY exception as "process dead":

```typescript
try {
  process.kill(p.pid, 0);
} catch {
  return false;  // BUG: treats EPERM (alive) as dead
}
```

**Fix (2026-04-05):** New `isProcessAlive()` helper that distinguishes:
- `ESRCH` = process doesn't exist = dead
- `EPERM` = process exists but different permissions = alive
- Any other error = treat as alive (safe default)

## Things We Tried (That Didn't Work)

1. **Restarting the session** — fixes it temporarily but issue recurs after hours/days
2. **Setting `restore_on_startup=1` in Chrome prefs** — this was for a different bug (HEY browser auth), unrelated to peers
3. **Checking if broker was running** — broker was always running (KeepAlive). The issue was the MCP server losing connection, not the broker dying.

## Configuration

| Setting | Value | File |
|---------|-------|------|
| Heartbeat interval | 15 seconds | server.ts `HEARTBEAT_INTERVAL_MS` |
| Stale cleanup interval | 30 seconds | broker.ts `setInterval(cleanStalePeers, 30_000)` |
| Stale threshold | 5 minutes | broker.ts `STALE_THRESHOLD_MS` (added 2026-04-05) |
| Hard stale threshold | 10 minutes | broker.ts `HARD_STALE_THRESHOLD_MS` (added 2026-04-05) |
| Broker port | 7899 | server.ts `BROKER_PORT` |
| Broker DB | ~/.claude-peers.db | broker.ts `DB_PATH` |
| Broker process | LaunchAgent `com.rob.claude-peers-broker` or inline | broker.ts |

## How to Debug

### Check if your session is registered
```bash
sqlite3 ~/.claude-peers.db "SELECT id, pid, cwd, substr(summary,1,60), last_seen FROM peers"
```

### Check if broker is running
```bash
ps aux | grep "broker.ts" | grep -v grep
curl -s http://127.0.0.1:7899/
```

### Check MCP server processes
```bash
ps aux | grep "claude-peers" | grep -v grep
```

### Force re-registration
Restart the session. The new MCP server will register fresh on startup.

### Test peer connectivity
Spawn a test session and try to find/message the main session:
```bash
tmux new-session -d -s "test/peers" -c /tmp
tmux send-keys -t "test/peers" "claude --dangerously-skip-permissions --dangerously-load-development-channels server:claude-peers" Enter
# Wait, then send: "List all peers and send a test message to the atlas session"
```

## Status

**2026-04-05:** All 3 bugs fixed in server.ts and broker.ts. Changes: +70/-27 lines across 2 files. Needs monitoring over the next few days to confirm the fix holds for 24+ hour sessions.
