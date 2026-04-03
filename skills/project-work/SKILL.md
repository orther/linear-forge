---
name: project-work
description: Work through all remaining issues in a Linear project, respecting dependency order
arguments:
  - name: project
    description: "Project name or keyword"
    required: true
---

Work through all remaining issues in a Linear project, automatically parallelizing where possible and respecting dependency order.

**Team**: All issue queries and updates are scoped to the **${user_config.linear_team}** (`${user_config.linear_team_key}`) team.

## Instructions

### Phase 1: Build the Work Queue

1. **Find the project**: Use `linear_list_projects` with `team: "${user_config.linear_team}"` and find the project matching `${project}`.

2. **Fetch all open issues**: Use `linear_list_issues` with the project filter. Exclude issues with status "Done" or "Cancelled".

3. **Build the dependency graph**: Map `blockedBy` and `parentId` relationships between issues.

4. **Categorize issues**:
   - **Ready**: No open blockers, no "Human Required" label — can be worked now
   - **Blocked**: Waiting on other open issues — work later when unblocked
   - **Human Required**: Has "Human Required" label — skip, report to user

5. **Report the plan to the user**:
   - Show total open issues, how many are ready, blocked, and human-required
   - Show which ready issues will be worked in this batch
   - Show which blocked issues will be picked up as dependencies resolve
   - Ask the user to confirm before proceeding

### Phase 2: Work Ready Issues in Parallel

For each **ready** issue, spawn a separate **Agent** with `isolation: "worktree"`. Launch up to **${user_config.max_parallel_agents}** (default: 5) ready-issue agents in a **single message** so they run concurrently.

Each agent receives this prompt (with values substituted):

---

You are working on Linear issue `{ISSUE_ID}` for the ${user_config.linear_team} (`${user_config.linear_team_key}`) team as part of the `{PROJECT_NAME}` project.

**Workflow:**

Use the `/linear-forge:work` skill to execute the full issue lifecycle:

```
/linear-forge:work {ISSUE_ID}
```

This handles: fetching the issue, moving to In Progress, creating a feature branch (prefix: `${user_config.branch_prefix}`), implementing, creating a PR against `${user_config.base_branch}`, review, testing, merging (`${user_config.merge_strategy}`), and updating Linear.

**Important:**
- Do NOT scope-creep — create new Linear issues for any discovered work
- Never commit directly to `${user_config.base_branch}` — always use PR workflow
- If the `/linear-forge:work` skill is not yet available, follow the manual steps:
  1. Fetch the issue with `linear_get_issue`
  2. Move to In Progress with `linear_save_issue`
  3. Post a starting comment with your approach
  4. Create a feature branch from `${user_config.base_branch}`
  5. Implement the work described in the issue
  6. Create a PR with a test plan section
  7. Run tests (`${user_config.test_command}`) and lint (`${user_config.lint_command}`)
  8. Merge via `${user_config.merge_strategy}` if all checks pass
  9. Move to Done and post a completion comment with the PR link

**Post-agent cleanup:**
${user_config.cleanup_commands}

---

### Phase 3: Re-evaluate and Continue

After all agents in the current batch complete:

1. **Report results** — summarize which issues succeeded, failed, or timed out
2. **Re-fetch open issues** for the project
3. **Re-evaluate the dependency graph** — blocked issues may now be ready
4. **If new issues are now ready** — present the next batch and ask the user to confirm before spawning
5. **If only blocked or human-required issues remain** — stop and report what's left
6. **If all issues are done** — report completion

### Stopping Conditions

- All project issues are Done or Cancelled
- Only "Human Required" issues remain
- A blocking issue requires human input
- The user asks to stop

### Safety Rules

- Maximum **${user_config.max_parallel_agents}** (default: 5) concurrent subagents at a time
- Always confirm with the user before starting each batch
- Never force-merge a PR that has failing tests
- If an agent times out, report it and move on
