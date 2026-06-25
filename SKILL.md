---
name: rsi-maker-loop
description: >-
  Run an autonomous, self-improving development loop on a codebase, driven by a
  BACKLOG.md and kept reliable over long horizons with MAKER-style error
  correction — extreme decomposition into micro-steps plus multi-agent voting,
  with the repo's own tests/typecheck/build as the primary judge. Use this
  whenever the user wants an agent to keep building or maintaining a project on
  its own: "autonomous loop", "RSI loop", "MAKER loop", "self-improving agent",
  "perpetual development", "keep working through my backlog", or any long-horizon,
  many-step task where the work must not drift or accumulate errors across
  iterations. Prefer this over a single ad-hoc edit when reliability across many
  dependent steps matters more than one quick change.
---

# RSI MAKER Loop

A perpetual, self-correcting development loop. It turns a backlog into shipped,
verified changes — one reliable micro-step at a time — and regenerates its own
backlog when empty, so it can run indefinitely.

It combines two ideas:

- **RSI loop** — a self-paced loop that, each iteration, advances a prioritized
  `BACKLOG.md` and re-schedules itself.
- **MAKER** (massively decomposed agentic processes) — drive the per-step error
  rate toward zero by decomposing work into tiny, checkable micro-steps and
  applying **error correction at every step** via redundancy + voting. This is
  what lets a long chain of steps run without derailing.

## The one idea that makes this work

A naive autonomous loop fails over long horizons because errors **accumulate**:
each big single-agent step has a real error rate, and over hundreds of steps the
process derails. MAKER fixes this by making each step (a) small enough to be
checkable and (b) **error-corrected before you build on it**.

In software you usually have a *ground-truth checker for free*: tests, the type
checker, the compiler, a linter. So the strongest, cheapest "vote" is often a
deterministic check. Use voting by independent agents only where no such check
exists (design choices, naming, API shape, "is this minimal?"). This keeps cost
sane while still catching errors at every step.

> **Honest boundary — read this.** MAKER bounds **execution** errors (is this
> diff correct, does it pass the checks), not **strategic** drift (is this the
> right thing to build). There is no ground truth for "is the product better."
> So this loop gives high reliability on *doing the step right*, while *what to
> do* stays a judgment call. That is why direction needs a value judge and
> periodic human checkpoints (below), not just voting.

## Setup (first run)

On the first iteration, establish the operating contract. Resolve it in this
order:

1. If `.rsi-maker-loop.json` exists at the repo root, use it (don't ask again).
2. **Else, if a human is present (interactive session): ask 2–3 quick setup
   questions**, then **persist the answers** to `.rsi-maker-loop.json` so you
   never re-ask and any later (e.g. cloud) run inherits them. Ask only the
   decision-changing ones:
   - *Where should changes land?* → `commit_target`: **branch + PR** (review
     first, recommended) or **main**.
   - *Run forever or one pass?* → `perpetual`.
   - *How hard should it try on judgment calls?* → `vote_k` (cost vs care).
   Auto-detect the rest (e.g. whether a git **remote** exists — PRs need one;
   without a remote, "branch + PR" degrades to a local branch).
3. **Else (unattended — no human to answer, e.g. a scheduled cloud run): do NOT
   block on questions.** Use the safe defaults below and record them, so the
   autonomous loop never stalls waiting for input.

Contract fields (defaults shown):

- **commit_target** (default `"branch-pr"`): `"branch-pr"` works on a dedicated
  branch and opens a PR for human review; `"main"` commits/pushes to the default
  branch behind the test gate. Default to `branch-pr` — unattended, unreviewed
  changes to `main` are how autonomous loops cause damage.
- **vote_k** (default `3`): how many independent candidate agents to run for an
  **unverifiable** step. Verifiable steps don't need K — the check is the judge.
- **perpetual** (default `true`): self-schedule the next iteration when done.
- **max_blast_radius** (default: small): files/lines a single iteration may
  touch before it must stop and ask. Keeps a bad iteration from sprawling.

If there is no `BACKLOG.md`, create one (see `references/backlog.md`) by
interviewing the user briefly about the goal, then seed P0–P3 items.

## The loop (one iteration)

Do exactly one iteration per invocation, then perpetuate (or stop).

1. **Intake.** Read `BACKLOG.md`. If the user has reported a bug or there are
   open issues addressed to the loop, those take priority over backlog items.
   Otherwise take the first unchecked item by priority (P0 → P3).

2. **Decompose** the item into an ordered list of **micro-steps**, each with a
   concrete success criterion. A good micro-step is small enough that a focused
   agent rarely gets it wrong and the result is *checkable* — ideally by a test,
   type check, or a one-line assertion. If a step isn't checkable, say how you'll
   judge it (majority vote / judge agent). See `references/voting.md`.

3. **Execute each micro-step with error correction:**
   - **Verifiable step** (a deterministic check exists): produce a candidate,
     run the check (`run the repo's tests / typecheck / build` for the touched
     area). Accept the first candidate that passes. If none passes after a small
     number of tries, re-decompose the step or escalate.
   - **Unverifiable step** (design/judgment): **first gate on consequence** — if
     the choice is obvious and low‑stakes, just decide it directly (no ensemble);
     spinning up K agents to ratify a trivial call only burns budget. Otherwise
     spawn **`vote_k` independent sub-agents** with deliberately *diverse* framings
     (see "Make votes independent"), collect their candidate outputs, and accept
     the **majority** — or, if there's no majority, have a judge agent pick with a
     stated reason.
   - Never build the next step on an unaccepted step.

4. **Compose & gate.** Assemble the accepted steps. Run the **full** quality gate
   the repo defines (its complete test suite + typecheck + build). Commit **only
   if everything is green.** A red gate means: fix or revert — never commit red.

5. **Record.** Check off the item in `BACKLOG.md`, add any follow-ups you
   discovered, and update docs. Add a test for genuinely new logic — future
   iterations rely on the gate to catch regressions.

6. **Commit** per `commit_target`. Write a message that names the backlog item.
   Don't commit session/scheduler state or scratch files.

7. **Regenerate (RSI).** If no unchecked items remain anywhere, generate a new
   **small batch (3–5)** of high-value items aimed at real gaps — not speculative
   features. Append them, commit the plan, and **surface it to the user before
   going deep**: direction is theirs. See `references/backlog.md`.

8. **Perpetuate.** If `perpetual`, schedule the next iteration (e.g. via a
   self-paced wakeup / the host's loop mechanism) with a prompt that re-enters
   this skill. Otherwise stop and report.

## Make votes actually independent

Voting only corrects errors if the voters fail *independently*. K agents with
the same model, same prompt, and same blind spot will confidently vote for the
same wrong answer. To get real independence:

- give each voter a **different framing/persona** (e.g. "optimize for
  minimalism", "optimize for correctness on edge cases", "optimize for matching
  existing conventions");
- vary temperature;
- where the host supports it, mix **different model families** for the voters.

One voter should always carry a **minimality mandate** ("prefer the smallest
change that works; reject speculative abstraction") — this is what keeps an
infinite loop from gold-plating itself into bloat.

## Cost control (important)

K-way redundancy multiplies cost. Spend it where it buys reliability:

- Steps with a deterministic check: **no voting** — generate and verify.
- Reserve `vote_k` ensembles for unverifiable, **high-consequence** steps
  (irreversible changes, security/crypto, public API shape).
- Keep iterations small (`max_blast_radius`) so a wrong turn is cheap to redo.

## Direction guard (anti-drift)

Because MAKER doesn't bound *strategic* drift, protect direction explicitly:

- Always prefer user-reported bugs/requests over self-generated items.
- At each backlog **regeneration**, pause for a human checkpoint (notify with the
  proposed batch; proceed on approval, or after a stated grace period if the
  user has opted into full autonomy).
- Keep a minimality voter in every unverifiable decision.

## Honest ceilings

- **Execution vs direction:** reliability applies to *doing steps right*, not to
  *choosing the right work*. Don't oversell "zero errors" for open-ended product
  work.
- **Independence is partial:** same-model voters share blind spots; diversify or
  the voting gain shrinks.
- **Decomposition has limits:** great for long, mechanical, checkable work; weak
  for genuinely novel architecture that must be designed holistically. When a
  task won't decompose into checkable atoms, say so and do it as a normal,
  human-reviewed change instead of faking micro-steps.
- **Cost is real:** unbounded perpetual + heavy voting burns budget. The defaults
  above keep it bounded; tune for your wallet.

## References

- `references/voting.md` — the per-step decomposition + voting/verification
  protocol in detail, with examples of verifiable vs unverifiable steps.
- `references/backlog.md` — `BACKLOG.md` format, priorities, the regeneration
  rule, and a starter template.
