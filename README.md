# Claude Warmup

A tiny GitHub Actions workflow that keeps your **Claude Pro/Max 5-hour usage window resetting at predictable times**.

## Why

Claude Pro/Max usage runs on a rolling 5-hour window: the window starts when you send your first message and resets 5 hours later. The annoyance is that reset times drift and are hard to predict.

This workflow sends one minimal `.` prompt at fixed times, so the window always starts on a clean schedule and you always know when your next reset is.

> **Honest note:** this does **not** give extra usage or bypass any limit. It only makes the reset schedule *predictable*. One ping per window is all it takes; sending more often just wastes quota for zero benefit.

## How it works

```
GitHub Actions (cron)  ->  spins up a runner  ->  installs Claude Code
                                                        |
                                                        v
                                        claude -p "."   (uses your OAuth token)
                                                        |
                                                        v
                                  5-hour window starts, runner is destroyed
```

The runner is ephemeral and leaves no chat history anywhere.

## Schedule

Runs every 6 hours: `00:00, 06:00, 12:00, 18:00 UTC`.

6 hours (not 5) leaves a ~1-hour buffer, so a slightly delayed run still lands in a fresh window. 6 divides 24 evenly, so the day is split cleanly with no drift. Edit the `cron` line in `.github/workflows/warmup.yml` for your own timezone.

## Setup

1. **Generate a long-lived token** (requires Claude Pro/Max):
   ```bash
   npm install -g @anthropic-ai/claude-code
   claude setup-token
   ```
   Copy the token it prints (it starts with `sk-ant-oat...`).

2. **Add it as a repository secret:**
   `Settings -> Secrets and variables -> Actions -> New repository secret`
   - **Name:** `CLAUDE_CODE_OAUTH_TOKEN`
   - **Value:** the token from step 1

3. **Test it:** `Actions` tab -> `Claude Warmup` -> `Run workflow`. A green check means you're done. After that it runs on its own.

## Security

- The token is stored as an **encrypted GitHub Actions secret**. It is never written to any file in this repo and never appears in commit history.
- The workflow references it only as `${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}` and never prints it; GitHub masks secrets in logs.
- **Never** hardcode the token into the workflow or any file, and never add a step that echoes it.
- If a token is ever exposed, revoke it and run `claude setup-token` again to issue a new one.

## Disclaimer

Unofficial, personal-use helper. Use it in line with Anthropic's usage policies. It uses your own account credentials and sends a single trivial request per window.
