# Plan-harden loop

An agent skill for Claude Code. After you write an implementation plan, an independent reviewer red-teams it and the plan is revised until a pass finds nothing above low severity. The loop is anchored to a **ratified requirements** block taken from what you actually asked for, so review does not quietly replace your priorities with "general quality."

Two skills ship together:

| Skill | Role |
|---|---|
| `plan-harden-loop` | Drives the loop. Runs automatically after you create or substantially revise a multi-step plan, and on `/plan-harden-loop`. |
| `plan-harden` | One red-team pass. The loop calls it. You can also run `/plan-harden` for a single pass. |

## Install

In Claude Code:

```
/plugin marketplace add Joshua-Scarle/Claude-code-plan-harden-loop-skill
/plugin install plan-harden-loop@plan-harden-loop
```

## What it does

1. Writes down your stated goals as `## Ratified requirements` and grounds load-bearing claims in the code.
2. Sends the plan to a reviewer subagent that did not write it.
3. That reviewer edits the plan to fix issues above low severity. A fresh reviewer then checks the edit for regressions.
4. Repeats, up to 8 passes, with a step-back at passes 2, 4, 6, and 8. A better approach is shown to you. It is not swapped in silently.
5. Stops when a pass finds nothing above low, or when the cap is hit. A cap hit waits for you. It does not ship the plan.

A light pass (one review, less bookkeeping) runs only when you explicitly ask for it.

## License

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
