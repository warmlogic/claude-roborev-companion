# AGENTS.md — roborev-companion developer guide

This is the canonical contributor guide, at the repo root. `.claude/CLAUDE.md` is a symlink to this file, so Claude Code loads the same content as project memory.

A Claude Code plugin with no hooks: a script on the Bash tool's `PATH` and one skill, both for working in repos that [roborev](https://github.com/kenn-io/roborev) reviews. roborev's own installer (`roborev agent-hook install`) owns the harness hooks and its bundled skills (`roborev-fix`, …); this plugin never duplicates them.

## Checks

```bash
claude plugin validate .            # manifest + layout ("No version specified" is expected)
bash -n bin/roborev-wait-push
shellcheck bin/roborev-wait-push
```

All must pass before committing.

## Testing and shipping

1. Run the checks.
2. Run `claude --plugin-dir .` and load the skill.
3. From a repo roborev reviews, run this checkout's script by its absolute path: `<this-repo>/bin/roborev-wait-push <pushed-sha>`. It only lists and waits on jobs, so it changes nothing.
4. Open a PR; `main` is protected, and each merge is a release.
5. If the `plugin.json` description changed, open a matching PR in warmlogic/ai-plugin-marketplace.

## Project layout

```text
bin/roborev-wait-push              # on the Bash tool's PATH while the plugin is enabled
skills/working-with-roborev/SKILL.md
```

## Non-obvious design decisions

- **The script's exit codes are a contract**, documented in its header and in the skill's exit-code table, which Claude acts on. Change them only together, and keep the one final `roborev-wait-push: <result> (exit N)` line on every path, which an `EXIT` trap prints.
- **One listing per decision.** The script reads the job's final state and the branch's other open FAIL and unfinished jobs from the same `roborev list --all-branches --json` call, so the daemon is asked once after the wait.
- **roborev source citations** in the script and skill name a file and function at the roborev version they were checked against (`roborev version`); recheck them after an upgrade.

## Unversioned

No `version` field, here or in the marketplace catalog entry: Claude Code keys an unversioned install by the source's commit SHA, so every merge to `main` is a release. The catalog entry's `description` must equal `.claude-plugin/plugin.json`'s, verbatim.
