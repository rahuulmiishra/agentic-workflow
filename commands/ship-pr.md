---
argument-hint: <upstream-branch> [new-branch-name] [--no-verify]
description: Move all working-tree changes (modified + untracked) to a new branch, push, and open a PR against the upstream branch
allowed-tools: Bash, Read, Edit, Write
---

You are executing the `/ship-pr` workflow.

Args received: `$ARGUMENTS`

Parse `$ARGUMENTS` as a whitespace-separated list. Extract:
- **`--no-verify`** flag — if present anywhere in the args, set `NO_VERIFY=true` and remove the token from the positional list. Default `NO_VERIFY=false`.
- **Positional args** (after stripping flags):
  - 1st positional → upstream branch (REQUIRED, e.g. `stage-hyp`, `main`, `stage-common`)
  - 2nd positional → new branch name (OPTIONAL; auto-generate if omitted)

If the upstream branch is empty, stop and tell the user: "`/ship-pr` requires an upstream branch. Usage: `/ship-pr <upstream> [branch-name] [--no-verify]`".

If `NO_VERIFY=true`, surface a one-line warning to the user before continuing: "⚠ `--no-verify` set — pre-commit + commit-msg hooks will be skipped."

## Goal

Take **everything** uncommitted in the current working tree (modified files AND newly created/untracked files), put it on a new branch, commit, push, and open a PR targeting `$1`.

## Steps

### 1. Sanity checks (run these Bash calls in parallel)

- `git rev-parse --is-inside-work-tree` — must be a git repo
- `git status --porcelain` — capture working-tree state
- `git rev-parse --abbrev-ref HEAD` — current branch
- `git ls-remote --exit-code --heads origin "$1"` — verify upstream exists on origin
- `gh auth status` — verify gh CLI is authenticated

### 2. Validate

- If `git status --porcelain` is empty → abort: "No working-tree changes to ship."
- If upstream lookup failed → run `git fetch origin "$1":refs/remotes/origin/$1` once and re-check. Still missing → abort: "Upstream branch '$1' not found on origin."
- If `gh auth status` failed → abort and tell user to run `gh auth login`.

### 3. Decide new branch name

- If `$2` provided → use verbatim. Check collision: `git rev-parse --verify "$2" 2>/dev/null` AND `git ls-remote --exit-code --heads origin "$2"`. Collision → abort with the conflicting name.
- Else → auto-generate. Format: `<upstream-short>-wip-<YYMMDD>-<3-letter-slug>`
  - `<upstream-short>`: strip `stage-` / `preprod-` prefix from `$1` (e.g. `stage-hyp` → `hyp`)
  - `<3-letter-slug>`: short hash of changed-file basenames (`git status --porcelain | awk '{print $2}' | md5 | cut -c1-3`)
  - Example: `hyp-wip-260429-a3f`

### 4. Inspect changes (so the commit + PR body are meaningful)

- `git diff` — modified files
- For untracked files: list paths from `git status --porcelain | grep '^??'`. For each `.js/.ts/.tsx/.jsx/.md/.json` file, `Read` the first 100 lines to understand intent. Skip binaries and lock files.
- `git log --oneline -5 "origin/$1"` — match repo's commit message style

### 5. Create branch + stage everything

- `git checkout -b <new-branch>` — uncommitted changes carry over
- `git add -A` — stages modified, new, and deleted files (THIS is what guarantees newly created files are included)
- Run `git diff --cached --name-only` and reject the commit if any of these patterns appear:
  - `.env`, `.env.*`, `*.pem`, `*.key`, `id_rsa*`, `*credentials*`, `*secrets*`, `*.p12`, `*.pfx`
  - Anything inside `node_modules/`
  - If a match is found → `git restore --staged <file>` for that file and tell the user which files were skipped (don't abort the whole commit unless ALL files were skipped).

### 6. Compose commit message

- Subject: ≤ 72 chars, imperative mood, summarize the change ("why" over "what" when non-obvious).
- Body: 1–3 short bullets describing what changed. Skip body if subject is self-explanatory.
- Footer (always):
  ```
  Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
  ```
- Match repo style observed in step 4 (e.g. `feat(scope): ...`, `TICKET-123: ...`, plain prose — whichever the recent log uses).

Commit with HEREDOC. If `NO_VERIFY=true`, append `--no-verify` to the commit command:
```bash
# default
git commit -m "$(cat <<'EOF'
<subject>

<optional body>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"

# when NO_VERIFY=true
git commit --no-verify -m "$(cat <<'EOF' ... EOF
)"
```

**If the commit fails due to a pre-commit hook AND `NO_VERIFY=false`:** read the error, fix the underlying issue (e.g. lint), `git add` the fixed files, and create a NEW commit. Do **not** auto-add `--no-verify` — the user must opt in via the flag. Never `--amend`.

### 7. Push + open PR

```bash
git push -u origin <new-branch>
```

```bash
gh pr create --base "$1" --head <new-branch> \
  --title "<subject>" \
  --body "$(cat <<'EOF'
## Summary
- <bullet 1>
- <bullet 2>

## Test plan
- [ ] <relevant check>
- [ ] <relevant check>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Use the `--repo` flag only if the local remote is not the default. Otherwise let `gh` infer.

### 8. Report

Print:
- PR URL (so it's clickable)
- One-liner: `Branch <new-branch> → <upstream>. <N> files, +<adds>/-<dels>.`

## Guardrails

- Use `--no-verify` ONLY when the user passes the `--no-verify` flag in args. Never auto-add it on hook failure.
- Never use `--no-gpg-sign`, `--amend`, or `git push --force`.
- Never commit secrets — re-check the staged file list before `git commit` (this check still runs even when `--no-verify` is set).
- Never alter `.gitignore` to bypass a check.
- If the user's current branch IS the upstream (`HEAD == $1`), proceed normally — `checkout -b` works fine.
- If pre-commit hooks reformat files, they will be in the commit once hooks pass. Do not amend; just retry the commit after `git add -A`.
- Don't open a PR if `gh pr create` would target a branch with an open PR from the same head; surface the existing PR URL instead.
