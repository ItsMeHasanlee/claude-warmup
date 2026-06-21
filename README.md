<div align="center">

# WarmWindow

### Predictable reset windows for Claude Code

*One tiny scheduled ping. Your usage window resets on a clock **you** choose — not whenever you happened to start.*

[Quick start](#quick-start) · [How it works](#how-it-works) · [Configuration](#configuration) · [Security](#security) · [Responsible use](#compliance--responsible-use)

</div>

---

## What it is (and what it isn't)

Claude Pro/Max usage runs on a rolling **5-hour window**: it starts the moment you send your first message and resets 5 hours later. Because it's anchored to *when you started*, your reset time drifts day to day — and often lands right in the middle of your work.

**WarmWindow** sends one minimal message on a fixed schedule, so your window always opens at times *you* pick. No more guessing when the next reset hits.

> **The honest part — read this first.**
> WarmWindow does **not** give you extra usage, raise your limits, or bypass anything. It only makes your reset times **predictable**. One ping per window is all it takes; more would simply be waste. If you're looking for "more quota," this isn't that tool — and no honest tool is.

That honesty *is* the product. Most "limit" hacks promise to beat the system. WarmWindow promises one thing and does it cleanly: **predictability you can build a routine around.**

## Why it's different

| | **WarmWindow** | Typical warmup scripts |
|---|---|---|
| Promise | Predictable resets, stated plainly | "Beat the limit / unlock hours" |
| Schedule | **Your choice** — 24/7 cadence *or* workday-anchored | Usually one fixed mode |
| Footprint | Leanest possible ping — Haiku, no session persisted | Often heavier prompts |
| Infra | Pure GitHub Actions — no servers, **$0** | Sometimes an external host |
| Security | Read-only token scope, no PR triggers, encrypted secret | Varies |
| Stance | Transparent + compliance-first | Often silent on the terms |

## How it works

```
GitHub Actions (your schedule)
        |
        v
  ephemeral runner  ->  installs the Claude Code CLI
        |
        v
  claude -p "hello world!" --model haiku --no-session-persistence
        |   (authenticates with YOUR OAuth token, stored as an encrypted secret)
        v
  your 5-hour window opens at a predictable time  ->  runner is destroyed
```

Nothing is stored. The runner is wiped after each run, so there is no chat history and no trace left behind.

## Quick start

1. **Get a long-lived token** (requires Claude Pro/Max):
   ```bash
   npm install -g @anthropic-ai/claude-code
   claude setup-token            # prints a token starting with sk-ant-oat...
   ```
2. **Use this template** to create your own repo (keep it private if you prefer).
3. **Add the token as a secret:** `Settings -> Secrets and variables -> Actions -> New repository secret`
   - **Name:** `CLAUDE_CODE_OAUTH_TOKEN`
   - **Value:** the token from step 1
4. **Pick your schedule:** edit the `cron` line in `.github/workflows/warmup.yml` (see [Configuration](#configuration) — times are UTC).
5. **Test:** `Actions -> WarmWindow -> Run workflow`. A green check means you're done.

## Configuration

GitHub Actions cron runs in **UTC only** (there is no timezone field), so you convert from your local time. Pick whichever mode fits how you work.

### Mode A — Predictable all day (default)

A ping every 6 hours, around the clock. Best if your hours are irregular or you just want resets you can rely on at any time.

```yaml
on:
  schedule:
    - cron: '0 0,6,12,18 * * *'   # 00:00 06:00 12:00 18:00 UTC
```

**Why 6 hours, not the 5-hour window length?** The extra hour is a deliberate **buffer** — it guarantees each ping lands *after* the previous window expired, so you always start a clean one. And 6 divides 24 evenly (4 equal slots a day), so the schedule stays locked to the same clock times forever, with **no drift**.

### Mode B — Anchored to your workday (great for 9-to-5)

One warmup a few hours before you start, so your window resets *into* your morning instead of at a random time. The window is **5 hours**, so:

> **reset time = warmup time + 5 hours**

Want to arrive at 09:00 and see *"resets in 2h"* (a fresh window at 11:00)? Warm up at **06:00 local** — `06:00 + 5h = 11:00`. You walk in on a nearly-full window that refreshes at 11:00, then your own work rolls the next window into the afternoon.

Convert your local warmup time to UTC for the cron:

| You start work | Warm up (local) | Timezone | cron (UTC) |
|---|---|---|---|
| 09:00 | 06:00 | London (UTC+0) | `0 6 * * 1-5` |
| 09:00 | 06:00 | New York (UTC-5) | `0 11 * * 1-5` |
| 09:00 | 06:00 | Baku (UTC+4) | `0 2 * * 1-5` |

> Rule of thumb: `UTC = local - your_offset`. The `1-5` means Monday-Friday (drop it for every day).

### Heads-up: GitHub cron is best-effort

Scheduled runs on GitHub (like free cron anywhere) usually fire a few minutes late and, under heavy load, can occasionally be **delayed 10-30 min or skipped**. Both modes are built around this: Mode A's 6-hour buffer and Mode B's 3-hour head start absorb the slippage, so a late ping still lands where you need it.

> **Tip:** check your current window any time with `/usage` in Claude Code, or at claude.ai/settings/usage.

## Security

- Your token lives in an **encrypted GitHub Actions secret** — never in the code, never in git history.
- The workflow references it only as `${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}` and **never prints it**; GitHub masks secrets in logs.
- The workflow has **no `pull_request` trigger**, so a fork can never run it against your secret.
- Default workflow token permissions are set to **read-only**.
- If a token is ever exposed, revoke it and run `claude setup-token` again.

## Compliance & responsible use

WarmWindow is designed to be used **honestly and within the rules**:

- It uses **only** the official Claude Code CLI and **your own** account credentials.
- It sends **one minimal request** per window. It does not scrape, parallelize, share accounts, or attempt to exceed any limit.
- It grants **no extra quota** — only predictable timing.

**You are responsible** for ensuring your usage complies with [Anthropic's Consumer Terms](https://www.anthropic.com/legal/consumer-terms) and [Usage Policy](https://www.anthropic.com/legal/aup). Automated, scheduled requests against your own account are at your own discretion and risk. If those terms prohibit scheduled keep-alive requests, **do not use this tool.**

## FAQ

**Does this get me more usage?** No. Same limits — just predictable reset times.

**Will it clutter my chat history?** No. Runs are headless on a throwaway runner and never appear in claude.ai.

**Does it cost anything?** No servers and no API billing — it runs on free GitHub Actions, and the single Haiku ping sits inside your existing subscription.

**Is my token safe in a public repo?** Yes. Secrets are encrypted and never stored in files. See [Security](#security).

## Credits

WarmWindow builds on a warmup technique the Claude developer community has explored in several open-source projects. Its focus is a clean, honest, security-first take: minimal footprint, transparent about what it does and doesn't do, and easy to align to your own schedule and timezone.

## Disclaimer

WarmWindow is an independent, community-maintained project. It is **not affiliated with, endorsed by, or sponsored by Anthropic**. "Claude" and "Claude Code" are trademarks of Anthropic, PBC, used here purely descriptively to indicate compatibility. The software is provided "as is", without warranty of any kind. See [LICENSE](LICENSE).

## License

[MIT](LICENSE) © 2026
