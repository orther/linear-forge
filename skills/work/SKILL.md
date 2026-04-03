---
name: work
description: Start a focused work session from a Linear issue — full issue-to-merge lifecycle
arguments:
  - name: issue
    description: "Issue identifier (e.g., '42' or 'RES-42')"
    required: true
---

Start a focused development session driven by a Linear issue.

## Issue Resolution

Resolve the issue identifier from `$ARGUMENTS`:

- If the input is a bare number (e.g., `42`), prefix it with `${user_config.linear_team_key}-` to form the full identifier (e.g., `RES-42`).
- If it already contains a prefix (e.g., `RES-42`), use it as-is.

## Instructions

### 1. Find the Issue

Search Linear for the resolved issue identifier using `linear_get_issue` (if an identifier like `${user_config.linear_team_key}-42` is provided) or `linear_list_issues` with `team: "${user_config.linear_team}"`.

Read the full context including description, comments, parent/child issues, and labels. Summarize the requirements before proceeding.

### 2. Move to In Progress

Update the issue status to "In Progress" via `linear_save_issue`.

### 3. Post a Starting Comment

Add a comment to the issue via `linear_save_comment`:
> Starting work on this. Approach: [brief plan based on requirements]

### 4. Implement

Work on the task, following the requirements from the issue. As you work:

- **Branch naming**: Create a feature branch using the pattern `${user_config.branch_prefix}/<issue-number>-<short-description>` (e.g., `res/134-port-work-skill`). The `branch_prefix` defaults to the lowercase team key if not configured.
- **Commit messages**: Reference the issue identifier in all commit messages (e.g., `${user_config.linear_team_key}-42: add retry logic`).
- **Progress updates**: Post comments on the Linear issue when making significant decisions or hitting milestones.
- **Scope control**: Create new Linear issues for any discovered work — do not scope-creep the current issue.

### 5. Create a PR

When implementation is complete:

- Push all changes to the remote.
- Create a PR targeting `${user_config.base_branch}` (default: `main`).
- Reference the Linear issue in the PR body (e.g., "Resolves ${user_config.linear_team_key}-42").
- Include a `## Test plan` section in the PR description with specific items to verify.
- Mark the PR as ready for review.

### 6. PR Review Agent

Spawn a subagent (`isolation: "worktree"`) using the `pr-reviewer` agent to review the PR:

- Read the full PR diff.
- Check for lint errors, type errors, and code quality issues.
- Fix any issues it can and push follow-up commits.
- Report back what was found and fixed.

### 7. Test Plan Agent

Spawn a subagent (`isolation: "worktree"`) using the `test-runner` agent to execute the test plan:

- Run through each test plan item from the PR description.
- Run the project's test suite using `${user_config.test_command}` (if not configured, auto-detect by checking for `package.json` scripts, `Makefile` targets, or common test runners).
- Fix any test failures it can and push follow-up commits.
- Report back pass/fail status for each item.

### 8. Merge or Report

Evaluate the results from the review and test agents:

- **If both pass cleanly**: Merge the PR using `${user_config.merge_strategy}` (default: `squash`).
- **If there are unfixable issues**: Report them to the user. Do **not** merge. Leave the PR open for human attention.

Post a summary comment on the Linear issue with the PR link and outcome.

### 9. Cleanup

If `${user_config.cleanup_commands}` is configured, run those commands after the agents complete (e.g., removing generated files, resetting local state).

### 10. Update Linear

Based on the outcome:

- **If merged**: Move the issue to "Done" via `linear_save_issue`.
- **If blocked**: Leave the issue as "In Progress" and post a comment explaining what needs human attention.
- List any new issues that were created during the session in the final summary.
