---
name: plan-harden
description: >
  Runs ONE hardening / red-team pass over a plan. An independent reviewer
  subagent that did not write the plan red-teams it, anchored to the plan's
  ## Ratified requirements. Triage every issue by severity, have the reviewing
  agent edit the plan directly (not the plan author), run a fresh adversarial
  subagent on the edited plan, and record the iteration in the plan file using
  the locked vocabulary. Invoked repeatedly by the plan-harden-loop skill, or
  directly via /plan-harden for a single standalone pass. Before any pass,
  unsettled user-owned forks are settled with the user first. This is ONE pass
  only — it does not drive the full loop and does not auto-fire when you create
  a plan (that is plan-harden-loop's job). Use it directly only when you want a
  single extra red-team pass on an existing plan.
---

# plan-harden — one hardening pass

You are running **exactly one** red-team/harden pass over a plan file. The
iterative loop (cap, rethink checkpoints, convergence, low-issue cleanup) is the
**plan-harden-loop** skill's job — do not re-implement it here. When invoked by
that loop you are told the current **iteration number** and handed the plan's
**ratified-requirements anchor**. When invoked directly via `/plan-harden` you
are iteration 1 of a standalone pass.

## The one pass — do these in order

1. **Anchor the reviewers to the ratified requirements.** Read the plan's
   `## Ratified requirements` block (the invariant the plan must preserve — see
   plan-harden-loop for how it is built). Every reviewer you spawn this pass is
   handed that block explicitly as *the spec to preserve*, and asked to report,
   **separately from general-quality findings**, whether the plan still honors
   each ratified requirement and its rank. Without that anchor, reviewers
   optimize a proxy ("general quality") and the loop amplifies drift away from
   what the user actually ranked first. If the plan legitimately has no ratified
   requirements it carries `RATIFIED-REQS: N/A` — then skip the priority-fidelity
   parts of this pass.

2. **Red-team adversarially.**
   - Spawn an **independent reviewer subagent that did not write the plan**. Ask
     it to hunt gaps, wrong assumptions, missed dependencies and consumers, spec
     deviations, and failure modes (not cosmetic nits), and to give the
     priority-fidelity read against the anchor.
   - **When the plan is consequential** — it crosses a documented cross-module
     contract, touches safety-critical logic, or was assembled from multiple
     design passes — ALSO spawn a **second fresh reviewer** (one that did not
     write the plan and is not the first reviewer) so the review is not a single
     channel. Note in the record whether this pass had one independent channel
     or two.
   - For plans carrying code-fact citations, a bounded **citation-verifier**
     subagent ("verify these `file:line` claims against their cited files: true
     or false each; quote the disproving line if false") is the cheap routine
     check.

3. **Triage every issue by severity** — `low` / `medium` / `high` / `critical`.

4. **The reviewing agent edits the plan directly** — not the original plan author.
   The agent that understood the findings applies fixes in the plan file:
   - When an **independent reviewer** subagent ran in step 2, **that same
     subagent** performs the edit.
   - If that reviewer cannot edit the file, spawn a dedicated **plan-editor**
     subagent handed the findings and the ratified-requirements anchor.
   - **Hand the editor the anti-regression warning verbatim** (from
     plan-harden-loop): fix identified issues only; no refactors, scope reorder,
     or unrelated "improvements"; prefer the **smallest change** per finding;
     every unnecessary edit risks new problems; preserve `## Ratified requirements`
     unchanged.
   - Resolve everything above `low` (record any `low`s left for the loop's cleanup
     phase).
   - **A ratified-requirement deviation is parked, never absorbed.** If a
     reviewer proposes anything that would deviate from the anchor — re-rank,
     change what is primary, or add, drop, or re-emphasize a recorded
     requirement — do not fold it into the plan as "hardening". Record it as a
     parked, spec-touching item and keep hardening the original ratified
     objective. Surface immediately only if a parked item blocks further useful
     hardening. Only the user may change the `## Ratified requirements` block.

5. **Fresh adversarial verification of the edited plan.** Spawn a **new**
   adversarial subagent that did **not** write the original plan and did **not**
   perform the edit in step 4. Anchor it to the ratified requirements. Ask it to
   red-team the **post-edit** plan for gaps, wrong assumptions, missed
   dependencies and consumers, failure modes, priority-fidelity drift, and —
   especially — **regressions or new issues introduced by the edit**. Triage its
   findings by severity. Do **not** have the step-4 editor fix these in the same
   pass — they feed the loop's re-classify and the next iteration if still above
   `low`.

6. **Record this iteration in the plan file** under a `## Plan-harden loop`
   section, appending an entry for this pass. **Emit the locked vocabulary
   verbatim** (see the contract below):
   - The section heading `## Plan-harden loop` (once).
   - An entry beginning `**Iteration <n>:**` (or `**Iteration <n>** —`) — always
     the word **Iteration**, a space, then the number. Never `Iteration-<n>`
     (hyphen).
   - Every `**Iteration <n>**` you write must be backed by a real red-team you
     actually ran. Never inflate the count.
   - Under it: each issue from the initial red-team and from post-edit
     verification, its **severity**, how the plan was changed to resolve it (or
     that it was parked), and any edit-regression flagged by post-edit
     verification.
   - A **priority-fidelity verdict** line: whether the ratified items are still
     honored and ranked, naming anything that deviates.
   - The loop owns the **convergence line** (`no issues above low` or
     `raised to user`) and the rethink line — emit those only if you are also
     acting as the loop. A bare `/plan-harden` pass records its findings and
     stops.

## Locked vocabulary

Emit these exact strings so a later reader, or a local check, can find them:

| Write this string | What it marks |
|---|---|
| `## Ratified requirements` | anchor presence |
| `## Consumers & rituals checked` | grounding presence |
| a real `path.ext:line` (e.g. `foo/bar.py:42`) | code-fact citation |
| `## Plan-harden loop` | loop-record heading |
| `**Iteration <n>:**` (Iteration, space, digit) | iteration entry |
| one real red-team per claimed `**Iteration <n>**` | iteration honesty |
| `no issues above low` **or** `raised to user` | convergence line |
| `## Requirements reconciliation` | final reconciliation |
| escapes: `GROUNDING: N/A` · `PLAN-HARDEN: N/A` · `RATIFIED-REQS: N/A` | skip the matching check |
| `PLAN-HARDEN: LIGHT (<reason>)` | user-requested light tier: waives the loop record and the ratified pair, but **not** the red-team pass and **not** grounding |

**All escape and tier markers must be line-anchored:** put the marker on a line of
its own (optionally bulleted, bolded, or backticked — `- **PLAN-HARDEN: LIGHT** (reason)`
and `**PLAN-HARDEN: LIGHT** (reason)` both work). **Not as a heading** — `## PLAN-HARDEN: LIGHT`
does not waive, so a plan whose section is merely titled after a marker cannot
disarm a check. A mid-sentence mention waives nothing either.

## Scope reminder

One pass. Hand control back after recording. Do not decide convergence, do not
advance the iteration counter, do not run the rethink checkpoint — those belong
to **plan-harden-loop**. If you were invoked directly and the user wants the full
iterative loop, tell them to use plan-harden-loop (or `/plan-harden-loop`).
