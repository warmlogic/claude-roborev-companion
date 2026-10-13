# roborev-companion

A Claude Code companion for [roborev](https://github.com/kenn-io/roborev), the AI review daemon that reviews your commits from git hooks. roborev ships its own Claude Code hooks and skills; this plugin adds what they leave out: a script that waits on a **push's** review (batched range jobs included) and a skill that teaches Claude how to read verdicts and disposition findings.

## Setup

1. **Install roborev**: `brew install kenn-io/tap/roborev`, or `curl -fsSL https://roborev.io/install.sh | bash`. Check with `roborev version`.
2. **Install roborev's skills and hooks for Claude Code**: `roborev skills install`, then `roborev agent-hook install` (it writes its hooks into Claude Code's settings; check with `roborev agent-hook status`). This plugin has no hooks of its own and doesn't duplicate those.
3. **Install this plugin** in Claude Code:

   ```text
   /plugin marketplace add warmlogic/ai-plugin-marketplace
   /plugin install roborev-companion@ai-plugin-marketplace
   ```

4. **Set up each repo**: `roborev init` installs the post-commit, post-rewrite, and pre-push hooks and starts the daemon (check with `roborev status`). Then add a `.roborev.toml` and a root `REVIEW.md` ([below](#configuration-best-practices)).
5. **Verify**: commit and push something trivial, then run `roborev-wait-push`; it should end with `roborev-wait-push: PASS (exit 0)` or a FAIL.

After a roborev upgrade (`roborev update`), rerun `roborev init` in each repo and `roborev agent-hook install`.

## What it provides

- **`bin/roborev-wait-push [ref]`**, on Claude's Bash `PATH` while the plugin is enabled. It finds the newest review job whose range ends at the ref (default `HEAD`), waits for it, and decides from the job's final state. Exit codes: `0` pass, or a FAIL closed with a comment; `1` an open FAIL, or one closed without a comment; `2` no job appeared within 60 s (`ROBOREV_WAIT_PUSH_TIMEOUT`), never a pass; `3` usage, git, or daemon error; `4` the job ended with no verdict (agent error, canceled). It warns about other open FAIL jobs on the same branch, and its last line always names the result and exit code, so `| tail -1` shows it. `roborev wait <sha>` can't do this in a batched repo: it matches one SHA exactly and finds no job for a range's head.
- **The `working-with-roborev` skill**, which Claude loads after pushing in a roborev repo or when handling a verdict, finding, or agent-hook reminder: which commits get a job, the severity gate, `failed` jobs, comment-then-close dispositions, widened ranges after a rebase, and hook-state hygiene. For fixing findings it defers to roborev's own `/roborev-fix`.

## Configuration best practices

### Global config

Set keys with `roborev config set --global <key> <value>` (the daemon hot-reloads) and leave the rest at their defaults:

```toml
# ~/.roborev/config.toml
default_agent = 'codex'              # a different model family from the agent writing the code
default_model = '<model>'            # explicit, so an upgrade or a changed agent default can't move it
review_reasoning = 'medium'          # explicit: empty means "thorough"
review_context_count = 3             # prior reviews and their comments feed the next review
review_min_severity = 'medium'       # the stored verdict is Fail only for a medium-or-higher finding
fix_min_severity = 'medium'          # fix passes act on the same band
auto_close_passing_reviews = true    # the open list holds only work

[agent_hook]
failed_review_threshold = 1          # one open Fail surfaces at the next Stop
```

- **One explicit model and effort default, overrides by exception.** The per-push review is the frequent first line, so it runs at a moderate default; depth belongs to the pre-merge review. The daemon resolves model and reasoning from config when it enqueues, so nothing in the push can raise one push's level. For a risky push, let the automatic job run, then add one deeper pass over the range: `roborev review --since <base> --reasoning xhigh`. For a trivial push, put a marker such as `[no-roborev]` in the commit message and list it in `.roborev.toml` as `excluded_commit_patterns = ['[no-roborev]']`; a range is skipped only when every commit in it carries the marker.
- **Keep secret fields in `~/.roborev/config.toml` empty** and supply secrets through the environment or the Keychain, so the config file itself can be versioned.
- `roborev config set --global default_backup_agent <agent>` keeps reviews flowing when the primary agent's quota runs out. If Codex is your reviewer, a quota checker helps explain a run of `failed` jobs, but nothing here depends on one.

### Per repo

- **A tight root `REVIEW.md`, committed to the default branch**: what the repo values, what to flag (its domain's failure classes, each as a demonstrable defect), and what to skip (speculative edge cases, style, anything CI reports). It is the strongest false-positive filter, and the agent hook stays silent in a repo without review guidance. roborev reads `REVIEW.md` and `.roborev.toml` from the default branch for commit and range reviews, so a branch's edits apply only after merge.
- **`exclude_patterns`** in `.roborev.toml` for generated and lock files that have their own drift tests.
- **Per-push batching**, recommended for agent-heavy repos: `post_commit_batch_size = 20`. The post-commit hook then only records pending commits, and the pre-push hook enqueues one range job, `<last reviewed>..<pushed head>`, before the push proceeds; a branch also flushes on its own every 20 commits. A failed flush never blocks the push; the reason is in `~/.roborev/post-commit.log`. Trade-off: a rebase or a merge from the default branch widens the next range (the skill covers handling that).
- **A machine-local `.roborev.toml`**: keep it untracked and listed in `.git/info/exclude`. A linked worktree with no copy reads the main checkout's file, but only while it is untracked; a tracked `.roborev.toml` is branch-specific and never inherited. That is what makes batching apply in agent worktrees.
- **An absolute `core.hooksPath`.** If the repo commits its hooks directory (`core.hooksPath` pointing at a tracked `.githooks/`), roborev installs into it and edits tracked files: either commit its changes (they no-op where roborev isn't installed) or keep its hook files (`post-commit`, `post-rewrite`, `pre-push`) untracked and listed in `.git/info/exclude`. Keep `core.hooksPath` absolute, pointing at the main checkout's hooks directory, so linked worktrees share the roborev hooks.

### Tuning

Run `roborev insights --since 30d` now and then (it takes minutes) and fold recurring false positives into `REVIEW.md`, or into a file's header comment when they concern that file's design.

## Development

See [AGENTS.md](AGENTS.md).

## License

[MIT](LICENSE)
