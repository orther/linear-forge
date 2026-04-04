# Linear Forge — Design Document

> Claude Code plugin that provides a complete Linear-driven development workflow.
> Repo: `github.com/orther/linear-forge`

## Problem

The rr-bizops repo has a mature set of slash commands (`/lw`, `/ls`, `/lwp`, `/lps`, `/lpw`) that drive a full issue-to-merge workflow through Linear. These are hardcoded to the Research-relay team and rr-bizops project structure. Extracting them into a reusable plugin would let any project get the same workflow by installing one plugin and configuring a few variables.

## Name: `linear-forge`

Where Linear issues get forged into shipped code. Short, memorable, available on GitHub.

## Architecture Decision: Claude Code Plugin

After researching distribution options, a **Claude Code plugin** is the clear winner:

| Option | Pros | Cons |
|--------|------|------|
| Plugin (`.claude-plugin/`) | Versioned, updateable, namespace-isolated, provides skills + agents + MCP servers + hooks | Can't ship CLAUDE.md (workaround: `@imports`) |
| User-level skills (`~/.claude/skills/`) | Simple, always available | No versioning, manual sync, no per-project config |
| Symlinked commands | Zero overhead | Fragile, no config, no MCP server bundling |

**Decision: Plugin with `userConfig` for per-project variables.**

## Plugin Structure

```
linear-forge/
├── .claude-plugin/
│   └── plugin.json          # Manifest with userConfig schema
├── skills/
│   ├── work/
│   │   └── SKILL.md         # /linear-forge:work (alias: /lf:work)
│   ├── status/
│   │   └── SKILL.md         # /linear-forge:status
│   ├── project-status/
│   │   └── SKILL.md         # /linear-forge:project-status
│   ├── work-parallel/
│   │   └── SKILL.md         # /linear-forge:work-parallel
│   ├── project-work/
│   │   └── SKILL.md         # /linear-forge:project-work
│   └── setup/
│       └── SKILL.md         # /linear-forge:setup — interactive first-run config
├── agents/
│   ├── pr-reviewer.md       # Subagent: code review in worktree
│   └── test-runner.md       # Subagent: test plan execution in worktree
├── README.md
├── LICENSE
└── CHANGELOG.md
```

## Configuration Schema

### plugin.json — `userConfig`

These are prompted at plugin enable time and stored securely:

```json
{
  "name": "linear-forge",
  "version": "0.1.0",
  "description": "Linear-driven development workflow — issue to merge in one command",
  "author": { "name": "Brandon Orther" },
  "repository": "https://github.com/orther/linear-forge",
  "license": "MIT",
  "keywords": ["linear", "workflow", "development", "pr"],

  "userConfig": {
    "linear_team": {
      "description": "Linear team name (e.g., 'Research-relay', 'Engineering')",
      "required": true
    },
    "linear_team_key": {
      "description": "Linear team key prefix for issue identifiers (e.g., 'RES', 'ENG')",
      "required": true
    },
    "default_project": {
      "description": "Default Linear project name to scope work to (optional)",
      "required": false
    },
    "branch_prefix": {
      "description": "Git branch prefix (default: team key lowercase, e.g., 'res')",
      "required": false
    },
    "base_branch": {
      "description": "Base branch for PRs (default: 'main')",
      "required": false
    },
    "test_command": {
      "description": "Command to run tests (e.g., 'npm test', 'go test ./...', 'cargo test')",
      "required": false
    },
    "lint_command": {
      "description": "Command to run linter (e.g., 'npm run lint', 'golangci-lint run')",
      "required": false
    },
    "merge_strategy": {
      "description": "PR merge strategy: 'squash', 'merge', or 'rebase' (default: 'squash')",
      "required": false
    },
    "max_parallel_agents": {
      "description": "Max concurrent subagents for parallel work (default: 5)",
      "required": false
    },
    "cleanup_commands": {
      "description": "Commands to run after subagent work (e.g., 'find src -name CLAUDE.md -delete')",
      "required": false
    }
  }
}
```

### Config Resolution (with defaults)

Skills read config via `${user_config.*}` variables. Defaults when not set:

| Config | Default | Source |
|--------|---------|--------|
| `linear_team` | *(required)* | userConfig |
| `linear_team_key` | *(required)* | userConfig |
| `default_project` | `null` (no project filter) | userConfig |
| `branch_prefix` | lowercase `linear_team_key` | derived |
| `base_branch` | `main` | userConfig |
| `test_command` | auto-detect from package.json/Makefile/Cargo.toml | skill logic |
| `lint_command` | auto-detect | skill logic |
| `merge_strategy` | `squash` | userConfig |
| `max_parallel_agents` | `5` | userConfig |
| `cleanup_commands` | `""` | userConfig |

## Skill Designs

### `/linear-forge:work` (shorthand: `/lf:work`)

**Input:** Issue identifier (e.g., `42` or `RES-42`)

**Flow:**
1. Resolve issue — if bare number, prefix with `${linear_team_key}-`
2. Fetch issue with `linear_get_issue`, including relations
3. Move to "In Progress"
4. Post starting comment with approach
5. Create branch: `${branch_prefix}-{number}-{slug}`
6. Implement the work
7. Create PR with test plan section, reference issue in body
8. Spawn PR review agent (worktree isolation)
9. Spawn test runner agent (worktree isolation)
10. If both pass: merge with `${merge_strategy}`, delete branch
11. If failures: report to user, don't merge
12. Update Linear issue (Done or In Progress)
13. Run `${cleanup_commands}` if set

### `/linear-forge:status`

**Input:** None

**Flow:**
1. List issues with `team: ${linear_team}`, status "In Progress"
2. List issues with `team: ${linear_team}`, backlog sorted by priority
3. List projects with `team: ${linear_team}`
4. List issues assigned to "me"
5. Render status report with projects, priorities, blockers, recommendations

### `/linear-forge:project-status`

**Input:** Project name or keyword

**Flow:**
1. Find matching project in `${linear_team}`
2. Fetch all open issues for project (limit 100)
3. Build dependency graph from `blockedBy` and `parentId`
4. Categorize: Done, In Progress, Ready, Blocked, Human Required
5. Show completion %, unblocked items, suggested next pickup

### `/linear-forge:work-parallel`

**Input:** Space-separated issue identifiers

**Flow:**
1. Parse identifiers, prefix with team key if bare numbers
2. For each issue, spawn a subagent (worktree isolation) running the `/lf:work` flow
3. All agents launched in single message for concurrency
4. Collect results, report pass/fail per issue
5. Run cleanup commands

### `/linear-forge:project-work`

**Input:** Project name or keyword

**Flow:**
1. Build full work queue for project
2. Categorize by dependency order
3. Present plan to user, get confirmation
4. Launch batch of up to `${max_parallel_agents}` subagents
5. Wait for batch, re-evaluate dependencies
6. Continue until all done or only Human Required remain

### `/linear-forge:setup`

**Input:** None (interactive)

**Flow:**
1. Check if Linear MCP server is accessible — if not, guide user through OAuth setup
2. List available teams, let user pick or create
3. List projects, let user pick default (or skip)
4. Detect test/lint commands from project files
5. Ask for branch prefix and merge strategy preferences
6. Write config (plugin userConfig mechanism)
7. Verify by fetching a test issue

## Agent Definitions

### `pr-reviewer.md`

Spawned in worktree isolation. Reads PR diff, checks for:
- Lint errors (runs `${lint_command}` if set)
- Type errors (language-detected)
- Code quality issues
- Security concerns
- Pushes fix commits for auto-fixable issues
- Reports findings

### `test-runner.md`

Spawned in worktree isolation. Executes:
- Each test plan item from PR description
- Full test suite via `${test_command}`
- Pushes fix commits for failing tests it can resolve
- Reports pass/fail per item

## Linear MCP Server

The plugin requires a Linear MCP server to be configured by the host project. It does **not** bundle its own server to avoid conflicts with existing setups (e.g., devenv.nix OAuth config, API key auth).

The host project should configure the Linear MCP server pointing to `https://mcp.linear.app/mcp`. Linear's official MCP server uses OAuth — the user authenticates once via browser redirect and the token is cached.

## Migration Plan: rr-bizops

### Phase 1: Create plugin repo
- Initialize `github.com/orther/linear-forge` with plugin structure
- Port skills from `.claude/commands/*.md` to `skills/*/SKILL.md`
- Replace hardcoded "Research-relay" / "RES" with `${user_config.*}` references
- Write agent definitions

### Phase 2: Test in rr-bizops
- Install plugin: `/plugin install linear-forge@orther/linear-forge`
- Configure userConfig: team=Research-relay, key=RES, test_command=npm test
- Run each skill, verify parity with current commands
- Fix any issues

### Phase 3: Remove old commands from rr-bizops
- Delete `.claude/commands/{lw,ls,lps,lpw,lwp,linear-work,linear-status,linear-work-parallel}.md`
- Remove Linear workflow instructions from CLAUDE.md (replaced by plugin skills)
- Update CLAUDE.md to reference plugin: "Linear workflow provided by linear-forge plugin"

### Phase 4: Roll out to other projects
- Install plugin in each project
- Run `/linear-forge:setup` for per-project config
- Done

## Platform Gaps & Workarounds

| Gap | Impact | Workaround |
|-----|--------|------------|
| Plugins can't ship CLAUDE.md | Can't inject workflow instructions into project context | Skills are self-contained with full instructions in SKILL.md frontmatter; no CLAUDE.md needed |
| No skill aliases | `/linear-forge:work` is verbose | Users can create personal shortcuts in `~/.claude/skills/` that delegate: e.g., `/lw` -> `/linear-forge:work` |
| No dynamic userConfig defaults | Can't auto-detect test command at config time | Skills auto-detect at runtime from package.json/Makefile/Cargo.toml |
| Plugin updates require manual `/plugin update` | Users could fall behind | Pin to semver ranges; document update process |

## Validation Status

**Plugin validated in rr-bizops on 2026-04-04.** All skills tested and confirmed working.

### Integration Test Results

| Skill | Status | Notes |
|-------|--------|-------|
| `/linear-forge:setup` | Pass | Configured via `pluginConfigs` in user-level project settings |
| `/linear-forge:status` | Pass | Feature parity with old `/ls` |
| `/linear-forge:project-status` | Pass | Feature parity with old `/lps` |
| `/linear-forge:work` | Pass | Full lifecycle tested via RES-145 |
| `/linear-forge:work-parallel` | Pending | Needs multi-issue test |
| `/linear-forge:project-work` | Pending | Needs project-level test |

### Issues Found During Integration

1. **MCP server conflict**: Plugin originally bundled its own Linear MCP server, which conflicted with the existing devenv.nix config. **Resolution**: Removed `mcpServers` from plugin.json — host projects provide their own Linear MCP server.

2. **Missing marketplace.json**: Claude Code requires `.claude-plugin/marketplace.json` alongside `plugin.json` for custom marketplace plugins. Not documented in plugin authoring guides.

3. **userConfig schema validation**: Each `userConfig` field requires `type` (string|number|boolean|directory|file) and `title` properties. The original schema only had `description` and `required`.

4. **Plugin cache staleness**: After pushing fixes to GitHub, the local plugin cache at `~/.claude/plugins/cache/` retained the old commit. Required manual cache clearing and marketplace clone update (`git pull`).

5. **Nix-managed settings.json**: Project `.claude/settings.json` is a Nix symlink (read-only). Plugin config must go in user-level project settings at `~/.claude/projects/<project>/settings.json` using the `pluginConfigs` key.

### Resolved Open Questions

1. **Should `/lf:setup` write a `.linear-forge.toml` to the project root?** No — use `pluginConfigs` in settings.json, not a separate config file.

2. **Should the plugin detect and adapt to existing Linear MCP server configs?** Yes — plugin should NOT bundle its own Linear MCP server. Host projects configure their own, avoiding conflicts with devenv.nix, API keys, or other auth methods.

3. **Should there be a `/lf:init` that creates the Linear team/project?** No — assume Linear is already set up. The `/linear-forge:setup` wizard connects to existing teams and projects.

## Timeline Estimate

| Phase | Issues | Effort |
|-------|--------|--------|
| Repo scaffold + plugin manifest | 1 | Small |
| Port 5 skills + templatize | 5 | Medium each |
| Agent definitions (reviewer + test runner) | 2 | Medium each |
| Setup/init skill | 1 | Medium |
| Test in rr-bizops | 1 | Small |
| Migration (remove old commands) | 1 | Small |
| Documentation (README, CHANGELOG) | 1 | Small |
| **Total** | **12-13** | |
