# Plan: share Claude's plan and session context with nuke-cli/claude (issue #18)

## Context

Issue #18 asks Claude Code to export its working context automatically, so other sessions and machines can pick up a branch's work. There are two triggers:

1. **Plan approved:** create `webpage/context/<branch>/` in `nuke-cli/claude` if it's missing, and push the approved plan plus `context.txt`. That file holds this session's context, limited to this branch.
2. **Successful `git push`:** push updates to the plan and `context.txt`. A different session gets its own `==== START/END OF SESSION_ID <id> ====` section.

Earlier attempts (`acbfab9`, `cae9c9d`) were reverted. They wrote a local `context.txt` and a per-user remote file. This plan replaces them but reuses their tested pieces, read with `git show cae9c9d:.claude/hooks/store-context.sh`:
- the `git push` regex and the `--dry-run` skip
- the `jq` transcript filter
- the `awk` per-session block replace
- `push_remote`, which merges, PUTs, and retries on a stale sha

Decisions:
- The plan file is `plan_<ticket>.md`, where the ticket is the branch's leading number (`18-…` → `plan_18.md`). Branches without a number use `plan.md`.
- Files go to the remote only. No local `context.txt` is written, and `.gitignore` doesn't change.
- Uploads go through the GitHub contents API on `main`, not `git push`, so the hook can't retrigger itself.
- The destination base is `webpage/context`. `STORE_CONTEXT_REPO` and `STORE_CONTEXT_DIR` override it for testing.

## Implementation steps

Each step is one commit, made with the issue #17 format. Each step is tested against a scratch repo (`STORE_CONTEXT_REPO=<user>/scratch`) by piping fake hook JSON into the script, before moving to the next step.

### Step 1: upload the approved plan
**Commit:** `feat: push approved plan to nuke-cli/claude`

- New `.claude/hooks/store-context.sh`:
  - Read the hook input with `jq`: `tool_name`, `tool_input`, `session_id`, `transcript_path`.
  - Resolve the branch with `git -C "$CLAUDE_PROJECT_DIR" symbolic-ref --short -q HEAD`. During a rebase, use `.claude/skills/commit/branch.sh`'s logic. Exit if HEAD is detached.
  - Ticket: `^[0-9]+` from the branch, which gives `plan_<n>.md` or `plan.md`. Remote dir: `$STORE_CONTEXT_DIR/<branch>/`.
  - `put_file <remote path> <local file>`, adapted from `push_remote`:
    - GET the sha
    - skip if the content is unchanged
    - PUT with the commit message `docs(claude): update <branch> <file>` and `session_id: <id>`
    - retry once on a stale sha
    - temp files go in `mktemp -d`
  - For `ExitPlanMode`, write `tool_input.plan` to a temp file and `put_file` it.
  - Finish with a `systemMessage` naming what was pushed, or saying that it failed. It never blocks the tool call.
- New `.claude/settings.json` with a `PostToolUse` hook, `matcher: "ExitPlanMode"`, timeout 30. It fires only on approval.
- **Check:**
  - The fake `ExitPlanMode` input creates `webpage/context/<branch>/plan_18.md`, which also creates the folder.
  - A rerun makes no new commit.
  - A detached HEAD pushes nothing.

### Step 2: upload the branch's session context with the plan
**Commit:** `feat: push branch session context with the plan`

- Add the transcript filter from `cae9c9d`:
  - user and assistant text only
  - no meta or sidechain entries
  - `[user]` / `[claude]` labels
  - new: `select(.gitBranch == $branch)`
- Add `merge_context`:
  - fetch the remote `context.txt` if it exists
  - drop this session's old block with the `awk` filter
  - append the new `START/END OF SESSION_ID` block
- On `ExitPlanMode`, `put_file` `context.txt` too.
- **Check:**
  - Sessions `s1` then `s2` give two sections.
  - Rerunning `s1` replaces only its block.
  - Entries from another branch are excluded.

### Step 3: update both files after a successful push
**Commit:** `feat: update plan and context after a successful push`

- In `settings.json`, add a second `PostToolUse` entry: `matcher: "Bash"`, `if: "Bash(*git push*)"`. Failed pushes go to `PostToolUseFailure`, so they never reach the hook.
- In the script, for `Bash`:
  - Apply the `git push` regex and the `--dry-run` skip.
  - Re-upload `context.txt`.
  - Find the last `ExitPlanMode` `tool_use` plan in this transcript for this branch and re-upload it. If there's none, leave the remote plan alone.
- **Check:**
  - A `git push` input updates `context.txt` (and the plan when one exists).
  - `git push --dry-run` and `git status` push nothing.
  - Session `s2` with no plan leaves `plan_18.md` unchanged.

### Step 4: document it
**Commit:** `docs: document session context sharing`

- Add a "Session context" section to `CLAUDE.md` covering:
  - the two triggers
  - the `webpage/context/<branch>/{plan_<n>.md,context.txt}` layout
  - per-session sections
  - the `gh auth login` requirement
  - **that `nuke-cli/claude` is public**, so prompts and replies are published on every plan approval and push

### Step 5: live check (no commit)

- Run `bash -n` and `shellcheck` on the script.
- Approve a real plan in a session, then confirm `nuke-cli/claude/webpage/context/18-enable-store-the-current-claude-session-context/` has `plan_18.md` and `context.txt`.
- Push the branch and confirm `context.txt` is updated.
- Record any review findings in `gotchas/18-enable-store-the-current-claude-session-context.txt`.
