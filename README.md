# linear-forge

Linear-driven development workflow — issue to merge in one command.

A [Claude Code](https://claude.ai/claude-code) plugin that provides a complete Linear-driven development workflow. Pick an issue, and linear-forge handles the full lifecycle: fetch context, create branch, implement, PR, review, test, merge, and update Linear.

## Installation

```
/plugin install linear-forge@orther/linear-forge
```

Then run the setup wizard:

```
/linear-forge:setup
```

## Skills

| Skill | Description |
|-------|-------------|
| `/linear-forge:work <issue>` | Full issue-to-merge lifecycle for a single issue |
| `/linear-forge:status` | Team-wide status overview — active work, priorities, blockers |
| `/linear-forge:project-status <project>` | Deep project view with dependency graph |
| `/linear-forge:work-parallel <issues...>` | Work multiple issues concurrently in isolated worktrees |
| `/linear-forge:project-work <project>` | Work through an entire project respecting dependency order |
| `/linear-forge:setup` | Interactive first-run configuration wizard |

## Agents

| Agent | Description |
|-------|-------------|
| `pr-reviewer` | Code review in worktree isolation — lint, type-check, security |
| `test-runner` | Test plan execution in worktree isolation — run suite, fix failures |

## Configuration

Configuration is set per-project via `userConfig` when you enable the plugin:

| Config | Required | Default | Description |
|--------|----------|---------|-------------|
| `linear_team` | Yes | — | Linear team name |
| `linear_team_key` | Yes | — | Team key prefix (e.g., `RES`) |
| `default_project` | No | none | Default project scope |
| `branch_prefix` | No | lowercase team key | Git branch prefix |
| `base_branch` | No | `main` | PR target branch |
| `test_command` | No | auto-detect | Test runner command |
| `lint_command` | No | auto-detect | Linter command |
| `merge_strategy` | No | `squash` | PR merge: squash/merge/rebase |
| `max_parallel_agents` | No | `5` | Max concurrent subagents |
| `cleanup_commands` | No | none | Post-subagent cleanup commands |

## Short Aliases

Create personal aliases in `~/.claude/skills/` for shorter commands:

```markdown
<!-- ~/.claude/skills/lw/SKILL.md -->
---
name: lw
description: Linear work (alias for linear-forge:work)
arguments:
  - name: issue
    required: true
---
Run `/linear-forge:work ${issue}`
```

## License

MIT
