---
name: project-status
description: Deep status view for a specific Linear project — issue breakdown, dependency graph, blockers
arguments:
  - name: project
    description: "Project name or keyword"
    required: true
---

Provide a deep status view of a specific project in Linear.

**IMPORTANT**: All queries MUST include `team: "${user_config.linear_team}"` to scope results to the configured team only.

## Arguments

`$ARGUMENTS` — Project name or keyword (e.g., `AI Agent Workforce`, `Fulfillment`, `E2E Test`)

## Instructions

1. **Find the project**: Use `linear_list_projects` with `team: "${user_config.linear_team}"` and find the project matching `$ARGUMENTS`

2. **Fetch all issues**: Use `linear_list_issues` with `team: "${user_config.linear_team}"` and `project` set to the matched project name. Fetch up to 100 issues.

3. **Build the dependency graph**: For issues that have `blockedBy` or `parentId` relationships, map out the dependency tree.

4. **Categorize issues by status**:
   - **Done**: completed issues
   - **In Progress**: actively being worked on
   - **Ready** (actionable): backlog issues with NO open blockers and NO "Human Required" label
   - **Blocked**: backlog issues waiting on other open issues
   - **Human Required**: issues with the "Human Required" label that need manual intervention

5. **Present the report**:

   ### Project: {name}
   **Status**: {project status} | **Progress**: {done}/{total} issues ({percentage}%)

   #### In Progress
   - List currently active issues

   #### Ready to Work (unblocked)
   - List issues that could be started now, sorted by priority
   - For each: show ID, title, priority, labels

   #### Blocked
   - List blocked issues with what they're waiting on
   - Show the blocking chain

   #### Human Required
   - List issues needing manual intervention with why

   #### Completed
   - Count and list recently completed issues

   #### Recommended Next
   - Suggest which ready issues to pick up next based on priority and dependencies
   - Identify which issues can be parallelized vs. must be sequential
