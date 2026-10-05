---
name: plan-harden-loop
description: >
  Iteratively red-team and harden a plan you just wrote, before you act on it,
  so review cannot drift off what the user asked for. Use automatically whenever
  you have just created or substantially revised an implementation, refactor, or
  multi-step plan and are about to execute or present it. Ground the plan, then
  drive repeated single hardening passes (the plan-harden skill) until
  convergence — a pass finds nothing above low severity — anchored to the plan's
  ## Ratified requirements. Each pass has the reviewing agent edit the plan
  directly (not the plan author), then runs a fresh adversarial subagent on the
  edited plan to catch edit regressions. FULL is the default: every plan runs
  the full loop at a cap of 8 iterations, a rethink checkpoint every 2
  iterations, and up to 2 extra low-issue cleanup passes. LIGHT is used only
  when the user specifically asks for it — one grounded red-team pass, no
  iteration record. A plan that merely looks routine does not qualify for LIGHT.
  Not for the validation loop that tests already-written code. Also invocable
  via /plan-harden-loop.
---

# plan-harden-loop — drive the hardening loop to convergence

You just created or substantially revised a plan and are about to act on it. Run
this loop first. It is the driver; each **pass** is the **plan-harden** skill.

**Re-run trigger:** any substantive edit to a plan (scope, approach, steps,
acceptance criteria, anything a reviewer's judgment rested on) invalidates the
review before it — loop again. Cosmetic edits (typos, formatting) do not.

## Pre-Step 0 — settle user-owned forks (before grounding / tier / red-team)

If the plan still has **unsettled user-owned forks** (product vision, preferences,
non-objective architecture, scope or priority the user owns), ask the user and
wait for an answer **before** Step 0 grounding, Step 0.5 tier pick, or any
red-team. Do not harden fake certainty.

## Step 0 — ground the plan (before any red-team)

1. **`## Ratified requirements` anchor.** Ensure the plan carries this block:
   the user's stated goals and any explicit priority ranking — **verbatim where
   the user gave priority language**, faithful paraphrase otherwise, **no reviewer
   editorializing**. This is the fixed invariant the loop preserves and every
   reviewer is anchored to. Only the user may change it. A genuinely mechanical
   plan escapes with `RATIFIED-REQS: N/A (single obvious objective — <one line>)`.
2. **Existing grounding.** Confirm the plan has its `## Consumers & rituals
   checked` section and `file:line` citations from source read this session (or
   `GROUNDING: N/A` with a reason). If missing, ground it before red-teaming —
   an under-grounded plan wastes red-team effort.

### Grounding requirements

- Every load-bearing claim about existing code carries a `file:line` citation from
  source read **this session**. Notes and prior summaries may locate a file; only
  live code substantiates a claim. Never write "verified" for a check not actually run.
- **Consumer enumeration:** for every contract, producer, field, or output the plan
  changes, search live code for read-sites (imports, path, column, and function
  references) and record them in a `## Consumers & rituals checked` section of the
  plan file, together with any project invariants you checked. If no code contracts
  are touched, that section says `GROUNDING: N/A (no code contracts changed)` plus
  a one-line reason.
- **Cheap execution probes** where reachability or flow matters (import check,
  schema dump, run the function on one input). Reading alone misses dead branches
  and unreached fix-sites.
- **Acceptance criteria must be falsifiable independently of the plan's assumptions:**
  wired-path tests, not isolated helpers; never hand-authored expected values that
  mirror the change.
- **Fragment coherence:** a plan assembled from multiple design passes gets one
  dedicated cross-task coherence read-through (vocabulary, sequencing, mutual
  test-breakage) before red-team.

Markers prove that sections exist. They do not prove the sections are honest.
Use `GROUNDING: N/A` and `PLAN-HARDEN: N/A`, each with a one-line reason, only
when they genuinely apply.

## Step 0.5 — pick the tier: LIGHT or FULL

**FULL is the default — every plan runs the FULL loop.** **LIGHT only when the
user specifically asks for it.** Looking routine is not a reason to choose LIGHT.

**LIGHT** declares `PLAN-HARDEN: LIGHT (<one-line reason>)` **on its own line** in
the plan file (a mid-sentence mention waives nothing), and then:

- **Still does** Step 0 grounding in full, and **one real red-team pass** (the
  `plan-harden` skill). Whether the plan's facts are true is independent of how
  strong the authoring model is — grounding is never relaxed, and neither is the
  independent review.
- **Skips** the bookkeeping: the `## Plan-harden loop` iteration record, the
  `## Ratified requirements` anchor, and the Step N reconciliation.
- **Converges at that one pass** unless the pass turns up something above `low`,
  in which case fix it and re-run, or escalate to FULL if the finding shows the
  plan was not a light review after all.

Without an explicit user ask, run FULL.

## Reviewer-direct edit + post-edit verification

Each hardening pass **splits finding from fixing**. The original plan author does
**not** apply red-team fixes — the reviewing agent does, because it understood the
issues in context and is less likely to misread findings than a separate author
relaying them.

Per pass (detail in **plan-harden**):

1. **Red-team.** An independent reviewer subagent that did **not** write the plan
   finds issues; triage by severity. When the plan crosses a documented
   cross-module contract, touches safety-critical logic, or was assembled from
   multiple design passes, also run a **second** fresh reviewer so the review is
   not a single channel.
2. **The reviewing agent edits the plan directly** — resolve everything above `low`
   in the plan file. When an independent reviewer subagent ran, **that same
   subagent** performs the edit. If it cannot edit, spawn a dedicated
   **plan-editor** subagent handed the findings and the ratified-requirements
   anchor.
3. **Fresh adversarial verification** — spawn a **new** adversarial subagent that
   did **not** write the original plan and did **not** perform the edit. It
   red-teams the **edited** plan to catch regressions and new issues introduced
   by the fix. Triage its findings before recording the iteration.

### Anti-regression warning for the editing agent (mandatory)

Hand every editing agent that will modify the plan this warning **verbatim**:

> You are editing the plan to fix identified issues ONLY. Do not refactor, reorder
> scope, add "improvements" to unrelated sections, or rewrite passages that were
> not flagged. Prefer the **smallest change** that resolves each finding. Every
> unnecessary edit risks introducing new problems. Preserve the `## Ratified
> requirements` block unchanged. Park any spec-touching deviation instead of
> editing it in.

## The loop

Default params (all per-task overridable by the user): **cap = 8**, above-`low`
tolerance, **low-cleanup = 2**, **rethink at iterations 2, 4, 6, 8**. This cap
of 8 is for the plan. A separate loop that tests written code is a different
procedure and is not this skill. One loop with an explicit `cleanup_remaining`
counter and a **re-classify after every pass** step, so an edit or cleanup fix
that introduces a new above-`low` issue cannot slip through the convergence gate:

```
cleanup_remaining = null           # null = low-cleanup not yet entered
for i in 1..cap:
  run plan-harden pass i           # red-team -> triage -> reviewer edits plan
                                   # -> fresh adversarial verify -> triage -> record
  record Iteration i               # via plan-harden, + priority-fidelity verdict
  if i in {2,4,6,8}: RETHINK       # see below; record the Rethink line
  RE-CLASSIFY all remaining issues, INCLUDING any from post-edit verification
  or introduced by this pass's edits:
    if any issue ABOVE low remains:
        cleanup_remaining = null    # a regression/new above-low cancels cleanup
        continue                    # back to main hardening
    else:                           # nothing above low
        # an early converged-exit must NOT fire while a parked anchor deviation or
        # a rethink 'better-option' is awaiting user sign-off — route those first.
        if parked_deviation_pending or better_option_pending:
            convergence = 'raised to user'; break
        if cleanup_remaining == null: cleanup_remaining = 2   # enter low-cleanup
        if no low issues remain:      converged; break
        if cleanup_remaining == 0:    converged (accept residual lows); break
        cleanup_remaining -= 1; continue
# cap reached without an early exit — mandatory user halt (never auto-ship):
convergence = 'raised to user — cap hit at iteration <cap>'
STOP: ask the user what to do (see Cap hit below). Do NOT present the plan as
final, or execute it, until they choose — even if re-classification shows only
residual low issues and nothing above low.
```

`no issues above low` is emitted ONLY when a *re-classified* pass confirms nothing
above `low` — never merely because the cleanup budget ran out. Rethink fires even
during low-cleanup.

## Rethink checkpoint (iterations 2, 4, 6, 8)

"Rethink" means **step back and re-evaluate optimality. Do not auto-switch
approaches.** At the end of an even iteration:

- Surface and challenge the plan's load-bearing **assumptions**.
- Weigh the current approach against realistic **alternatives**, judged on the
  **ratified requirements**.
- Conclude one of:
  - *confirmed-best* → continue the loop.
  - *better-option-found* → **surface it to the user; never silently switch.** A
    better option is a user decision, exactly like a spec deviation.
- Record a line: `Rethink @ iter <n>: confirmed-best` or
  `Rethink @ iter <n>: better-option — <what>`.

## Park, never absorb

A reviewer proposal that deviates from the `## Ratified requirements` anchor
(re-rank, change what is primary, add, drop, or re-emphasize a requirement) is
**parked, not absorbed**: record it, keep hardening the original ratified
objective, and batch it to the reconciliation below. Surface mid-loop only if a
parked item blocks further useful hardening. A permissive permission mode does
not turn a ratified-requirement deviation into an absorbable adjustment.

## Step N — final reconciliation before presenting or executing

Before you present the plan or execute an inline one, write a
`## Requirements reconciliation` section: map each ratified requirement and its
rank to how the final plan serves it (or `changed/dropped — approved by user on
<ref>`), and list every parked spec-touching item. This catches slow drift no
single pass flagged. A `RATIFIED-REQS: N/A` plan has nothing to reconcile — skip
it.

## Cap hit — mandatory halt (do not move on)

When the iteration **cap** is reached **without** an early converged exit (the
loop exhausted all `cap` passes), **stop** and **ask the user what to do**. This
applies **even when** the last re-classification finds nothing above `low` —
residual `low` issues at cap are not silently accepted, and the plan is not
treated as done.

Until the user answers:

- Do not present the plan as final or begin execution.
- Record convergence as **`raised to user — cap hit at iteration <n>`** (include
  a one-line summary of any above-`low` issues still open and any residual `low`s
  you would otherwise have accepted).

When you ask, give each option what / pros / cons. Mark **(Recommended)** on
continue hardening only if above-`low` issues remain — otherwise recommend
explicitly on accept versus change approach. Typical menu:

1. **Continue hardening** — user raises the cap (or resets tolerance); resume the loop.
2. **Accept current plan** — user explicitly accepts residual issues (including lows);
   only then may convergence become `no issues above low` (note accepted residual lows)
   and you may exit or execute.
3. **Change approach** — user directs a substantive replan; invalidate prior review
   and loop again from Step 0 as needed.

Silence is not a choice. Wait for a real answer.

## Raise to the user — never silently absorb

- **Cap hit** (always — see above).
- Any parked spec-touching deviation, and any rethink `better-option`.
- Any significant change the loop forced, or issue caught, that materially affects
  scope, approach, or outcome — especially a fix that would deviate from agreed specs.
  Present options (what it is / pros / cons) and your recommendation.

## Exit

- **Converge** when a re-classified pass finds nothing above `low` and no
  escalation is pending.
- **Cap = 8.** If the cap is hit without an early converged exit, do not keep
  hardening silently, do not auto-accept residual lows, and do not ship — **halt
  and ask the user** (continue with a raised cap / accept the current plan /
  change approach). The user may override cap or tolerance per task.

The plan file's `## Plan-harden loop` record (written by each plan-harden pass)
plus the `## Requirements reconciliation` are the loop's written record. Emit the
locked vocabulary verbatim (see plan-harden's contract table). Record only passes
you actually ran. Never inflate the iteration count.
