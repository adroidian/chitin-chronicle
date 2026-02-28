# HOW IT WORKS - Mechanical Truth

**Audience:** The CEO who needs to audit the engine, not trust the dashboard.

## File Locations & Formats

All data lives in `editorial/`:

```
editorial/
├── registry.json          # Master content list (all entries)
├── ledger.json            # Publication log (append-only)
├── timeline.json          # Series tracking (read-only by this script)
├── claims/
│   ├── *.claim            # Active claims (2hr TTL)
│   └── archive/           # Expired/released claims
```

**Format:** Plain JSON. No database. No transactions. Just files.

### registry.json
```json
[
  {
    "id": "content-id",
    "title": "Post Title",
    "type": "post",
    "status": "published",
    "author": "agent-name",
    "channels_published": ["twitter", "telegram"],
    "created_at": "2026-02-28T...",
    "published_at": "2026-02-28T..."
  }
]
```

### ledger.json
```json
[
  {
    "content_id": "content-id",
    "title": "Post Title",
    "channel": "twitter",
    "author": "agent-name",
    "published_at": "2026-02-28T...",
    "url": "https://twitter.com/...",
    "status": "published"
  }
]
```

### claims/*.claim
```json
{
  "agent": "vesper",
  "content_id": "content-id",
  "action": "publish",
  "channel": "twitter",
  "claimed_at": "2026-02-28T17:30:00.000Z"
}
```

Filename pattern: `{content-id}-{agent}.claim`

---

## Command Flow - Step by Step

### `editorial claim <content-id> <action> <channel>`

**What happens:**

1. **Agent identity:** Read from `OPENCLAW_AGENT` env var, fallback to `USER`, fallback to `"unknown"`
   - **Gap:** Easily spoofed. No verification.

2. **Conflict check:** Read all `*.claim` files in `claims/` directory
   - Parse each JSON file
   - Check if claim is still valid (< 2 hours old)
   - If expired, move to `archive/` immediately
   - If another agent claimed same `content_id` + `channel`, exit with error

3. **Write claim file:** `claims/{content-id}-{agent}.claim`
   - Contains: agent, content_id, action, channel, timestamp
   - **No atomic write.** Standard `fs.writeFileSync()`.

4. **Git commit:** `git add . && git commit -m "..."`
   - **Fails silently** if not in a git repo or no changes
   - **No verification** that commit succeeded

**Exit codes:**
- `0` = success
- `1` = conflict or usage error

**Race condition window:** Between step 2 (read claims) and step 3 (write claim). Another agent could write a claim in this gap.

---

### `editorial check <content-id> <channel>`

**What happens:**

1. **Agent identity:** Same as claim (env vars)

2. **Conflict check:** Same logic as claim
   - If another agent has active claim, exit 1 with conflict message

3. **Publication check:** Read `ledger.json`
   - Search for matching `content_id` + `channel`
   - If found, exit 0 and show publication details

4. **No claim file written.** This is read-only.

**Exit codes:**
- `0` = safe to publish (or already published)
- `1` = conflict detected

**Race condition window:** Between check and actual publish. Another agent could publish during this gap.

---

### `editorial publish <content-id> <channel> <url> [title]`

**What happens:**

1. **Agent identity:** Same env var logic

2. **Update ledger:** `ledger.json`
   - Read entire file
   - Append new publication record
   - Write back (full file replace)
   - **No atomic append.** Another publish could clobber this.

3. **Update registry:** `registry.json`
   - Read entire file
   - Find entry by `id` or create new
   - Update `status` to "published"
   - Add `channel` to `channels_published` array (if not already there)
   - Write back (full file replace)
   - **No atomic update.** Another publish could clobber this.

4. **Archive claim:** Move `claims/{content-id}-{agent}.claim` to `archive/`
   - **Advisory only.** If agent didn't claim first, this is a no-op.

5. **Git commit:** `git add . && git commit`
   - Silent failure tolerance

**Race condition window:** 
- Between ledger read and write
- Between registry read and write
- Another agent could publish the same content to same channel simultaneously
- **Both publishes would succeed.** You'd get duplicate ledger entries.

---

### `editorial release <content-id>`

**What happens:**

1. **Agent identity:** Same env var logic

2. **Archive claim:** Move `claims/{content-id}-{agent}.claim` to `archive/`
   - Filename must match exactly: `{content-id}-{agent}.claim`
   - If file doesn't exist, silent no-op

3. **Git commit:** Standard silent-fail commit

**Gap:** If another agent published while you held the claim, releasing does nothing to prevent that. This is cleanup only.

---

### `editorial status`

**What happens:**

1. **Read all claims:** List `claims/*.claim`, parse each
   - Auto-expire claims > 2 hours old (move to archive)
   - Show agent, content_id, action, channel, age in minutes

2. **Read ledger:** Show publications from last 48 hours
   - Sorted by published_at (newest first)

3. **Read timeline:** Show latest day for each series
   - **Timeline is not written by this script.** External dependency.

4. **Summary stats:** Count registry entries, ledger entries, active claims

**Output only.** No state changes (except auto-expiring old claims).

---

## The Race Condition

**What it is:**

Time-of-check to time-of-use (TOCTOU) race. The gap between reading state and writing state.

**When it happens:**

1. **Claim race:** Agent A and Agent B both check for conflicts → both see none → both write claims → last write wins, first claim is overwritten
   - **Filesystem doesn't prevent this.** No file locking.

2. **Publish race:** Agent A and Agent B both publish same content to same channel simultaneously
   - Both read ledger → both append → last write wins, one entry lost
   - Both read registry → both update → last write wins, one channel lost

3. **Check-to-publish race:** Agent A checks (safe) → Agent B publishes → Agent A publishes → duplicate publication

**How likely:**

- **Low risk** if agents are well-behaved and use the system correctly
- **High risk** if multiple agents process same content queue simultaneously
- **Guaranteed failure** if two agents run `publish` at exact same second

**Impact:**
- **Claim race:** Silent conflict. One agent thinks they claimed it, but their claim file was overwritten.
- **Publish race:** Duplicate ledger entries. Lost registry updates. Confusion.

**Mitigation:**
- Hope agents don't collide (current strategy)
- 2-hour TTL auto-expires stale claims (helps with abandoned work)
- Git history provides audit trail (can manually reconcile)

**What would actually prevent it:**
- File locking (flock)
- Atomic compare-and-swap operations
- SQLite with transactions
- Distributed lock service (Redis, etcd)
- **None of these are implemented.**

---

## Advisory vs Structural

### Advisory (You Have to Remember)

**Everything is advisory.** The system has no enforcement power.

1. **Calling the script:** Agents must remember to run `editorial check` before publishing
   - Nothing stops an agent from just publishing directly
   - No hook in the message tool
   - No process wrapper

2. **Agent identity:** Read from environment variables
   - Any agent can set `OPENCLAW_AGENT=vesper` and impersonate
   - No cryptographic identity
   - No authentication

3. **Git commits:** Fail silently
   - If not in a git repo, claim/publish still succeeds
   - If git isn't installed, claim/publish still succeeds
   - No verification that state was persisted

4. **2-hour TTL:** Passive cleanup
   - Doesn't prevent an agent from ignoring an active claim
   - Just auto-expires old claims when status/check reads them
   - Not enforced during publish

5. **Conflict detection:** Depends on agents running `check` or `claim` first
   - If agent skips straight to `publish`, no conflict detection happens
   - Ledger check in `check` command only warns, doesn't block

### Structural (Actually Enforced)

**Nothing is structurally enforced.**

The closest thing to enforcement:
- **Exit codes:** `claim` and `check` exit 1 on conflict
  - But agents must check exit codes
  - Agents could ignore and proceed anyway

---

## Gaps - The Honest List

### 1. **No Enforcement Layer**
The skill provides a coordination tool. It doesn't prevent bad behavior. An agent that ignores the tool or misuses it will break the system.

### 2. **No Atomic Operations**
All JSON updates are read-modify-write. No transactions. No locking. Race conditions are possible and will eventually happen at scale.

### 3. **Git as a "Lock" is Fiction**
Git commits are advisory logging, not mutual exclusion. Multiple agents can commit simultaneously. Git will auto-merge or create conflicts, but the JSON files will be corrupted.

### 4. **Identity is Not Verified**
Agent names come from environment variables. Any agent can claim to be any other agent. No PKI. No signatures.

### 5. **No Rollback**
If a publish operation fails halfway through (ledger written, registry write fails), there's no automatic rollback. Manual cleanup required.

### 6. **No Notification System**
If an agent ignores a claim or publishes over another agent's work, the original agent never finds out (unless they run `status`).

### 7. **TTL Cleanup is Passive**
Expired claims are only cleaned up when another agent reads the claims directory. If nobody runs `status` or `check` for 3 hours, expired claims sit there.

### 8. **Timeline is External**
The `timeline.json` file is displayed in `status` but never written by this script. It's a phantom dependency. If it doesn't exist or has bad data, status output is misleading.

### 9. **No Validation**
- No schema validation on JSON files
- No URL validation
- No content-id format requirements
- Garbage in, garbage out

### 10. **Silent Failures Everywhere**
- Git commit failures ignored
- Missing files treated as empty arrays
- Invalid JSON files skipped silently
- Archive failures ignored (if source file doesn't exist)

---

## What Would Break It in Production

### Guaranteed Breaks

1. **High concurrency:** 5+ agents publishing simultaneously → JSON corruption, lost updates, duplicates

2. **Filesystem issues:** NFS lag, network drive latency, filesystem full → corrupted JSON, lost claims

3. **Git conflicts:** Multiple agents committing simultaneously → merge conflicts in JSON files → manual intervention required

4. **Clock skew:** If agent clocks are out of sync, TTL checks fail, claims expire prematurely or never expire

5. **Process killed mid-write:** Agent crashes between ledger write and registry write → inconsistent state

### Likely Breaks

1. **Abandoned claims:** Agent crashes without releasing claim → 2-hour lockout on that content (until auto-expire)

2. **Duplicate publications:** Agent A checks, Agent B publishes, Agent A publishes → same content on same channel twice

3. **Identity confusion:** Multiple agents with same name (e.g., `OPENCLAW_AGENT=vesper` on two machines) → claims overwrite each other

4. **Ledger bloat:** Append-only ledger grows unbounded → slow reads, large git repo, memory issues

### Edge Cases

1. **Archive directory full:** Filesystem runs out of inodes → archiving fails → claims pile up

2. **JSON parse errors:** Manual edit introduces syntax error → all agents fail to read registry/ledger → system unusable until fixed

3. **Git repo corruption:** `.git` directory deleted or corrupted → all git operations fail → system continues but no audit trail

4. **Timezone bugs:** Timestamps are ISO 8601 but no timezone normalization → comparison logic breaks if agents use different timezones

---

## The Bottom Line

**What this is:**
A lightweight, file-based coordination system for preventing duplicate work in a low-traffic, single-instance agent environment.

**What this is NOT:**
- A production-grade distributed lock service
- A database with ACID guarantees
- A system that can handle concurrent writes safely
- An enforcement mechanism (it's advisory)

**When it works:**
- Single agent publishing a few times per day
- Occasional multi-agent coordination with manual deconfliction
- Low stakes (duplicate tweet is annoying, not catastrophic)

**When it breaks:**
- Multiple agents publishing simultaneously
- High throughput (dozens of publishes per hour)
- Critical deduplication requirements (must never duplicate)
- Hostile agents (intentionally ignoring claims)

**The honest assessment:**
This is a "good enough for now" solution that will work until it doesn't. It's better than no coordination at all. It's not production-ready for a multi-agent swarm operating at scale.

If you need real guarantees, replace the JSON files with SQLite and add file locking. Or use Redis with SETNX. Or accept that occasional duplicates will happen and build detection/cleanup tooling instead.

**Trade-offs:**
- **Simplicity:** No dependencies, just Node.js + filesystem + git
- **Auditability:** Git history shows every claim/publish action
- **Debuggability:** JSON files are human-readable and editable
- **Cost:** No infrastructure, no database server, no network calls

The price you pay: **No guarantees.**

Choose wisely.
