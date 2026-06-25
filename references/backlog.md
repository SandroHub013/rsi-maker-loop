# BACKLOG.md — format, priorities, regeneration

The backlog is the loop's queue and memory. It must be resumable: any fresh
agent should be able to read it and know exactly what to do next.

## Priorities

- **P0 — reliability & correctness**: it must work and not lose data. Bugs,
  persistence, error handling, flaky-test fixes.
- **P1 — architecture that unblocks scale/quality**: interfaces, seams, test
  harnesses, performance foundations. Build the abstraction now; the heavy
  cloud/infra implementation can come later behind it.
- **P2 — features toward the goal**: user-facing capability.
- **P3 — hardening & polish**: security passes, a11y, docs, CI, refactors that
  pay for themselves.

Within a priority, order by value/effort. The loop always takes the first
unchecked item top-down.

## Item format

Use plain Markdown checkboxes so progress is visible in any viewer:

```markdown
## P0 — reliability & correctness
- [ ] Persist the work queue across restarts (swappable store; local impl now).
  - [ ] follow-up: flush on shutdown.
- [x] Dedup incoming events by id.
```

Keep items **small enough to finish in one iteration**. If an item is really an
epic, split it; the loop works best on checkable chunks.

## Regeneration rule (the "RSI" part)

When **no unchecked item remains anywhere** in the file:

1. Generate a **small batch of 3–5** new items — the highest-value real gaps
   toward the project's goal (reliability, performance, security, UX), **not**
   speculative features. Apply the minimality bar from `voting.md`.
2. Prioritize them P0–P3 and append.
3. Commit the new plan on its own.
4. **Surface the batch to the user and pause for a checkpoint before going deep.**
   Direction is the user's call; the loop's autonomy is on *execution*, not on
   *what the product should become*.

This is what makes the loop perpetual without letting it wander.

## Starter template

```markdown
# <project> — Backlog (autonomous loop)

Queue for the rsi-maker-loop. Goal: <one sentence>.

## Loop rules (every iteration)
1. User-reported bugs/requests first.
2. Else: first unchecked item, P0→P3.
3. Minimal correct change; reuse existing modules; no new deps if a few lines do.
4. Commit only if the full gate (tests + typecheck + build) is green; add a test
   for new logic.
5. Update this file (check off + follow-ups) and docs.
6. Commit per the configured target (default: branch + PR).
7. If the backlog is empty: regenerate a 3–5 item batch and surface it for review.

## P0 — reliability & correctness
- [ ] ...

## P1 — architecture
- [ ] ...

## P2 — features
- [ ] ...

## P3 — hardening
- [ ] ...
```
