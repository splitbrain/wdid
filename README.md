# wdid — what did I do?

A small CLI that reports your Claude Code activity for a given day, derived
from the JSONL session logs under `~/.claude/projects/`.

## Usage

```
./wdid                  # today
./wdid yesterday
./wdid 2026-04-30
./wdid --no-summarize   # skip Haiku-based descriptions
```

Output:

```
START  END    DUR    PROJECT  DESC
-----------------------------------------------------------------------------------
09:12  10:04  00:48  api      Add rate limiting middleware and integration tests
10:20  11:35  01:11  webapp   Migrate user settings page to new component library
13:30  14:02  00:31  scripts  Investigate slow nightly backup job
-----------------------------------------------------------------------------------
       total  02:30  (3 sessions)
```

## How it figures out engagement

A single Claude Code session log can span many real working sessions
(`claude --resume` reattaches to old logs hours or days later). `wdid`
walks each log, sorts events by timestamp, and splits into engagement
sessions wherever the gap between consecutive events exceeds 30 minutes.

Engagement time per session is the sum of consecutive in-session gaps
(naturally capped at 30 min by the splitting rule), plus a 3-minute
trailing read buffer when the session ends on an assistant message — the
logs have no close/exit signal, so we credit roughly the same idle window
that triggers Claude's own `away_summary`.

Subagent logs (`.../subagents/...`) are skipped — they overlap with their
parent session and would double-count time.

## Session descriptions

The `DESC` column comes from, in order of preference:

1. A Haiku-generated summary (cached at `~/.cache/wdid/titles-YYYY-MM-DD.json`).
   The summary is built from your prompts, the tool actions Claude took
   (file edits, shell commands), and the last few assistant replies, so it
   tends to reflect what was actually done rather than just the opening
   prompt.
2. Claude's own `ai-title` entry, if Haiku is disabled or fails.
3. The first non-slash user prompt.

## Requirements

- Python 3.9+ (standard library only)
- Optional: `ANTHROPIC_API_KEY` environment variable for Haiku-based
  summaries. Without it, `wdid` falls back to `ai-title` / first prompt.

The summarizer uses `claude-haiku-4-5-20251001` via the Anthropic
Messages API, called directly through `urllib`. Each title is cached, so
subsequent runs make zero API calls.
