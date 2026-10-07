# claude-usage

Claude Code plan usage, framed against the time left in the same window.

`/usage` tells you that you're 60% through your weekly allowance. It doesn't tell you
whether that's fine. This does: for each rate-limit window it draws the usage bar, and
directly beneath it a bar for how much of the window has actually elapsed.

**Example**

<img width="594" height="251" alt="image" src="https://github.com/user-attachments/assets/e6e4ee5a-4410-43b6-9a2a-cdbe88f8e4ab" />


**If the usage bar is longer than the time bar, you're spending faster than an even split.**

<img width="594" height="251" alt="image" src="https://github.com/user-attachments/assets/7713ba68-1b51-46a1-89bc-46d5c1aeba21" />

Each window gets a pace figure (usage share ÷ elapsed share — `1.00x` is exactly an even
burn) and a projection of where you land at this rate. The header adds how long you'd have
to sit out to bring an over-pace window back to parity.

## Install

```bash
git clone https://github.com/robgibbons/claude-usage.git
sudo ln -sf "$PWD/claude-usage/claude-usage" /usr/local/bin/claude-usage
```

Requires `bash`, `jq`, `awk`, `curl`, and **GNU** `date` (it uses `date -d`, so Linux or
macOS with `coreutils` on PATH). The bars need a UTF-8 terminal.

## Usage

```bash
claude-usage              # live report, repainting in place every 60s
claude-usage --secs 30    # ...at a custom interval (a bare number works too)
claude-usage | cat        # piped or redirected: renders once and exits
```

It's built to live in a dedicated terminal pane. Piped or redirected output renders once
instead of looping, so `claude-usage | grep …` still terminates.

### As a status line

`--statusline` prints a compact one-line readout for Claude Code's status line:

```
Opus 5 · my-project  5h 38% 1.90x  7d 8% 0.26x
```

Register it in `~/.claude/settings.json`:

```json
"statusLine": {
  "type": "command",
  "command": "/usr/local/bin/claude-usage --statusline",
  "refreshInterval": 60
}
```

Use `--refresh` instead of `--statusline` to get the same cache refresh with no visible
status line.

## Where the numbers come from

Two sources feed one cache, and they cover for each other.

**The statusLine hook** is the cheap one. Claude Code hands `rate_limits` to a registered
status line command and nothing else, so wiring up the hook refreshes the figures for free
on every turn of every live session.

On its own it goes stale. A session is handed the `rate_limits` from its *last API
response*, so a session left idle replays frozen numbers for as long as it sits there —
sometimes for days, while the real windows roll underneath it.

**So the report also fetches** from the endpoint `/usage` itself reads, signing the request
with the OAuth token Claude Code already stores, to repair exactly that staleness.

That fetch is **throttled** — once every 5 minutes, backing off to 30 minutes after a 429.
The endpoint's budget is per-account and shared with Claude Code's own polling from every
running session, and it answers 429 for minutes on end once that budget is spent. Fetching
on every repaint spends it for no gain: usage only moves while a session is working, which
is precisely when the hook is already firing.

Neither source is trusted blindly. A status line payload is only written to the cache while
the 5-hour window it describes is still open, which is what tells a live reading apart from
an idle session's replay. The header reports how old the figures actually are rather than
which mechanism delivered them.

### About the token

The live fetch reads the OAuth access token from `~/.claude/.credentials.json` and sends it
to `api.anthropic.com` — the same request Claude Code makes, to the service that issued it.
It is strictly read-only with respect to that file: an expired token simply ends the fetch
and the report falls back to the cache. It will never spend the refresh token, because
refreshing rotates it, and racing Claude Code for that rotation would sign you out of the
CLI. Works without any of this too — with the status line hook alone, or with neither, in
which case it tells you so.

## Configuration

All optional, all environment variables.

| Variable | Default | What it does |
|---|---|---|
| `CLAUDE_USAGE_FETCH_SECS` | `300` | Minimum seconds between live fetches |
| `CLAUDE_USAGE_BACKOFF_SECS` | `1800` | How long to back off after a 429 |
| `CLAUDE_USAGE_BAR_WIDTH` | `30` | Bar width in characters |
| `CLAUDE_USAGE_CACHE` | `~/.claude/usage-cache.json` | Where figures are cached |
| `CLAUDE_USAGE_CREDENTIALS` | `~/.claude/.credentials.json` | Where the token is read from |
| `CLAUDE_USAGE_ACCOUNT` | `~/.claude.json` | Where the account label is read from |
| `CLAUDE_USAGE_API` | `https://api.anthropic.com` | API base URL |

## Notes

Rate-limit windows are 5 hours and 7 days. The account line under the title names whose
plan the figures are counting against, which matters if you switch logins; a personal
plan's auto-generated organization name is suppressed as noise, a real workspace name is
shown.

Not affiliated with Anthropic.
