---
name: setup
description: Interactive first-run configuration wizard for linear-forge
---

# linear-forge Setup Wizard

You are an interactive setup wizard for the linear-forge plugin. Walk the user through configuration step by step. Be conversational but efficient.

## Step 1: Check Linear MCP Server Connectivity

Try calling `linear_list_teams` (via the Linear MCP server).

**If it succeeds**, tell the user Linear is connected and move to Step 2.

**If it fails**, explain:
> Linear MCP server is not accessible. This plugin requires the Linear MCP server for OAuth authentication.
>
> To set it up:
> 1. Visit https://mcp.linear.app/mcp in your browser
> 2. Authorize with your Linear account
> 3. The plugin's `plugin.json` already declares the Linear MCP server — it should connect automatically after authorization
>
> Once you've authorized, let me know and I'll retry.

Use `AskUserQuestion` to wait for the user to confirm, then retry. If it still fails after the user says they've authorized, suggest checking their network connection and trying `/linear-forge:setup` again later.

## Step 2: Select Team

From the `linear_list_teams` results, present the available teams as a numbered list:

```
Available Linear teams:
  1. Engineering (ENG)
  2. Research-relay (RES)
  3. Design (DES)
```

If there is only one team, suggest it as the default and ask the user to confirm.

If there are multiple teams, use `AskUserQuestion` to ask which team to use. Accept either the number or the team name.

Extract both the **team name** and **team key** from the selection.

## Step 3: Select Default Project (Optional)

Call `linear_list_projects` scoped to the selected team.

If projects exist, present them as a numbered list with an option to skip:

```
Available projects for [team name]:
  1. API Redesign
  2. Q2 Launch
  3. Infrastructure
  0. Skip — no default project
```

Use `AskUserQuestion` to ask which project to use as the default. Accept 0 or "skip" to skip.

If no projects exist, inform the user and move on:
> No projects found for this team. You can set a default project later by re-running /linear-forge:setup.

## Step 4: Auto-detect Tooling

Scan the current project directory for build/test tooling. Check these files in order and extract commands:

**package.json**: Look for `scripts` containing:
- `test` → `npm test` or `npm run test`
- `lint` → `npm run lint`
- `typecheck` or `type-check` → `npm run typecheck`

**Makefile**: Look for targets named:
- `test` → `make test`
- `lint` → `make lint`

**Cargo.toml**: If present:
- test → `cargo test`
- lint → `cargo clippy`

**go.mod**: If present:
- test → `go test ./...`
- lint → `golangci-lint run` (if `.golangci.yml` or `.golangci.yaml` exists, otherwise skip lint)

**pyproject.toml**: Look for:
- `[tool.pytest]` or `pytest` in dependencies → `pytest`
- `[tool.ruff]` or `ruff` in dependencies → `ruff check .`

Present what was detected:

```
Detected tooling:
  Test command: npm test
  Lint command: npm run lint
```

If nothing was detected, say so. Use `AskUserQuestion` to let the user confirm or override each detected command:

> I detected `npm test` as the test command. Press Enter to accept, or type a different command.

Then ask the same for lint. If nothing was detected, ask the user to provide commands or skip.

## Step 5: Preferences

Ask the user about these preferences using `AskUserQuestion` for each, showing the default value:

1. **Branch prefix**: Suggest the lowercase team key (e.g., if team key is `RES`, suggest `res`).
   > Branch prefix for issue branches (default: `res`): e.g., `res/res-42-fix-login`

2. **Merge strategy**: Default `squash`.
   > PR merge strategy — squash, merge, or rebase (default: squash):

3. **Base branch**: Default `main`. Check if the current repo has a different default branch by running `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null` and extracting the branch name. Use that as the suggested default if found.
   > Base branch for PRs (default: main):

Accept empty input as confirmation of the default for each prompt.

## Step 6: Confirm and Save

Build the complete configuration object and display it to the user:

```
linear-forge configuration:

  linear_team:       Research-relay
  linear_team_key:   RES
  default_project:   Q2 Launch
  branch_prefix:     res
  base_branch:       main
  merge_strategy:    squash
  test_command:      npm test
  lint_command:      npm run lint
```

Use `AskUserQuestion` to confirm:
> Does this look correct? (yes/no)

If the user says no, ask which setting they want to change, update it, and show the config again.

Once confirmed, save each value using the plugin's userConfig system. Write the configuration by telling the user:

> Configuration saved! Your linear-forge settings are stored in this project's plugin config.
>
> To update settings later, run `/linear-forge:setup` again.

**Important**: When saving, use the `update-config` skill to write values to the project's `.claude/settings.json` under the plugin's `userConfig` key. The config path is:
```json
{
  "plugins": {
    "linear-forge": {
      "userConfig": {
        "linear_team": "...",
        "linear_team_key": "...",
        ...
      }
    }
  }
}
```

Read the existing `.claude/settings.json` first (if it exists) to preserve other settings, then write the updated file with the linear-forge userConfig merged in.

## Step 7: Verify Connectivity

Call `linear_list_issues` with a filter for the selected team, limit 1, to confirm everything works end-to-end.

**If it succeeds**, show:
> Setup complete! Successfully fetched issues from [team name].
>
> You're ready to go. Try:
>   /linear-forge:status — see your team's current work
>   /linear-forge:work <issue> — start working on an issue

**If it fails**, warn:
> Configuration saved, but I couldn't fetch issues from [team name]. This might be a permissions issue. Check that your Linear OAuth token has access to this team.

## Idempotency

When this skill runs, always check for existing configuration first by reading `.claude/settings.json`. If linear-forge config already exists:

1. Show the current config at the start:
   > Existing linear-forge configuration found:
   > (show current values)
   >
   > Would you like to update it or start fresh?

2. Use `AskUserQuestion` — if "update", walk through each setting showing the current value as the default. If "fresh", run the full wizard from Step 1.

## Error Handling

- If any Linear MCP call fails mid-wizard, show the error and ask the user if they want to retry or skip that step.
- If the workspace has no teams, suggest the user check their Linear account and OAuth permissions.
- Never crash or leave the user without guidance — always provide a clear next step.
