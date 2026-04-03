---
name: work-parallel
description: Work on multiple Linear issues in parallel — each in its own isolated subagent
arguments:
  - name: issues
    description: "Space-separated issue identifiers (e.g., '42 43 44' or 'RES-42 RES-43')"
    required: true
---

Work on multiple Linear issues simultaneously, each in its own isolated subagent.

## Arguments

`$ARGUMENTS` — Space-separated Linear issue identifiers (e.g., `42 46 50` or `${user_config.linear_team_key}-42 ${user_config.linear_team_key}-46`)

Issue identifiers can be bare numbers (auto-prefixed with `${user_config.linear_team_key}-`) or full identifiers.

## Instructions

For each issue identifier provided in `$ARGUMENTS`, spawn a **separate Agent** (using `isolation: "worktree"`) to work on it **in parallel**. All agents should be launched in a single message so they run concurrently.

Respect the `${user_config.max_parallel_agents}` limit (default: 5). If more issues are provided than the limit, batch them — launch the first batch, wait for completion, then launch the next.

Each agent should receive this prompt (with the issue ID substituted):

---

You are working on Linear issue `{ISSUE_ID}` for the **${user_config.linear_team}** (`${user_config.linear_team_key}`) team.

**Steps:**

1. **Fetch the issue** using `linear_get_issue` with identifier `{ISSUE_ID}`. Read the full description, comments, labels, and project context.

2. **Move to In Progress** using `linear_save_issue` to update the status to "In Progress".

3. **Post a starting comment** on the issue: "Starting work on this. Approach: [brief plan based on issue description]"

4. **Create a feature branch** named `${user_config.branch_prefix}-{number}-{slug}` (e.g., `${user_config.branch_prefix}-22-lot-variant-linking`) off of `${user_config.base_branch}`.

5. **Implement the work** described in the issue:
   - Follow existing codebase patterns
   - Write tests for new functionality
   - Reference the issue ID in commit messages (e.g., `{ISSUE_ID}: descriptive message`)
   - Keep changes focused on the issue scope

6. **Create a PR** targeting `${user_config.base_branch}` when implementation is done:
   - Reference the Linear issue in the PR body
   - Include a test plan section in the PR description
   - Mark the PR as ready for review

7. **PR Review**: Spawn the `pr-reviewer` agent to review the PR:
   - Read the PR diff and check for lint, type errors, code quality issues
   - Fix any issues it can and push follow-up commits
   - Report back what was found and fixed

8. **Test Plan Execution**: Spawn the `test-runner` agent to run the test plan:
   - Execute each test plan item from the PR description
   - Run the project's test suite (`${user_config.test_command}`)
   - Fix any test failures it can and push follow-up commits
   - Report back pass/fail for each item

9. **Merge or Report**:
   - If review and tests both pass: merge the PR using `${user_config.merge_strategy}` strategy (default: squash)
   - If unfixable issues exist: report them, do NOT merge

10. **Post-work cleanup**: If `${user_config.cleanup_commands}` is configured, run those commands.

11. **Update the Linear issue**:
    - If merged: move to "Done", post summary comment with PR link
    - If blocked: leave as "In Progress", explain what needs human attention
    - Note any follow-up items or discovered work

**Important:**
- Do NOT scope-creep — create new Linear issues for discovered work
- Follow the project's PR-based workflow (never commit directly to `${user_config.base_branch}`)

---

After all agents complete, summarize the results:

- **Completed**: issues that were merged with PR links
- **Blocked**: issues that need human attention and why
- **Follow-ups**: any new issues or discovered work
