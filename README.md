# roborev-companion

A Claude Code companion for [roborev](https://github.com/kenn-io/roborev), the AI review daemon that reviews your commits from git hooks. roborev ships its own Claude Code hooks and skills; this plugin adds what they leave out: a script that waits on a **push's** review (batched range jobs included) and a skill that teaches Claude how to read verdicts and disposition findings.

## Setup

1. **Install roborev**: `brew install kenn-io/tap/roborev`, or `curl -fsSL https://roborev.io/install.sh | bash`. Check with `roborev version`.
2. **Install roborev's hooks and skills for Claude Code**: `roborev agent-hook install` writes its hooks into Claude Code's settings and installs roborev's bundled skills (`roborev-fix`, …); check with `roborev agent-hook status`. This plugin has no hooks of its own and doesn't duplicate those.
3. **Install this plugin** in Claude Code:

   ```text
   /plugin marketplace add warmlogic/ai-plugin-marketplace
   /plugin install roborev-companion@ai-plugin-marketplace
   ```

4. **Set up each repo**: `roborev init` installs the post-commit, post-rewrite, and pre-push hooks and starts the daemon (check with `roborev status`). Then add a `.roborev.toml` and a root `REVIEW.md` ([below](#configuration-best-practices)).
5. **Tell Claude when to use it**: roborev's git hooks run silently, so add one line to your own `~/.claude/CLAUDE.md` (not a repo's shared `CLAUDE.md`, since teammates may not have roborev or this plugin): "In a repo whose git hooks call roborev (`grep -l roborev "$(git rev-parse --git-common-dir)"/hooks/*`), after pushing, run `roborev-wait-push` and act on its verdict; load the `roborev-companion:working-with-roborev` skill first."
6. **Verify**: commit and push something trivial, then run `roborev-wait-push`; it should end with `roborev-wait-push: PASS (exit 0)` or a FAIL.

After `roborev update`, rerun `roborev init` in each repo and `roborev agent-hook install`: the hooks change between versions.

## What it provides

- **`bin/roborev-wait-push [ref]`**, on Claude's Bash `PATH` while the plugin is enabled. It finds the newest review job whose range ends at the ref (default `HEAD`), waits for it, and decides from the job's final state. Its exit codes are listed in [the skill](skills/working-with-roborev/SKILL.md#the-80-path). It warns about other open FAIL and unfinished jobs on the same branch, and its last line always names the result and exit code, so `| tail -1` shows it. `roborev wait <sha>` can't do this in a batched repo: it matches one SHA exactly and finds no job for a range's head.
- **The `working-with-roborev` skill**, which Claude loads after pushing in a roborev repo or when handling a verdict, finding, or agent-hook reminder: which commits get a job, the severity gate, `failed` jobs, comment-then-close dispositions, widened ranges after a rebase, and hook-state hygiene. For fixing findings it defers to roborev's own `/roborev-fix`.

## Configuration best practices

### Global config

Set keys with `roborev config set --global <key> <value>` (the daemon hot-reloads) and leave the rest at their defaults:

```toml
# ~/.roborev/config.toml
default_agent = 'codex'              # a different model family from the agent writing the code
default_model = '<model>'            # explicit: roborev ignores ~/.codex/config.toml, and an upgrade can't move it
review_reasoning = 'medium'          # explicit: empty means "thorough"
review_context_count = 3             # prior reviews and their comments feed the next review
review_min_severity = 'medium'       # the stored verdict is Fail only for a medium-or-higher finding
fix_min_severity = 'medium'          # fix passes act on the same band
auto_close_passing_reviews = true    # the open list holds only work
default_backup_agent = 'claude'      # keeps reviews flowing when the primary's quota runs out
default_backup_model = 'sonnet'
agent_quota_cooldown = '30m0s'       # how long to stay on the backup before retrying the primary

[agent_hook]
failed_review_threshold = 1          # one open Fail surfaces at the next Stop (default 4)

# Failover to the backup agent is silent and has no event of its own: detect it from {agent}
[[hooks]]
event = "review.*"
command = "<script> {agent} {job_id} {repo_name}"   # exits quietly for the primary; otherwise logs one line per job; never fails
```

- **One explicit model and effort default, overrides by exception.** The per-push review is the frequent first line, so it runs at a moderate default; depth belongs to the pre-merge review. The daemon resolves model and reasoning from config when it enqueues, so nothing in the push can raise one push's level. For a risky push, let the automatic job run, then add one deeper pass over the range: `roborev review --since <base> --reasoning xhigh`. For a trivial push, mark every commit in it with `[no-roborev]` (the per-repo config below lists that marker).

### Per repo

- **A tight root `REVIEW.md`, committed to the default branch**: what the repo values, what to flag (its domain's failure classes, each as a demonstrable defect), and what to skip (speculative edge cases, style, anything CI reports). It is the strongest false-positive filter, and the agent hook stays silent in a repo without review guidance. roborev reads it from the default branch, so a branch's edits apply only after merge; [the skill](skills/working-with-roborev/SKILL.md#repo-configuration) covers that and tuning it from review history.
- **A committed `.roborev.toml` with one review per push.** The recommended default for every repo:

  ```toml
  # .roborev.toml
  post_commit_batch_size = 20                 # one review per push, not per commit
  excluded_commit_patterns = ['[no-roborev]'] # a push whose every commit carries the marker isn't reviewed
  exclude_patterns = []                       # generated and lock files that have their own drift tests, e.g. "uv.lock"
  ```

  With batching, the post-commit hook only records commits, and the pre-push hook enqueues one range job, `<last reviewed>..<pushed head>`, before the push proceeds, so each push gets one verdict; a branch also flushes on its own every 20 commits. A failed flush never blocks the push; the reason is in `~/.roborev/post-commit.log`. Trade-off: a rebase or a merge from the default branch widens the next range (the skill covers handling that). Committing the file gives every branch and worktree the same settings, and it's inert for teammates without roborev. If teammates shouldn't get batching, keep the file untracked instead and list it in `.git/info/exclude`: a linked worktree with no copy of its own then reads the main checkout's file.
- **An absolute `core.hooksPath`.** If the repo commits its hooks directory (`core.hooksPath` pointing at a tracked `.githooks/`), roborev installs into it and edits tracked files: either commit its changes (they no-op where roborev isn't installed) or keep its hook files (`post-commit`, `post-rewrite`, `pre-push`) untracked and listed in `.git/info/exclude`. Keep `core.hooksPath` absolute, pointing at the main checkout's hooks directory, so linked worktrees share the roborev hooks.
  Never hide a roborev edit with `git update-index --skip-worktree`: it blocks `git pull` when the tracked hook changes upstream.

## Development

See [AGENTS.md](AGENTS.md).

## License

[MIT](LICENSE)
