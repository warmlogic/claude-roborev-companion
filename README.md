# roborev-companion

A Claude Code companion for [roborev](https://github.com/kenn-io/roborev), the AI review daemon that reviews your commits from git hooks. roborev ships its own Claude Code hooks and skills; this plugin adds what they leave out: a script that waits on a **push's** review (batched range jobs included) and a skill that teaches Claude how to read verdicts and disposition findings.

Setup, below, is done once per machine and once per repo. Day to day, Claude loads the `working-with-roborev` skill on its own, so there's nothing more to read.

## Setup

1. **Install roborev and `jq`** (the script needs it): `brew install kenn-io/tap/roborev jq`, or roborev's `curl -fsSL https://roborev.io/install.sh | bash`. Check with `roborev version`.
2. **Configure roborev**: edit `~/.roborev/config.toml` using the [block below](#global-config); the daemon reloads it on its own. `roborev config set --global` handles scalar keys but not `[[hooks]]`, so editing the file is the simple path.
3. **Install roborev's hooks and skills for Claude Code**: `roborev agent-hook install` writes its hooks into Claude Code's settings and installs roborev's bundled skills (`roborev-fix`, …); check with `roborev agent-hook status`. This plugin has no hooks of its own and doesn't duplicate those.
4. **Install this plugin** in Claude Code:

   ```text
   /plugin marketplace add warmlogic/ai-plugin-marketplace
   /plugin install roborev-companion@ai-plugin-marketplace
   ```

5. **Set up each repo**: `roborev init` installs the post-commit, post-rewrite, and pre-push hooks and starts the daemon (check with `roborev status`). Then add a `.roborev.toml` and a root `REVIEW.md` ([below](#configuration-best-practices)).
6. **Tell Claude when to use it**: roborev's git hooks run silently, so add this line to your own `~/.claude/CLAUDE.md` (not a repo's shared `CLAUDE.md`, since teammates may not have roborev or this plugin):

   ```text
   In a repo whose git hooks call roborev (`grep -l roborev "$(git rev-parse --git-path hooks)"/*`), after pushing, run `roborev-wait-push` and act on its verdict; load the `roborev-companion:working-with-roborev` skill first.
   ```

7. **Verify**: the script is on Claude's Bash `PATH` only, so in a Claude Code session in the repo (after `/reload-plugins`, or in a new session), commit and push a small change without `[no-roborev]`, then ask Claude to run `roborev-wait-push`. It should end with `PASS (exit 0)` or `FAIL open (exit 1)`.

## What it provides

- **`bin/roborev-wait-push [ref]`**, on Claude's Bash `PATH` while the plugin is enabled. It finds the newest review job whose range ends at the ref (default `HEAD`), waits for it, and decides from the job's final state. Its exit codes are listed in [the skill](skills/working-with-roborev/SKILL.md#the-80-path). It warns about other open FAIL and unfinished jobs on the same branch.
- **The `working-with-roborev` skill**, which tells Claude how to wait on a push's review and act on each verdict.

## Configuration best practices

### Global config

Set these keys and leave everything else at its default. The block assumes the Codex CLI is installed and logged in.

```toml
# ~/.roborev/config.toml
default_agent = 'codex'              # a different model family from the agent writing the code
default_model = '<model>'            # your Codex model; explicit: roborev ignores ~/.codex/config.toml, and an upgrade can't move it
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
command = "<script> {agent} {job_id} {repo_name}"   # a script you write: exits quietly for the primary; otherwise logs one line per job; never fails
```

- **One explicit model and effort default, overrides by exception.** The per-push review is the frequent first line, so it runs at a moderate default; depth belongs to the pre-merge review. The daemon resolves model and reasoning from config when it enqueues, so nothing in the push can raise one push's level. For a risky push, let the automatic job run, then add one deeper pass over the range: `roborev review --since <base> --reasoning xhigh`.

### Per repo

- **A tight root `REVIEW.md`, committed to the default branch**: state each finding class as a demonstrable defect, and skip anything CI reports. It is the strongest false-positive filter, and the agent hook stays silent in a repo without review guidance. Claude Code's Code Review also reads it, so write it as team-wide review guidance. roborev reads it from the default branch, so a branch's edits apply only after merge; [the skill](skills/working-with-roborev/SKILL.md#repo-configuration) covers that and tuning it from review history.
- **A committed `.roborev.toml` with one review per push.** The recommended default for every repo:

  ```toml
  # .roborev.toml
  post_commit_batch_size = 20                 # one review per push, not per commit
  excluded_commit_patterns = ['[no-roborev]'] # a push whose every commit carries the marker isn't reviewed
  exclude_patterns = []                       # generated and lock files that have their own drift tests, e.g. "uv.lock"
  ```

  - With batching, the post-commit hook only records commits, and the pre-push hook enqueues one range job, `<last reviewed>..<pushed head>`, before the push proceeds; a branch also flushes on its own every 20 commits. A rebase or a merge from the default branch widens the next range (the skill covers handling that).
  - A failed flush never blocks the push; the reason is logged in `~/.roborev/post-commit.log`.
  - Commit the file so every branch and worktree gets it; it's inert for anyone without roborev.

- **Only if the repo sets `core.hooksPath`** (`git config --get core.hooksPath`; empty means skip this): roborev installs its hooks into that directory, which may be tracked.
  - Commit roborev's hook changes (they no-op where roborev isn't installed), or keep its hook files (`post-commit`, `post-rewrite`, `pre-push`) untracked and listed in `.git/info/exclude`.
  - Never hide a roborev edit with `git update-index --skip-worktree`: it blocks `git pull` when the tracked hook changes upstream.
  - Keep the path absolute, pointing at the main checkout's hooks directory, so linked worktrees share the roborev hooks.

## Development

See [AGENTS.md](AGENTS.md).

## License

[MIT](LICENSE)
