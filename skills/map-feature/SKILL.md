---
name: map-feature
description: Use before changing any code in a web app feature you have not traced in this session, when a task names a route, screen, flow, or bug to reproduce, or when asked to understand, map, trace, or reproduce how a feature currently behaves.
---

# Map feature

Establish how the feature behaves today, from runtime evidence, and write it down once so every later step and session reuses it. Nothing is changed in this skill. Scale depth to the task: a one-file text change needs Boundaries and the one Flow step it touches; anything touching state, requests, flags, or shared code gets every section.

## Inputs

Task, feature or route (name, URL, or entry files), expected behaviour, reproduction steps if known, acceptance criteria. Missing entry point: find it from the route table or the screen's visible text, then continue.

## Repo facts

Read the repo's `CLAUDE.md` for a `## Agent repo facts` section:

```
## Agent repo facts
- run: npm start -> http://localhost:3000
- login: <test account or how to sign in>
- oracle: https://staging.example.com   (deployed app treated as the old system)
- standards: CODING_STANDARDS.md, docs/frontend-guidelines.md
- flags: <how feature flags are toggled locally>
- feature-maps: .claude/feature-maps/
```

Missing fields: infer from `package.json` scripts, `README`, env files, and record what you inferred under Environment in the map. If a new worktree is missing `node_modules` or gitignored env files, install and copy them from the main checkout before judging anything (L022 in `~/.claude/skills/learnings/`). Cannot start the app: write the map's static sections, then stop with a BLOCKED report naming the exact failure.

## Steps

1. **Load context.** Repo instructions, existing map at `<feature-maps>/<feature>.md` (update it, never recreate), tests for the feature, one similar implementation if it clarifies conventions. If `~/.claude/memory/projects/<repo>/context.json` exists, read it instead of rescanning the repo.
2. **Scope.** From the entry files follow imports and consumers outward. Stop at shared utils, design-system components, and packages; list those by name only. Record anything outside this app that reads the same state or contract (L009).
3. **Trace the flow on paper.** One line per step with `file:line`: user action -> handler -> state update or side effect -> API/service -> response handling -> rendered result. Include routing, permissions, feature flags, caching, and background events where they touch this task. Both branches of any flag that gates the flow (L010).
4. **Run it.** Start the app with the repo facts. Open the flow in a browser tab in its own window; assert `document.visibilityState === "visible"` before any measurement (L007). Reproduce the target flow once; note environment, user role, test data, and flag state.
5. **Observe.** Console, network requests (method, path, payload shape, status), state transitions (Redux actions in order with the slice, or hook/context updates). Use the browser tools and existing logs first. Add a temporary log only to resolve a named uncertainty, always prefixed `[TRACE]`, never with credentials or personal data. Mind the tool's output limits (L008): keep results in `window` and read back in chunks.
6. **Connect.** Map each observation to the code line that caused it. Mark each statement **observed** (runtime) or **inferred** (source). Mark mocked responses and dependencies you could not reach.
7. **Write the map** in the template below. Keep `.claude/feature-maps/` out of the diff: append the line to `$(git rev-parse --git-path info/exclude)` if it is not already there.
8. **Clean.** `rg '\[TRACE\]'` across the repo and remove every hit before the next step. Report "no instrumentation left" only after the search returns nothing.

Blocked or inspection impossible: report the exact limitation, finish every section that does not depend on it, and never claim a runtime observation you did not make.

## Clean verification (used by fix-bug and build-feature after any code change)

1. `rg '\[TRACE\]|debugger'` across the repo and remove every hit.
2. Rerun the target flow in the browser and record what you observed (network, state, UI) as evidence in the map.
3. Run the repo's checks: typecheck, lint, the feature's tests, then the full suite if the repo documents one. Keep the output.
4. Read the whole diff once for unrelated changes and orphaned imports or variables.

Evidence recorded while instrumentation was still in the tree does not count (L012).

## Map template

```markdown
# <feature>
Updated: YYYY-MM-DD · task: <one line> · branch: <name>

## Environment
run · url · login/role · flags · test data · inferred vs documented

## Boundaries
Entry points. Key files and symbols. Stop lines (shared utils, design system, packages) by name.

## Flow
1. <user action> -> <handler file:line> -> <state/effect> -> <request> -> <response handling> -> <render>

## Contracts and state
Endpoints (method, path, payload shape, status). State ownership: slice/selectors/thunks or hooks/context.

## Consumers
Shared code or state this feature touches that other screens or apps use.

## Repro and tests
Steps to reproduce. Relevant test files and their baseline result.

## Findings
Observed: ...   Inferred: ...   Mocked/inaccessible: ...   Unknowns: ...

## Intended behaviour changes
| change | reason |
```

## Red flags

- Editing source before the flow has been observed once in the browser
- A Flow line with no `file:line`
- "Observed" written for something read from code
- Map written from a hidden tab or a stale build (L007, L012)
