---
name: working-with-roborev
description: "Use after pushing in a repo whose git hooks call roborev, or when handling a roborev verdict, finding, or Stop reminder: wait with roborev-wait-push, read the stored verdict, and fix or comment-then-close findings."
---

# Working with roborev

[roborev](https://github.com/kenn-io/roborev) reviews commits in the background: its git hooks enqueue jobs with a local daemon, and an AI agent writes a review with a stored verdict, `P` (pass) or `F` (fail). A repo uses it when its hooks call roborev: `grep -l roborev "$(git rev-parse --git-common-dir)"/hooks/*` (or the `core.hooksPath` directory). `roborev version` names the installed version; the source citations below are paths in the roborev repo at v0.71.0.

## The 80% path

1. Commit, push, then run `roborev-wait-push` (optional ref, default `HEAD`) from the repo, any subdirectory, or any of its worktrees. It finds the review job that ends at that commit, waits for it, and ends with one line, `roborev-wait-push: <result> (exit N)`, so `| tail -1` shows the outcome. A review usually takes one to two minutes after the push, which can outlast the Bash tool's default 2-minute timeout: run it with a long timeout (`timeout: 600000`) or in the background.
2. Act on the exit code:

   | Exit  | Meaning                                                                  | Do                                                                                                                                                                  |
   | ----- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | 0     | PASS, or a closed FAIL                                                   | Done.                                                                                                                                                               |
   | 1     | FAIL open                                                                | Handle each finding ([below](#handling-findings)), then wait again.                                                                                                 |
   | 2     | No job ends at that commit within 60 s, or the repo has no jobs at all   | Never a pass. Was it pushed, from a branch? See `~/.roborev/post-commit.log`; for a repo with no jobs, `roborev init`. `ROBOREV_WAIT_PUSH_TIMEOUT` sets the window. |
   | 3     | Usage, git, or daemon error, or the job was still running after the wait | Read the message: rerun for a still-running job, `roborev status` for the daemon.                                                                                   |
   | 4     | The job ended with no verdict (agent error, canceled)                    | No review happened, so there is nothing to close; report the range as unreviewed, not passed. The message shows the job's error.                                    |
   | 128+N | INTERRUPTED: the script was killed by a signal, such as a tool timeout's | Rerun it, with a longer timeout. Not a daemon error.                                                                                                                |

   A closed FAIL passes whoever closed it, since `roborev compact` closes the jobs it consolidates without a comment. When you close one yourself, comment first (below).

3. Warnings before the final line name **other** open FAIL jobs, and other queued or running jobs, on the same branch. They don't change the exit code, but a clean verdict for this push doesn't clear them: handle them too.
4. Report the reviewed range with its verdict and anything you dispositioned.

## Which commits get a job

- **Batched repo** (`post_commit_batch_size` of 2 or more in `.roborev.toml`): commits get no job of their own. The pre-push hook enqueues one **range** job, `<last reviewed>..<pushed head>` (a branch also flushes on its own every N commits). `roborev wait <sha>` reports "No job found" and `roborev show <sha>` "no review found" for a range head, since they match `git_ref` exactly against one SHA (`cmd/roborev/daemon_client.go` `findJobForCommit`); `roborev-wait-push` matches the range end.
- **Unbatched repo:** every commit gets a job. `roborev wait <sha>` works, and so does `roborev-wait-push <sha>`.
- **A push from a detached HEAD enqueues nothing**, because the pre-push flush keys on the branch. After `git push origin <sha>:refs/heads/<branch>`, queue the range yourself with `roborev review --since <base> --wait` and read that job's verdict.
- `roborev list` shows only the current branch; use `roborev list --all-branches` (add `--open` for unresolved jobs), or worktree commits look unreviewed.

## Reading a verdict

- **Judge the stored verdict, not the review's `Agent assessment:` line.** roborev re-derives the verdict from `review_min_severity` (`internal/review/runner.go`), so with a `medium` gate a review with only Low findings prints `Agent assessment: Fail` but is stored `P`. The stored verdict is the `Verdict` column of `roborev list`, `.job.verdict` in `roborev show --json`, or the script's exit code. Low findings don't block or trigger the hook, but read them.
- **A `failed` status is not a failed review.** It is an agent error (quota, empty diff): no verdict, nothing to close. `roborev wait` exits 1 on it too, which is why the script decides from the job's state instead.

## Handling findings

- **Real defect:** fix it, commit, push, wait again. The old job stays open; `roborev compact --wait --branch <branch>` re-checks open findings against the current code, closes the fixed ones, and opens one consolidated job for what remains. It costs an agent run; don't run two on one branch at once.
- **False positive, or not worth fixing:** `roborev comment --job <id> -m "<evidence>"`, **then** `roborev close <id>`. Comments feed later reviews; a bare close teaches nothing. Both take one id each.
- **Judge findings by who hits them.** A finding that needs an input no ordinary user produces (a crafted environment value, a NUL byte) is a limit: comment with that reason and close. A fix adds a guard nobody needs, and the next review probes the guard. Trust boundaries (untrusted input, auth, secrets) and anything that can lose data are the exceptions, judged on impact.
- **A finding that lands on a design limit belongs in the code.** The reviewer reads the checkout, so a header comment stating a file's purpose, scope, and known limits is review guidance for that file, and often stops a repeated finding class.
- **A rebase, amend, force push, or merge from the default branch widens the next range** in a batched repo, by design, so it carries upstream or parent-branch code. Fix only findings in this branch's own diff (`git diff origin/HEAD...HEAD`, or against the parent branch for a stack); `comment` then `close` the rest, naming the branch or PR that owns that code. For a scoped verdict, `roborev review --branch --base <parent-branch> --wait`.
- **After replaying commits** (rebase, cherry-pick) in an unbatched repo, judge each per-commit finding at the branch head: one that a later commit in the same push fixes is closed with a comment naming that commit.
- **Skip a trivial push** (a typo, a docs-only tweak) by putting `[no-roborev]` in every commit message of the push, when the repo's `.roborev.toml` lists it in `excluded_commit_patterns`. A range is skipped only when every commit in it carries the marker, so `roborev-wait-push` then reports no job (exit 2): expected here, not a pass to report.
- **Declare intended gaps in the commit body** ("Intermediate: X removed here, replaced in the next commit"). The body is in the review prompt, so a declared gap isn't flagged.
- Don't run `roborev fix` or `roborev refine` with uncommitted work in the tree; they commit.

## Agent-hook reminders

If `roborev agent-hook install` is set up, a Stop reminder names open FAIL job ids and an exact `roborev agent-hook fix-done <session>` command. Run `/roborev-fix <ids>` (roborev's own skill), then that exact command, and only once each job is fixed or commented and closed.

- **The hook counts open FAILs by repo and branch**, not by session or worktree, so a reminder can name jobs from another agent's commits on the same branch. Send those to the agent that made the commits, which has the context and the worktree. A session that coordinates other agents can start with `ROBOREV_AGENT_HOOK_FAILED_REVIEW_THRESHOLD=0` and read verdicts with `roborev list --all-branches --open`; `roborev snooze on -d 2h` mutes one worktree and branch for a while.
- **Hook state is never pruned.** Every session that runs a tool in a git repo gets an entry in `~/.roborev/agent-hook/state.json`, scripted runs in temp repos included. Clear those by id, which touches nothing live:

  ```bash
  roborev agent-hook status | jq -r '.sessions | to_entries[] | select(.value.last_cwd | test("^/(private/)?tmp/|^/var/folders/")) | .key' | xargs -n1 roborev agent-hook reset
  ```

  `roborev agent-hook reset --all` also clears every session's fix ownership; run it only when no agent is mid-fix. Launch throwaway `claude -p` runs with `--settings '{"disableAllHooks": true}'` so they never record state.

## Repo configuration

- **`REVIEW.md` and `.roborev.toml` are read from the default branch** for commit and range reviews (roborev `docs/configuration.md`), so a branch's edits to them apply only after merge.
- **Tune from history:** `roborev insights --since 30d` (slow; run it in the background), then fold recurring false positives into `REVIEW.md`, or into a file's header when they concern that file's design. Its counts include reviews of code since deleted, so check that a pattern still exists before acting on it.
