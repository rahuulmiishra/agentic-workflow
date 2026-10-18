---
name: fix-bug
description: Use when asked to fix a reported bug in a web app from a description, screenshot, or reproduction steps, before touching any code, and whenever a fix must be reviewed and regression-checked before hand-off.
---

# Fix bug

Reproduce before touching code. Verify the clean diff, never the instrumented one. Review and QA both run with learnings loaded, and what they catch goes back into the learnings.

**REQUIRED SUB-SKILL:** map-feature (step 1). Learnings referenced as `L0xx` live in `~/.claude/skills/learnings/`.

## Inputs

Required: task (the bug), feature or route, expected behaviour, reproduction steps, acceptance criteria. Step 0 checks them. Steps given but the bug does not reproduce after the environment check in map-feature: BLOCKED.

## Steps

0. **Check inputs.** Required: task (the bug), feature or route, expected behaviour, reproduction steps, acceptance criteria. A field counts as present only when the user stated it, or it follows from what they stated with no second reading (say how). A missing feature or route that the screen's visible text or the route table names unambiguously counts as present. Any required field missing: do not map, run, or edit anything. If you can ask, ask once, listing every missing field, and wait. As a subagent, end with `BLOCKED: missing inputs: <fields>` and nothing else.
1. **Map and reproduce.** Run map-feature with the bug's reproduction as the target flow. The bug counts as reproduced only when the wrong behaviour has been observed live in the browser and recorded under Findings as **observed**. Nothing after this step starts before that. Once it is, continue to step 2 in the same run: reproducing is the gate, not the deliverable.
2. **Localize.** Walk the map's Flow and find the first step where observed diverges from expected. Name the mechanism at `file:line` with the evidence that shows it (network payload, state value, console line). A fix hypothesis that cannot name the mechanism is not ready. If a fix does not work, do not patch the same theory again: diff against a working sibling flow or an earlier commit (L028).
3. **Fix surgically.** Smallest change at the mechanism, in the repo's existing patterns. No adjacent cleanup, no renames, no formatting. Delete imports or variables the change orphaned. Append the cause and the fix to the map's Findings.
4. **Verify on the clean tree.** Run "Clean verification" from map-feature with the bug's reproduction as the target flow.
5. **Review.** Launch the `local-diff-reviewer` agent with: base ref, the task and acceptance criteria, the map path. It fixes what it finds and returns findings, changes, and "New blind spots". If it changed any code: run `/code-learnings update` with its report, then repeat step 4 in full.
6. **QA.** Run `/learnings qa` to build the QA block. Launch a general-purpose agent with that block plus the reproduction steps, acceptance criteria, environment, and the map path, asking it to verify the fix and look for regressions it classifies introduced / pre-existing-made-reachable / pre-existing-untouched. Any introduced or made-reachable issue: fix it, run `/learnings update`, repeat step 4, rerun QA on the new diff. Do not relay the agent's claims without checking its evidence lines (L015).
7. **Report** in the shape below.

## BLOCKED

Cannot reproduce, cannot start the app, or cannot inspect something the fix depends on. If you can ask the user, ask once with the exact gap. If you are running as a subagent and cannot ask, finish every step that does not depend on the gap, then end with `BLOCKED:` followed by what you tried, the exact limitation, and what you need. Never report verification you did not perform.

## Report

```
Result: fixed | partially fixed | BLOCKED
Root cause: <mechanism, file:line>
Change: <files, one line each>
Verification: <browser observation after cleanup; checks run with pass/fail output>
Review: <findings count, what was changed, learnings recorded>
QA: <outcome; findings by class introduced / pre-existing-made-reachable / pre-existing-untouched>
Limitations: <what could not be verified and why>
Map: <path>
```

## Red flags

- Editing source before the wrong behaviour was observed in the browser
- Fix verified while `[TRACE]` logs or `debugger` statements remain
- Second fix attempt on the same unproven theory
- Reviewer or QA output relayed without re-running step 4
- "Fixed" reported with no browser evidence after cleanup
