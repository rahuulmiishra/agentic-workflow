# agentic-workflow

Personal collection of Claude Code slash commands and agentic workflow assets.

## Layout

```
agentic-workflow/
└── commands/        # Claude Code slash commands (.md files)
    └── ship-pr.md
```

## Commands

### `/ship-pr <upstream-branch> [new-branch-name] [--no-verify]`

Move all working-tree changes (modified + newly created files) to a new branch, commit, push, and open a PR against the given upstream branch.

**Args**

| Arg | Required | Description |
|-----|----------|-------------|
| `<upstream-branch>` | yes | Target branch the PR will merge into (e.g. `stage-hyp`, `main`, `stage-common`) |
| `[new-branch-name]` | no | Custom name for the new branch. If omitted, auto-generated as `<short-upstream>-wip-<YYMMDD>-<slug>` |
| `--no-verify` | no | Pass to skip pre-commit + commit-msg hooks. The flag may appear anywhere in the args. Use only when you trust the working tree is clean and hooks are noisy/slow. |

**What it does**

1. Sanity checks the repo, working tree, upstream branch, and `gh` auth.
2. Picks a new branch name (your arg or auto-generated).
3. Inspects modified + untracked files so the commit message and PR body are meaningful.
4. `git checkout -b` carries the working-tree changes onto the new branch.
5. `git add -A` stages **all** changes including newly created files.
6. Filters out secrets (`.env`, `*.pem`, `*.key`, credentials, etc.) and `node_modules/` before committing.
7. Composes a commit message matching the repo's existing log style.
8. Pushes the branch with `-u` and opens a PR via `gh pr create --base <upstream>`.
9. Prints the PR URL.

**Guardrails**

- Uses `--no-verify` only when you explicitly pass the flag — never auto-added on hook failure.
- Never uses `--amend` or `git push --force`.
- Never commits secrets (secret-pattern check runs even with `--no-verify`).
- If a pre-commit hook fails (and `--no-verify` was NOT passed), fixes the underlying issue and creates a new commit.
- If a PR from the same head already exists, surfaces the existing URL instead of erroring.

**Example**

```text
/ship-pr stage-hyp
/ship-pr stage-common shared-pkg-bump
/ship-pr main TE-13231-hotfix
/ship-pr stage-hyp --no-verify
/ship-pr stage-hyp my-branch --no-verify
```

## Installing

Slash commands resolve from two locations:

| Scope | Path |
|-------|------|
| User (all projects) | `~/.claude/commands/` |
| Project (this repo only) | `<repo>/.claude/commands/` |

To use these commands across all your projects, symlink or copy `commands/ship-pr.md` into `~/.claude/commands/`:

```bash
mkdir -p ~/.claude/commands
ln -sf "$(pwd)/commands/ship-pr.md" ~/.claude/commands/ship-pr.md
```

Verify it shows up:

```bash
claude /help        # within a Claude Code session
```

Then invoke from anywhere with `/ship-pr <upstream>`.
