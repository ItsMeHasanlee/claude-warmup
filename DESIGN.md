# Design notes: why a simple fixed schedule

WarmWindow uses a plain fixed-interval cron and nothing more: no stored state, no usage polling, no adaptive retries. This note explains why that simple design is also the most effective one, including for the tricky cases where a ping lands at an awkward moment.

## What warmup can and cannot do

The Claude Pro/Max usage limit is a rolling 5-hour window. It is server-side state tied to your account:

- The window starts at your first message after an idle gap, and resets exactly 5 hours later.
- A message anchors a new window only if no window is currently active. Otherwise it just spends a little of the current window's budget.
- Once the budget is exhausted, further messages are rejected until the window resets. A rejected message spends nothing and changes nothing.

WarmWindow gives you no extra quota. Its only job is to control when the window starts, so your reset time is predictable.

## The three situations a ping can hit

When a scheduled ping fires, exactly one of these is true:

1. No active window (you are idle): the ping starts a fresh window. This is the intended effect, and it anchors the reset to a predictable clock time.
2. Active window, budget not exhausted: the ping is accepted, costs about one Haiku "hello", and does not change the window. Harmless.
3. Budget exhausted: the ping is rejected (HTTP 429), costs nothing, and does not change the window. Harmless.

## Key insight: warmup only matters while you are idle

While you are actively using Claude, your own messages already anchor each new window the moment the previous one resets. A ping during active use either lands in a live window (situation 2) or is rejected (situation 3). Either way it changes nothing.

So warmup has an effect only during idle periods: before your day starts, overnight, or in long gaps. There, a fixed ping anchors the next window to a predictable time. That is the whole value.

## Why a fixed schedule is optimal

The argument, step by step:

- The window is server-side and resets exactly 5 hours after it starts. Warmup cannot add quota; it only sets the start time.
- During active use, your own messages govern the windows, so warmup is a no-op then.
- Both awkward cases (situations 2 and 3) are harmless under a fixed schedule. An accepted ping is nearly free, a rejected ping is a no-op, and because pings are spaced (every 6 hours, not retried in a tight loop) there is no cascade.
- Therefore the fixed schedule reaches the goal (predictable resets, and a fresh window when you start) at minimal cost, with no state to manage.

## Why an adaptive "retry until it works" scheme is worse

A natural idea is to detect an exhausted window and retry every hour until a ping gets through. It looks helpful but it is a net loss:

- It destroys predictability. An exhausted window resets at "whenever it happened to start, plus 5 hours", which depends on your irregular usage. Re-anchoring to that moment drifts your reset off the clock, which defeats the entire purpose.
- It sends more failing pings. Every hourly retry during an exhausted window is rejected until the reset.
- It adds complexity and a fragile dependency. Knowing the current state requires polling an unofficial usage endpoint and keeping state, both of which can break.

The only thing it "saves" is a few microscopic Haiku pings. That is a large cost for no real benefit.

## Why 6 hours, not 5

The window is 5 hours, but the schedule pings every 6. The extra hour is a buffer: during idle periods it guarantees each ping lands after the previous window expired, so it reliably starts a fresh one. Six also divides 24 evenly, so the four daily slots stay locked to the same clock times with no drift. This buffer is the only tuning the design needs.

## One honest caveat

Occasionally a scheduled ping will land inside an exhausted window and show up as a failed (red) run in the Actions tab. This is harmless and self-healing: it spends nothing, changes nothing, and the next scheduled ping succeeds. Adding logic to avoid the occasional red run would cost far more than it is worth, so WarmWindow deliberately keeps it simple.

## Conclusion

Given the goal (predictable resets and a fresh window at a time you choose, with no extra quota possible), a simple stateless fixed schedule maximizes predictability and robustness at minimal cost, and no adaptive scheme improves on it. The simplest design is also the healthiest one.
