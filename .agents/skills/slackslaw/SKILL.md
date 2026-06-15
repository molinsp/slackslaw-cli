---
name: slackslaw
description: >-
  Query a local Slack archive (channels + DMs) via the slackslaw CLI — search
  messages, view threads, list what's new, ask natural-language questions
  against the archive, or sync new data from Slack. Use this skill whenever the
  user wants to find, recall, or reason about Slack content (1:1 DMs, group
  DMs, public/private channels) for the `dialecticanet` workspace. The local
  archive is the source of truth; no live Slack API hit is needed for
  read-only operations. Always use this skill for anything slackslaw or
  slackdump-archive related.
---

# slackslaw

Local-first CLI for querying a Slack archive that lives in
`~/.slackslaw/workspaces/<workspace>/slackdump.sqlite`. Built on top of
`slackdump` v4. All read operations are offline SQLite queries.

## Workspace facts (this machine)

- Workspace: **`dialecticanet`** (canonical `dialecticanet.slack.com`)
- Archive dir: `~/.slackslaw/workspaces/dialecticanet/`
- DB: `~/.slackslaw/workspaces/dialecticanet/slackdump.sqlite`
- Auth (do not delete): `~/Library/Caches/slackdump/dialecticanet.bin`
- Curated channel list (URLs): `~/Documents/slack_channles_to_sync_locally.txt`

## Most common commands

```bash
# Search across all archived channels + DMs
slackslaw search "experiences redshift" --after 14d
slackslaw search "deployment" --workspace dialecticanet --limit 50
slackslaw search "from:jason.tunstall new joiner"   # inline author filter
slackslaw search "in:team-all-product deploy"       # inline channel filter
slackslaw search "expr" --mine                # only messages relevant to me
slackslaw search "expr" --json                # JSONL for piping
slackslaw search "expr" --format slack        # raw Slack mrkdwn

# Dump a thread by Slack URL (offline, no API call)
slackslaw threads 'https://dialecticanet.slack.com/archives/C08GCPL8P28/p1779203319178329'

# Messages received since last sync
slackslaw whatsnew
slackslaw whatsnew --mine                     # filtered to me + keywords
slackslaw whatsnew --json

# Unread messages
slackslaw unread
slackslaw unread --json

# LLM-ready conversation context
slackslaw context --since 7d
slackslaw ask "summarise the dataeng-support thread on cross-cluster sharing"

# Archive stats / status
slackslaw status
slackslaw stats

# Raw SQL escape hatch (slackdump v4: MESSAGE.TXT, S_USER, author in DATA JSON)
slackslaw schema [--workspace dialecticanet]
slackslaw sql 'SELECT COUNT(*) FROM MESSAGE;'
slackslaw sql 'SELECT USERNAME FROM USER;' --workspace dialecticanet   # USER→S_USER alias
```

## slackdump v4 schema (for hand-written SQL)

| Table | Key columns | Notes |
|---|---|---|
| `MESSAGE` | `TS`, `CHANNEL_ID`, `TXT`, `DATA` | Body is `TXT` (not `TEXT`). Author: `json_extract(CAST(m.DATA AS TEXT), '$.user')` |
| `S_USER` | `ID`, `USERNAME`, `DATA` | Not `USER`. `real_name` / `name` also in `DATA` JSON |
| `CHANNEL` | `ID`, `NAME` | Dedupe with `GROUP BY ID` when joining |

Prefer `slackslaw search "from:USER keywords"` over raw SQL when possible.

## Sync

The sync layer wraps `slackdump archive` / `slackdump resume`. Both passes
(channels + DMs) write into the same SQLite DB; messages dedupe via UPSERT.

### Incremental sync (the daily/weekly one)

```bash
caffeinate -i -s slackslaw sync \
  --filter-channels ~/Documents/slack_channles_to_sync_locally.txt
```

Drop `--full` and `--since` — the resume engine picks up from the checkpoint.

- **Pass 1 (channels):** `slackdump resume` — only new messages since checkpoint.
- **Pass 2 (DMs):** `slackdump archive -chan-types im,mpim` — re-crawls DMs but UPSERTs dedupe.

If `slackdump resume` fails (e.g. schema mismatch after upgrade), slackslaw
auto-falls-back to a full archive run for that pass.

### Full sync (after a wipe, or first time)

```bash
caffeinate -i -s slackslaw sync --full \
  --filter-channels ~/Documents/slack_channles_to_sync_locally.txt \
  --since 365d
```

`--since 365d` caps how far back to crawl (defaults to all-time if omitted).

### Background sync (survives terminal close)

```bash
nohup caffeinate -i -s slackslaw sync \
  --filter-channels ~/Documents/slack_channles_to_sync_locally.txt \
  > /dev/null 2>&1 &
tail -f ~/.slackslaw/slackslaw.log
```

### Selecting channels — flag semantics (important)

| Flag | Channels | DMs | Group DMs |
|---|---|---|---|
| `slackslaw sync --full` | ✅ all | ✅ | ✅ |
| `slackslaw sync --full --filter-channels F` | only those in F | ✅ | ✅ |
| `slackslaw sync --full -c general,ops` | only `general`,`ops` | ❌ | ❌ |

- `--filter-channels <path>` = "full sync, just filter channels via file" — DMs stay included via Pass 2.
- `-c name1,name2` = restrictive single-purpose targeting — no DMs.

The file at `~/Documents/slack_channles_to_sync_locally.txt` contains Slack
**archive URLs** (e.g. `https://dialecticanet.slack.com/archives/C0123ABCD`)
one per line, `#` for comments. URLs skip the slackdump name→ID resolver,
which avoids `slackdump list channels` rate-limit pain on large workspaces.

## Auth (re-auth) — when sync errors with `004 Authentication Error`

slackdump's default browser auth has known issues with current Chrome 148.x
(Rod's `--no-startup-window` flow stalls). Use **legacy browser** mode:

```bash
slackdump workspace new -legacy-browser dialecticanet
# Firefox window pops up (Playwright bundled, ~70 MB download first time)
```

If browser-based auth is unavailable, fall back to manual token + cookie:

1. In Chrome at https://app.slack.com (logged into dialecticanet) → DevTools console:
   - `JSON.parse(localStorage.localConfig_v2).teams[Object.keys(JSON.parse(localStorage.localConfig_v2).teams)[0]].token` → `xoxc-…`
   - `document.cookie.split('; ').find(c => c.startsWith('d=')).slice(2)` → `xoxd-…`
2. Then:
   ```bash
   slackdump workspace new -token 'xoxc-…' -cookie 'xoxd-…' dialecticanet
   ```

`slackslaw auth dialecticanet` exists but calls the default (broken) Rod path;
prefer `slackdump workspace new -legacy-browser dialecticanet` until upstream
fixes it.

## Sleep / network gotchas

- Wrap long syncs with `caffeinate -i -s` to prevent idle/system sleep.
- `caffeinate` does **not** prevent clamshell sleep on battery. Keep the lid
  open, or plug in AC + disable "Prevent automatic sleep when display is off".
- If a sync hangs (0% CPU, no DB writes for several minutes), the network
  socket is dead from a sleep event. Kill the slackdump PID and re-run — the
  archive resumes from checkpoint.

## Useful DB queries (when CLI commands don't fit)

```bash
DB=~/.slackslaw/workspaces/dialecticanet/slackdump.sqlite

# How many messages, files, channels?
sqlite3 "$DB" "SELECT (SELECT COUNT(*) FROM MESSAGE) AS messages,
                       (SELECT COUNT(*) FROM FILE)    AS files,
                       (SELECT COUNT(DISTINCT ID) FROM CHANNEL) AS channels;"

# Top channels by message count
sqlite3 -header -column "$DB" "
SELECT COALESCE(c.NAME, m.CHANNEL_ID) AS channel, COUNT(*) AS cnt
FROM MESSAGE m LEFT JOIN CHANNEL c ON c.ID = m.CHANNEL_ID
GROUP BY m.CHANNEL_ID ORDER BY cnt DESC LIMIT 20;
"

# Find a DM by counterparty name
sqlite3 "$DB" "
SELECT m.TS, m.TXT
FROM MESSAGE m JOIN CHANNEL c ON c.ID = m.CHANNEL_ID
WHERE c.NAME LIKE '%giorgos.kountouris%' AND m.TXT LIKE '%experiences%'
ORDER BY m.TS DESC LIMIT 20;
"
```

## When to use which command

| User wants | Command |
|---|---|
| Find a message they remember | `slackslaw search "<keywords>" --after 30d` or `slackslaw search "from:USER keywords"` |
| See messages since last sync | `slackslaw whatsnew --mine` |
| Read a specific thread | `slackslaw threads <slack-url>` |
| Ask a question over Slack history | `slackslaw ask "<question>"` |
| LLM-ready context for an agent | `slackslaw context --since 7d` |
| Refresh the archive | `slackslaw sync --filter-channels …` (above) |
| Inspect DB structure / counts | `slackslaw schema` or `slackslaw stats` |

## Pitfalls

- **Don't pass bare channel names** to `slackdump archive` — it requires IDs or
  URLs (`invalid link: <name>`). slackslaw's resolver handles this for
  `-c name` and `--filter-channels` with names, but URLs in the file are
  always the fastest path.
- **Don't run two sync invocations in parallel** against the same DB — SQLite
  WAL contention will slow both. Run sequentially.
- **`slackdump list channels` is rate-limit-prone** at 1000+ channels; use
  `-member-only` (slackslaw does this automatically when resolving names).
- **`-time-from` requires `YYYY-MM-DD`** (date only) in slackdump 4.3.0 —
  RFC3339 is rejected. slackslaw formats this correctly.

## Reference

- Source: `slackslaw.sh` in this repo (install via symlink to `/usr/local/bin/slackslaw`)
- Agent skill: `.agents/skills/slackslaw/SKILL.md`
- slackdump 4.3.0 (Homebrew) — `slackdump help` for upstream docs
- The slackslaw script supports `--quiet` for cron-style invocations.
