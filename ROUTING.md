# Delegation & Model Routing

Written for the orchestrator — usually Opus 4.8 running the main loop, with Agent-tool access to subagents on Opus 4.8, Sonnet 5, and Haiku. "Peer tier" always means whatever model the orchestrator itself is running; if that's Fable, peer tier is Fable. The tiers — mechanical, workhorse, peer — are the doctrine; model names are just the current lineup. When the lineup changes, remap names onto tiers.

The router optimizes one number: **total cost to verified-done** — subagent tokens plus your review time plus retries plus the confusion a wrong result injects into your context. Per-token price is the small term. A cheap wrong answer that costs three review cycles is more expensive than one peer-tier call that comes back right.

## Decision 1 — Whether to delegate

**The default answer is no.** Solo work has zero handoff cost; every delegation starts in deficit — a complete prompt to write, a result to review, a retry to risk — and must pay for itself before it produces anything. There are exactly four triggers that justify it. If you can't name which one applies in a single sentence, work solo.

Delegate when at least one of these is true:

- **Context protection.** The work produces bulk you don't need to keep: reading many files to answer one question, broad searches, build/log/test output. The orchestrator's context is its scarcest resource — keep the conclusion, delegate the reading.
- **Parallelism.** Independent items with no shared state. Fan them out simultaneously in one message; a serial chain of independent delegations wastes wall-clock for nothing.
- **Fresh eyes.** Verification, review, and refutation work better *without* your context — a subagent can't inherit the bias of the reasoning that produced the answer. Delegating an adversarial check is a feature, not a cost.
- **Isolation.** Mass edits or risky experiments go to a worktree-isolated agent so failure doesn't contaminate the working tree.

Do it yourself when:

- It's a single lookup and you already know the file or symbol.
- The work needs the full conversation context. (Exception: fork yourself — a fork inherits your context and keeps its tool noise out of it. Use a fork when the subtask needs everything you know but would flood you with output.)
- Writing a complete delegation prompt would cost more than doing the job.
- It's sequential edits on state you're already holding — round-tripping it through an agent adds handoff error for no gain.

Never delegate, at any tier:

- The final Gate 4 verification of the whole task. Subagents verify pieces; you verify the whole, because only you hold the original request and the standing rules.
- The definition of done. Subagents execute a spec; they don't get to decide what the user meant.
- A judgment call the user delegated to *you* specifically (a design choice they asked your opinion on).

### Not reasons to orchestrate

Each of these feels like a trigger and isn't:

- **The task is big.** Big-and-sequential is solo work with a plan. Size justifies delegation only when it decomposes into genuinely independent pieces (parallelism) or into bulk reading you don't need to hold (context protection).
- **The task is important.** Importance raises the verification bar, not the agent count. Adding agents without a trigger adds handoff-error surface exactly where you can least afford it.
- **The stages are conceptually separate.** Separate ≠ independent. If stage B consumes stage A's output, delegating them separately adds two handoffs to what is still a sequential job.
- **Wanting to look thorough.** Thoroughness is measured at Gate 4 by what you verified, not by how many agents you spawned. A fleet returning unverified claims is less thorough than one solo pass you checked.
- **It worked last time.** The trigger has to hold for *this* task.

### The minimum-orchestration ladder

The right amount of orchestration is the minimum that solves the problem — the same rule as minimum code. Escalate one rung at a time and stop at the first that works:

solo → solo + a script (repetition) → one subagent → parallel fan-out → multi-stage pipeline

Each rung must be justified by a named trigger the previous rung couldn't satisfy. Jumping straight to a fleet because the task "feels like a fleet task" is the routing equivalent of speculative architecture.

## Decision 2 — Which model

Route by **spec completeness** and **cost of a wrong answer** — never by task size. A huge mechanical job routes down; a three-line judgment call routes up.

**The down-routing test, before any tier choice:** route below peer tier only when the quality ceiling is fixed — the spec or an existing pattern already determines what good looks like, so a weaker model can't quietly lower it; it can only fail visibly, and your check will catch that. The moment the subagent's own choices can shape how good the outcome is — design, abstraction, naming, trade-offs — the ceiling moves with the model: stay at peer tier. Plausible-but-sub-optimal is the outcome this test exists to prevent; it compiles, passes the happy path, and bills you at maintenance time.

### Haiku — mechanical tier

Use when the spec is complete, the output is cheaply verifiable, and there are zero gaps for the model to fill. Treat the mechanical tier as a literal executor: assume any gap in the spec gets filled silently rather than flagged — so leave none.

- Well-specified transformations and format conversions.
- Run a command, summarize its output against stated criteria.
- Sweeps with exact match criteria ("list every file importing X").
- Extraction into a defined schema from known-shape input.

Test: *would a careful intern with a checklist succeed?* If success requires taste, it's not a Haiku task.

### Sonnet 5 — workhorse tier (the default once the down-routing test passes)

Use for judgment inside a defined frame: the goal is clear, the approach is roughly known, and reasonable relevance calls are required along the way.

- Exploration and search that needs judgment about what's relevant ("find where auth failures are handled").
- Scoped implementation with clear requirements and existing patterns to imitate.
- First-pass code review, research, multi-source summarization.
- Writing tests against a defined behavior.

When in doubt between Haiku and Sonnet, pick Sonnet — the retry you save pays for it. When in doubt between Sonnet and peer tier, pick peer tier: doubt about the tier is doubt about whether the quality ceiling is fixed, and that doubt answers itself.

### Opus 4.8 — peer tier

Use when the subagent must exercise the judgment you would exercise yourself:

- Debugging with ambiguous symptoms, where the first theory is probably wrong.
- Architecture and design decisions; anything security-sensitive.
- Adversarial verification of findings you will act on. The refuter must be at least as strong as whatever produced the claim, or refutation is theater.
- Tasks where a subtly-wrong answer is worse than no answer — plausible-but-wrong survives review; obviously-wrong doesn't.
- Plan generation for multi-step work.
- Any retry after a lower tier failed verification once.

### The verifiability override

This trumps the tier descriptions above, in both directions:

- If you can verify the output **completely and cheaply** (a checkable answer, existing tests, a small diff you'll read anyway), route **one tier down**. A wrong answer costs one retry, and you'll catch it. Completely means correctness *and* quality — if assessing the design quality of the result would mean re-deriving the design yourself, it is not cheaply verifiable.
- If verifying the output means **redoing the work**, route **up**. You're buying trust you can't afford to audit — buy it from the strongest tier.

### The ambiguity rule

Ambiguity is the silent killer at low tiers: Haiku and Sonnet resolve it plausibly and without flagging it, and plausible-but-unflagged is exactly what slips past review. Two legal moves: resolve the ambiguity in the prompt yourself (then route down), or keep the ambiguity and route up to a tier that will surface it. Never send an ambiguous spec to Haiku.

A requirement-only spec — what to build, not how — is ambiguity by design: it delegates the design decisions to the implementer, which is exactly where tier differences concentrate. Upgrade it to a design spec (interfaces, invariants, edge cases, what not to build) before routing down, or route the implementation to peer tier.

## The delegation prompt contract

A subagent knows nothing you don't tell it (forks excepted). Every delegation prompt carries five things:

1. **Context it can't see** — why this matters, constraints already discovered, approaches already tried and why they failed. Omitting failed attempts guarantees the subagent repeats them.
2. **Done, precisely** — the exact deliverable and the check it must pass. Gate 1 applies to the subagent's task too; if you can't write its done-check, you're not ready to delegate it.
3. **Output contract** — demand raw material: file paths, line numbers, verbatim snippets, counts, exit codes. Its final message is your tool result; narrative prose without specifics is a report you can't verify (Gate 4) and can't act on.
4. **Boundaries** — what not to touch, what not to decide, when to stop and report back instead of improvising.
5. **Scope partitions** (parallel fan-outs only) — non-overlapping assignments, or two agents silently do the same work and a third does none.

## Escalation ladder

- Output fails your check → fix the **prompt** and retry once at the same tier. First failures are usually the spec's fault, not the model's; escalating a bad prompt just buys you a more articulate wrong answer.
- Second failure → escalate one tier. Never re-roll the same tier on the same prompt — that's Gate 3's rule (two failed attempts = wrong diagnosis) applied to routing, and here the wrong diagnosis is usually "this was ever a Haiku/Sonnet task."
- Failure at peer tier → the problem isn't the model, it's the decomposition. Return to Gate 1 and re-slice the task; don't shop the same broken spec around.

## Trust but verify — always

Nothing a subagent returns is true because it says so, at any tier. Apply Gate 4 to agent output exactly as to your own work: spot-check at the layer of the claim, sample the tails of anything enumerated, and treat a surprisingly clean result as a broken check until you can explain why it's real. The tier you routed to changes how much you sample, not whether you sample.

When the stakes justify a full judge pass — work you'll build on, ship, or report upward — run it as a procedure, not a vibe:

1. **The report is a set of claims, not evidence.** List what was supposedly done, supposedly verified, and supposedly left untouched; each claim is a row to prove or refute.
2. **The diff is ground truth.** Establish what actually changed (`git diff`/`git status`, or a directory diff) and compare it against the ask's blast radius before believing a word of the report's narrative.
3. **Re-run every claimed verification.** Don't read code and nod — run the tests, the build, the script, the page, and capture the output. A claim you can't re-run (environment, credentials, human-eyes-only) is labeled UNVERIFIABLE, never assumed true.
4. **Hunt frauds in real-world frequency order:** weakened checks (assertions loosened or deleted, expected values changed to match new behavior, tests skipped, real calls mocked); false completion ("should work", a pass claimed with no run shown); scope creep (changes beyond the ask); spec betrayal (code changed to satisfy a check that contradicts the spec — apply Gate 3's authority order); debris (scratch files, debug prints, commented-out code).
