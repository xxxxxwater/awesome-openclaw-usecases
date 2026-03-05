# Telegram Stream Reply Reliability Hardening

## Summary

When OpenClaw is used through Telegram for long-running requests, users can occasionally see unstable delivery behavior. Two high-impact symptoms are common:

- users see an in-progress/streaming state but never receive a final answer, and
- users receive **two assistant outputs** for one turn (a streamed message, then a second aggregated final resend).

This use case hardens the reply pipeline so every accepted user message has a deterministic and clean UX: **one turn, one visible assistant message, streamed in place**.

## Problems

In production-like traffic, four edge cases commonly cause Telegram reply issues:

1. **Session drift**: active conversation context changes (`/new`, `/reset`, or concurrent turns) before the original turn completes.
2. **Weak dedup keys**: legitimate final messages are classified as duplicates in retry/concurrency scenarios.
3. **No user-visible fallback**: send failures are only logged and never surfaced to Telegram users.
4. **Double-send on stream finalize**: the stream channel sends token updates, then finalization logic posts a second "full" message to the same chat.

## Reliability Pattern

### 1) Stable turn key across the full chain

Create and propagate a strong correlation key:

`conversation_id + message_id + turn_id`

Apply this key consistently at:

- stream start,
- tool-call continuation,
- finalization phase,
- retry and fallback paths.

### 2) Single-message streaming contract (no final duplicate post)

For each turn, enforce one of these contracts only:

- **Streaming mode (recommended for Telegram):**
  - create one placeholder assistant message,
  - update/edit that same message as tokens arrive,
  - mark completion by final edit/state update,
  - **do not send a second aggregated final message**.
- **Non-stream mode:**
  - send only one final message once completion is reached.

Implementation guardrails:

- Persist `delivery_mode` (`streaming` or `final_only`) in turn context.
- Persist `output_message_id` for streamed turns.
- In finalization, branch by `delivery_mode`:
  - `streaming` → `editMessage(output_message_id, final_content)` only.
  - `final_only` → `sendMessage(final_content)` only.
- Block cross-path fallthrough with explicit assertions/logging.

### 3) Dedup only on strong uniqueness

- Replace coarse dedup with strong-key dedup.
- Include `phase` in dedup scope (`stream_chunk`, `stream_finalize`, `fallback`) to avoid accidental cross-phase replay.
- Keep idempotency guards on sender side.

### 4) Final-send retry and fallback

Recommended behavior:

1. Primary send/edit attempt.
2. Exponential backoff retries (configurable count and interval).
3. If all retries fail, send a standardized fallback text with `trace_id`.

Example fallback message:

> "Your request was processed, but delivery failed. Please retry once. Trace: `<trace_id>`"

### 5) Observability events

Emit lifecycle events per turn:

- `reply_started`
- `reply_stream_chunk`
- `reply_stream_finalized`
- `reply_sent`
- `reply_failed`
- `reply_fallback_sent`
- `reply_duplicate_blocked`

Include:

- latency and retry count,
- `delivery_mode`,
- `output_message_id`,
- target conversation snapshot,
- error category,
- `trace_id`.

## Validation Checklist

- Long-running task + `/reset`: user still gets deterministic terminal status.
- Concurrent Telegram turns: no silent drop under overlap.
- Streaming turns: exactly one visible assistant message per turn (edited in place).
- Injected sender failures: retries + fallback fire deterministically.
- Normal short turns: no regression in latency.

## Rollout & Risk Controls

- Gate new dedup/retry/single-message-streaming behavior behind feature flags.
- Add alerting on:
  - duplicate-send rate,
  - fallback-send rate,
  - "more than one output message per turn" anomaly.
- Define auto-rollback thresholds before production rollout.

## Why it matters

For Telegram-heavy OpenClaw setups, trust depends on clear delivery semantics. A robust design must guarantee both:

- no silent drops, and
- no duplicated final resend after streaming.

Users should consistently experience one coherent assistant message per turn, from first token to completion.
