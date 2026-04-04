# linear-forge

Linear-driven development workflow -- issue to merge in one command.

A [Claude Code](https://claude.ai/claude-code) plugin that provides a complete Linear-driven development workflow. Pick an issue, and linear-forge handles the full lifecycle: fetch context, create branch, implement, PR, review, test, merge, and update Linear.

linear-forge requires a [Linear MCP server](https://mcp.linear.app/mcp) to be configured in your project or globally. The plugin does not bundle its own MCP server, so it works with any Linear MCP setup (OAuth, API key, devenv, etc.).

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration Reference](#configuration-reference)
- [Skills Reference](#skills-reference)
  - [/linear-forge:work](#linear-forgework)
  - [/linear-forge:status](#linear-forgestatus)
  - [/linear-forge:project-status](#linear-forgeproject-status)
  - [/linear-forge:work-parallel](#linear-forgework-parallel)
  - [/linear-forge:project-work](#linear-forgeproject-work)
  - [/linear-forge:setup](#linear-forgesetup)
- [Agents](#agents)
  - [PR Reviewer](#pr-reviewer)
  - [Test Runner](#test-runner)
- [Short Aliases](#short-aliases)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Installation

```
/plugin install linear-forge@orther/linear-forge
```

This installs the plugin. You must have a Linear MCP server configured separately (e.g., via `devenv.nix`, `.claude/settings.json`, or global MCP config). On first use, Linear will prompt you to authorize via OAuth in your browser.

## Quick Start

1. **Install the plugin** (see above).

2. **Run the setup wizard** to configure your team and tooling:

   ```
   /linear-forge:setup
   ```

   The wizard will:
   - Verify Linear MCP connectivity
   - Let you pick your Linear team and default project
   - Auto-detect your test and lint commands
   - Set branch prefix, merge strategy, and base branch

3. **Start working on an issue**:

   ```
   /linear-forge:work 42
   ```

   This fetches issue RES-42 (or whatever your team key is), moves it to In Progress, creates a feature branch, implements the work, opens a PR, runs review and tests, merges, and moves the issue to Done.

4. **Check team status** at any time:

   ```
   /linear-forge:status
   ```

## Configuration Reference

Configuration is set per-project via `userConfig` when you enable the plugin. The setup wizard (`/linear-forge:setup`) handles this interactively, but you can also edit `.claude/settings.json` directly:

```json
{
  "plugins": {
    "linear-forge": {
      "userConfig": {
        "linear_team": "Research-relay",
        "linear_team_key": "RES",
        "default_project": "Q2 Launch",
        "branch_prefix": "res",
        "base_branch": "main",
        "test_command": "npm test",
        "lint_command": "npm run lint",
        "merge_strategy": "squash",
        "max_parallel_agents": 5,
        "cleanup_commands": "find src -name CLAUDE.md -delete"
      }
    }
  }
}
```

| Config | Required | Default | Description |
|--------|----------|---------|-------------|
| `linear_team` | Yes | -- | Linear team name (e.g., `Research-relay`, `Engineering`) |
| `linear_team_key` | Yes | -- | Team key prefix for issue identifiers (e.g., `RES`, `ENG`) |
| `default_project` | No | none | Default Linear project to scope work to |
| `branch_prefix` | No | lowercase team key | Git branch prefix (e.g., `res` produces `res/42-fix-login`) |
| `base_branch` | No | `main` | Target branch for PRs |
| `test_command` | No | auto-detect | Test runner command (e.g., `npm test`, `go test ./...`, `cargo test`) |
| `lint_command` | No | auto-detect | Linter command (e.g., `npm run lint`, `golangci-lint run`) |
| `merge_strategy` | No | `squash` | PR merge method: `squash`, `merge`, or `rebase` |
| `max_parallel_agents` | No | `5` | Maximum concurrent subagents for parallel work |
| `cleanup_commands` | No | none | Shell commands to run after subagent work completes |

### Auto-detection

When `test_command` or `lint_command` are not set, the plugin detects them from your project files:

| Project File | Test Command | Lint Command |
|-------------|-------------|-------------|
| `package.json` | `npm test` | `npm run lint` |
| `Makefile` | `make test` | `make lint` |
| `Cargo.toml` | `cargo test` | `cargo clippy` |
| `go.mod` | `go test ./...` | `golangci-lint run` |
| `pyproject.toml` | `pytest` | `ruff check .` |

## Skills Reference

### /linear-forge:work

**Full issue-to-merge lifecycle for a single issue.**

```
/linear-forge:work 42
/linear-forge:work RES-42
```

This is the core skill. It takes a Linear issue through the complete development cycle:

1. **Fetch** -- Looks up the issue in Linear, reads description, comments, labels, and related issues. Summarizes requirements before starting.
2. **Move to In Progress** -- Updates the issue status in Linear.
3. **Post starting comment** -- Adds a comment to the issue with the planned approach.
4. **Implement** -- Creates a feature branch (`{prefix}/{number}-{slug}`), writes code, commits with issue references, and creates new Linear issues for any discovered out-of-scope work.
5. **Create PR** -- Pushes changes, opens a PR against the base branch with a test plan section and Linear issue reference.
6. **PR Review** -- Spawns the [pr-reviewer](#pr-reviewer) agent in worktree isolation to lint, type-check, and review the diff. Auto-fixes trivial issues.
7. **Test Execution** -- Spawns the [test-runner](#test-runner) agent in worktree isolation to run the test plan and full test suite. Fixes minor test issues.
8. **Merge or Report** -- If review and tests pass, merges the PR using the configured strategy. If issues remain, leaves the PR open and reports what needs human attention.
9. **Cleanup** -- Runs configured cleanup commands.
10. **Update Linear** -- Moves the issue to Done (if merged) or leaves it In Progress with a comment explaining blockers.

**Issue resolution**: Bare numbers (e.g., `42`) are automatically prefixed with the configured team key (e.g., `RES-42`). Full identifiers like `RES-42` are used as-is.

---

### /linear-forge:status

**Team-wide status overview.**

```
/linear-forge:status
```

Queries Linear for a comprehensive view of your team's current state:

- **Active work** -- Issues currently In Progress
- **Backlog priorities** -- Highest priority items ready to be picked up
- **Projects** -- Project-level progress and completion percentages
- **My issues** -- Issues assigned to you
- **Recommendations** -- Suggests the next issue to pick up based on priority

No arguments required. Uses the configured `linear_team` to scope all queries.

---

### /linear-forge:project-status

**Deep project view with dependency graph and actionable breakdown.**

```
/linear-forge:project-status "Q2 Launch"
/linear-forge:project-status Infrastructure
```

Provides a detailed report for a specific project:

- **Progress** -- Done/total issues with percentage
- **In Progress** -- Currently active issues
- **Ready to Work** -- Unblocked issues sorted by priority, with labels
- **Blocked** -- Issues waiting on dependencies, showing the blocking chain
- **Human Required** -- Issues needing manual intervention
- **Recently Completed** -- Latest finished work
- **Recommended Next** -- Which ready issues to pick up, noting which can be parallelized vs. must be sequential

---

### /linear-forge:work-parallel

**Work multiple issues concurrently in isolated worktrees.**

```
/linear-forge:work-parallel 42 43 44
/linear-forge:work-parallel RES-42 RES-43 RES-44
```

Spawns a separate subagent for each issue, each running in its own git worktree for full isolation. All agents launch simultaneously and run the same lifecycle as `/linear-forge:work`.

Respects `max_parallel_agents` (default: 5). If you provide more issues than the limit, they are batched -- the first batch completes before the next launches.

After all agents finish, a summary shows:

- **Completed** -- Issues merged with PR links
- **Blocked** -- Issues needing human attention and why
- **Follow-ups** -- New issues or discovered work

---

### /linear-forge:project-work

**Automated full-project execution with dependency ordering.**

```
/linear-forge:project-work "Q2 Launch"
/linear-forge:project-work Infrastructure
```

Works through an entire project's open issues automatically:

1. **Build work queue** -- Fetches all open issues, maps dependencies, and categorizes them as ready, blocked, or human-required.
2. **Present plan** -- Shows what will be worked on and asks for confirmation before proceeding.
3. **Work ready issues** -- Launches parallel subagents (up to `max_parallel_agents`) for all unblocked issues.
4. **Re-evaluate** -- After each batch completes, re-fetches the dependency graph. Previously blocked issues may now be ready.
5. **Continue or stop** -- Presents the next batch for confirmation. Stops when all issues are done, only human-required issues remain, or a blocker requires human input.

Safety rules:
- Always confirms with the user before starting each batch
- Never force-merges a PR with failing tests
- Reports timeouts and failures without halting the entire run

---

### /linear-forge:setup

**Interactive configuration wizard.**

```
/linear-forge:setup
```

Walks through first-time setup step by step:

1. **Verify Linear connectivity** -- Tests the MCP server connection. Guides you through OAuth authorization if needed.
2. **Select team** -- Lists available Linear teams and lets you pick one.
3. **Select default project** -- Optionally pick a project to scope work to.
4. **Auto-detect tooling** -- Scans for `package.json`, `Makefile`, `Cargo.toml`, `go.mod`, `pyproject.toml` and extracts test/lint commands.
5. **Set preferences** -- Branch prefix, merge strategy, base branch.
6. **Confirm and save** -- Shows the full config and writes it to `.claude/settings.json`.
7. **Verify** -- Fetches an issue to confirm end-to-end connectivity.

The wizard is idempotent. Running it again shows existing configuration and offers to update individual settings or start fresh.

## Agents

Agents are spawned by skills during the PR lifecycle. They run in worktree isolation so they cannot interfere with your working directory.

### PR Reviewer

Spawned during `/linear-forge:work` (step 6) and `/linear-forge:work-parallel`.

The PR reviewer agent:

1. **Reads the full PR diff** using `gh pr diff`.
2. **Detects project tooling** -- Finds linters, type checkers, and formatters for JS/TS, Go, Rust, Python, Ruby, PHP, and more.
3. **Runs lint checks** -- Uses the configured `lint_command` or auto-detected linter.
4. **Runs type checks** -- TypeScript (`tsc --noEmit`), Go (`go vet`), Rust (`cargo check`), Python (`mypy`/`pyright`).
5. **Auto-fixes trivial issues** -- Formatting, import ordering, trailing whitespace. Commits and pushes fixes automatically. Never auto-fixes logic, types, or security issues.
6. **Reviews code quality** -- Evaluates correctness (nil dereference, error handling, race conditions), security (injection, hardcoded secrets, input validation), performance (N+1 queries, missing pagination), and design (single responsibility, test coverage).
7. **Reports findings** with a structured verdict: `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`.

### Test Runner

Spawned during `/linear-forge:work` (step 7) and `/linear-forge:work-parallel`.

The test runner agent:

1. **Parses the test plan** from the PR description's `## Test plan` section.
2. **Determines the test command** from config or auto-detection (supports npm, yarn, pnpm, make, cargo, go, pytest, mix, gradle, maven).
3. **Establishes a baseline** -- Runs the full test suite first to distinguish pre-existing failures from new regressions.
4. **Executes test plan items** -- Verifies each checkbox item, running the relevant tests or commands. Items requiring manual verification are marked as skipped.
5. **Runs the full test suite** -- Always runs even if all plan items pass.
6. **Fixes failing tests** -- Fixes straightforward issues (missing imports, assertion updates for intentional changes). Does not fix real bugs, architectural issues, or pre-existing failures.
7. **Reports results** with per-item status, test suite summary, fix commits, and overall `PASS`/`FAIL` verdict.

## Short Aliases

The full skill names are verbose. Create personal aliases in `~/.claude/skills/` for shorter commands.

### /lw -- Linear Work

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

### /ls -- Linear Status

```markdown
<!-- ~/.claude/skills/ls/SKILL.md -->
---
name: ls
description: Linear status (alias for linear-forge:status)
---
Run `/linear-forge:status`
```

### /lps -- Linear Project Status

```markdown
<!-- ~/.claude/skills/lps/SKILL.md -->
---
name: lps
description: Linear project status (alias for linear-forge:project-status)
arguments:
  - name: project
    required: true
---
Run `/linear-forge:project-status ${project}`
```

### /lwp -- Linear Work Parallel

```markdown
<!-- ~/.claude/skills/lwp/SKILL.md -->
---
name: lwp
description: Linear work parallel (alias for linear-forge:work-parallel)
arguments:
  - name: issues
    required: true
---
Run `/linear-forge:work-parallel ${issues}`
```

### /lpw -- Linear Project Work

```markdown
<!-- ~/.claude/skills/lpw/SKILL.md -->
---
name: lpw
description: Linear project work (alias for linear-forge:project-work)
arguments:
  - name: project
    required: true
---
Run `/linear-forge:project-work ${project}`
```

To create an alias, make the directory and file:

```bash
mkdir -p ~/.claude/skills/lw
# Then create SKILL.md with the content above
```

## Troubleshooting

### Linear MCP server not found

The plugin requires a Linear MCP server to be available. Configure one in your project (e.g., `devenv.nix`, `.claude/settings.json`) or globally. The server should point to `https://mcp.linear.app/mcp`.

### OAuth authorization

The Linear MCP server uses OAuth. On first use:

1. Claude Code will open a browser window to `mcp.linear.app`.
2. Sign in with your Linear account and authorize the application.
3. The OAuth token is stored by the MCP server -- you should not need to re-authorize unless you revoke access.

If authorization fails:
- Ensure you are signed into the correct Linear workspace in your browser.
- Check that your Linear account has access to the team you configured.
- Try visiting `https://mcp.linear.app/mcp` directly in your browser to re-authorize.

### Common errors

**"No team found matching..."**
Your `linear_team` config does not match any team in your Linear workspace. Run `/linear-forge:setup` to reconfigure, or check the exact team name in Linear settings.

**"Issue not found"**
The issue identifier does not exist or your OAuth token does not have access to it. Verify the issue exists in Linear and that you are using the correct team key prefix.

**"MCP server not responding"**
The Linear MCP server at `mcp.linear.app` may be temporarily unavailable. Wait a moment and retry. If persistent, check your network connection and any proxy/firewall settings.

**Agent timeout**
Subagents spawned by `/linear-forge:work-parallel` or `/linear-forge:project-work` may time out on large issues. The parent skill will report the timeout and continue with remaining work. Consider breaking large issues into smaller ones.

**Cleanup command failures**
If `cleanup_commands` fail, they are reported but do not block the workflow. Check that the commands are valid shell commands and that referenced paths exist.

## Contributing

Contributions are welcome. The plugin is structured as:

```
.claude-plugin/
  plugin.json          # Plugin manifest with userConfig schema
skills/
  work/SKILL.md        # /linear-forge:work
  status/SKILL.md      # /linear-forge:status
  project-status/SKILL.md  # /linear-forge:project-status
  work-parallel/SKILL.md   # /linear-forge:work-parallel
  project-work/SKILL.md    # /linear-forge:project-work
  setup/SKILL.md       # /linear-forge:setup
agents/
  pr-reviewer.md       # PR review agent
  test-runner.md       # Test execution agent
```

Skills are Markdown files with YAML frontmatter that define the skill name, description, and arguments. The body contains instructions that Claude Code follows when the skill is invoked.

Agents are Markdown files that define the behavior of subagents spawned in worktree isolation.

To test changes locally, use the plugin's development install:

```
/plugin install linear-forge@/path/to/local/clone
```

## License

MIT
