---
name: build-feature
description: Use when asked to build, add, extend, or change a feature or behaviour in a web app from a spec, ticket, or description, before writing implementation code, and whenever the result must be compared against the deployed app and regression-checked before hand-off.
---

# Build feature

Map first. Declare every intended behaviour change before writing code. The deployed app is the oracle for everything not on that list.

**REQUIRED SUB-SKILL:** map-feature (step 1 and Clean verification). Learnings referenced as `L0xx` live in `~/.claude/skills/learnings/`.

## Inputs

Required: task, feature or route, expected behaviour, acceptance criteria. Optional: spec or ticket. Step 0 checks them. The oracle URL and its login come from the repo facts in map-feature. No oracle in the repo facts: ask if you can; as a subagent, skip the runtime half of step 5, state that under Limitations, and use the tests plus the intended-changes table as the oracle.

## Steps

0. **Check inputs.** Required: task, feature or route, expected behaviour, acceptance criteria. A field counts as present only when the user stated it, or it follows from what they stated with no second reading (say how). A missing feature or route that the screen's visible text or the route table names unambiguously counts as present. Any required field missing: do not map, run, or edit anything. If you can ask, ask once, listing every missing field, and wait. As a subagent, end with `BLOCKED: missing inputs: <fields>` and nothing else.
1. **Map.** Run map-feature on every existing flow the feature touches. A new feature with no existing flow: map the nearest sibling flow and the entry point where the new one attaches. The current behaviour counts as understood only when every Flow line the task touches has a `file:line` and each touched flow has been run in the browser and recorded as **observed**. Nothing after this step starts before that. Once it is, continue to step 2 in the same run: mapping is the gate, not the deliverable.
2. **Declare changes.** Before any code, fill the map's "Intended behaviour changes" table: every user-visible, network, or state difference the task requires, each with its reason tied to the acceptance criteria. Anything not in the table must stay identical to the oracle. Keep the table current as understanding changes.
3. **Implement.** Existing patterns, smallest diff, no adjacent cleanup. Check the repo's documented rules while writing, not at review time (L019). **REQUIRED SUB-SKILL:** superpowers:test-driven-development wherever the repo has a unit seam for the change. Shared state or contracts with consumers outside this app get those consumers checked before the change lands (L009).
4. **Verify on the clean tree.** Run "Clean verification" from map-feature with the acceptance flows as the target.
5. **Compare against the oracle.** Open each acceptance flow on the oracle URL and locally with the same login, role, test data, and flag state. Replay the same inputs on both, including a few valid inputs beyond the acceptance ones. Record on both sides: network calls (method, path, payload shape, status) and the rendered result. Every difference must match a row in the intended-changes table. A difference with no row is either a regression to fix or an undeclared change to add with its reason. Differences caused by the oracle's own data or flags are not findings; note them (L021).
6. **Review.** Launch the `local-diff-reviewer` agent with: base ref, the task and acceptance criteria, the map path. If it changed any code: run `/code-learnings update` with its report, then repeat steps 4 and 5.
7. **QA.** Run `/learnings qa` to build the QA block. Launch a general-purpose agent with that block plus the acceptance flows, the intended-changes table, environment, and the map path, asking it to verify the feature and classify every finding introduced / pre-existing-made-reachable / pre-existing-untouched. Introduced or made-reachable: fix, run `/learnings update`, repeat steps 4 and 5, rerun QA on the new diff. Verify the agent's evidence before relaying it (L015).
8. **Report** in the shape below.

## BLOCKED

Cannot start the app, cannot reach the oracle, or the spec leaves two readings that would produce different work. If you can ask the user, ask once with the exact gap. As a subagent, finish every step that does not depend on it, state the assumption you took where one reading was clearly safer, and end with `BLOCKED:` plus what you tried, the exact limitation, and what you need. Never report verification you did not perform.

## Report

```
Result: done | partial | BLOCKED
Change: <files, one line each>
Intended behaviour changes: <table or map section reference>
Oracle comparison: <flows compared; every difference accounted for yes/no; unaccounted ones listed>
Verification: <browser observation after cleanup; checks run with pass/fail output>
Review: <findings count, what was changed, learnings recorded>
QA: <outcome; findings by class>
Limitations: <what could not be verified and why>
Map: <path>
```

## Red flags

- Code written before the intended-changes table has a row for it
- A difference from the oracle explained after the fact instead of declared
- Oracle comparison skipped without a line under Limitations
- Green tests used as the only evidence
