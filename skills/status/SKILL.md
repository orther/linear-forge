---
name: status
description: Review current Linear team status — active issues, blockers, and priorities
---

Provide a comprehensive status overview of the **${user_config.linear_team}** team in Linear.

## Instructions

1. **Active work**: Use `linear_list_issues` with `team: "${user_config.linear_team}"` and status "In Progress" to see what's currently being worked on

2. **Backlog priorities**: Use `linear_list_issues` with `team: "${user_config.linear_team}"` sorted by priority to show the most important upcoming work

3. **Projects**: Use `linear_list_projects` with `team: "${user_config.linear_team}"` to see project-level progress

4. **My issues**: Use `linear_list_issues` with `team: "${user_config.linear_team}"` and `assignee: "me"` to see issues assigned to the current user

5. **Summarize**: Present a clear status report:
   - What's currently in progress
   - Top priority items in the backlog
   - Project completion percentages
   - Any blockers or overdue items
   - Recommended next issue to pick up
