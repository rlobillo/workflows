# /report - Generate Comprehensive Triage Status Report

## Purpose

Produces a full triage dashboard with three distinct tables covering untriaged issues,
current-sprint work, and future-sprint work. Includes field suggestions, customer flags,
recent activity summaries, and linked PR status.

## Prerequisites

- Jira MCP server must be configured and accessible

## Process

1. **Fetch issues from Jira**

   Run three JQL queries to populate each table:

   **Query A — Untriaged issues** (any of the 5 triage fields missing):

   ```
   project = "OpenShift Virtualization" AND component = "CNV Install, Upgrade and Operators" AND (type = Bug OR type = Vulnerability OR type = Weakness) AND status not in (Closed, Verified) AND (assignee is EMPTY OR "QA Contact" is EMPTY OR sprint not in (openSprints(), futureSprints()) OR priority = Undefined OR fixVersion is EMPTY OR assignee = 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 OR "QA Contact" = 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9) ORDER BY createdDate ASC
   ```

   **Query B — Triaged, current sprint** (all 5 fields populated, in an open sprint, not resolved):

   ```
   project = "OpenShift Virtualization" AND component = "CNV Install, Upgrade and Operators" AND (type = Bug OR type = Vulnerability OR type = Weakness) AND status not in (Closed, Verified) AND sprint in openSprints() AND assignee is not EMPTY AND "QA Contact" is not EMPTY AND priority != Undefined AND fixVersion is not EMPTY AND assignee != 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 AND "QA Contact" != 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 ORDER BY priority ASC
   ```

   **Query C — Triaged, future sprint** (all 5 fields populated, in a future sprint, not resolved):

   ```
   project = "OpenShift Virtualization" AND component = "CNV Install, Upgrade and Operators" AND (type = Bug OR type = Vulnerability OR type = Weakness) AND status not in (Closed, Verified) AND sprint in futureSprints() AND sprint not in openSprints() AND assignee is not EMPTY AND "QA Contact" is not EMPTY AND priority != Undefined AND fixVersion is not EMPTY AND assignee != 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 AND "QA Contact" != 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 ORDER BY priority ASC
   ```

2. **Enrich each issue with extra context**

   For every issue across all three queries, gather:

   - **Customer flag**: check if the reporter is external (non-Red Hat) or if the issue
     has a "customer" label/flag or was created via a support case link. Mark as "Customer: Yes/No".
   - **Recent activity (last 7 days)**: scan comments, status transitions, and field changes.
     Summarize the type of activity (e.g., "comment by dev", "status changed to ON_QE",
     "fix version updated", "no activity").
   - **Linked PRs**: use `getJiraIssueRemoteIssueLinks` or `getTeamworkGraphContext` to find
     GitHub pull requests linked to the issue. For each PR report:
     - PR URL (as clickable link)
     - Status: Open / Merged / Closed
     - Recent human activity: yes/no and brief description (ignore bot-only activity like
       CI checks, auto-labels). Focus on human reviews, comments, and commits.

3. **Build Table 1 — Untriaged Issues**

   Columns:

   | Issue | Summary | Missing Fields | Suggestions | Apply? |
   |-------|---------|----------------|-------------|--------|

   - **Issue**: clickable link `[CNV-XXXXX](https://redhat.atlassian.net/browse/CNV-XXXXX)`
   - **Summary**: issue title (truncated if long)
   - **Missing Fields**: comma-separated list (e.g., "Assignee, Sprint, Priority")
   - **Suggestions**: proposed values with brief reasoning for each missing field
   - **Apply?**: after presenting the table, ask the user if they want to apply any or all
     suggestions. If confirmed, use Jira MCP tools (`editJiraIssue`) to set the fields.
     Process one issue at a time, confirming before each update.

4. **Build Table 2 — Triaged, Current Sprint (pending resolution)**

   Columns:

   | Issue | Summary | Priority | Customer? | Activity (7d) | PRs | PR Activity |
   |-------|---------|----------|-----------|---------------|-----|-------------|

   - **Issue**: clickable link
   - **Summary**: issue title
   - **Priority**: current priority value
   - **Customer?**: Yes/No
   - **Activity (7d)**: brief description of any activity in the last 7 days, or "None"
   - **PRs**: linked PR URLs (clickable) with status (Open/Merged/Closed), or "None"
   - **PR Activity**: recent human activity on linked PRs, or "None"

5. **Build Table 3 — Triaged, Future Sprint (pending resolution)**

   Same columns as Table 2:

   | Issue | Summary | Priority | Customer? | Activity (7d) | PRs | PR Activity |
   |-------|---------|----------|-----------|---------------|-----|-------------|

6. **Executive summary**

   Above the tables, show a brief summary:
   - Total issues in scope
   - Untriaged count
   - Current sprint count (and how many are customer-reported)
   - Future sprint count (and how many are customer-reported)
   - Issues with no activity in the last 7 days (stale count)
   - Issues with open PRs awaiting review

7. **Save and present**
   - Save the full report to `artifacts/cnv-bug-triage/full-report.md`
   - Present all three tables inline in the conversation
   - After Table 1, ask if the user wants to apply any suggestions

## Output

- **Full report**: `artifacts/cnv-bug-triage/full-report.md`
  - Executive summary + three tables with all columns described above

## Usage Examples

Generate the full report:

```
/report
```

## Success Criteria

After running this command:

- [ ] Three separate tables generated (untriaged, current sprint, future sprint)
- [ ] All issue keys rendered as clickable Jira links
- [ ] Customer flag checked for every issue
- [ ] Last 7 days activity summarized for every issue
- [ ] Linked PRs identified with status and human activity
- [ ] Suggestions provided for untriaged issues with option to apply
- [ ] Report saved to `artifacts/cnv-bug-triage/full-report.md`

## Next Steps

After reviewing the report:

1. Apply suggested triage values for untriaged issues (the agent will ask)
2. Follow up on stale issues with no recent activity
3. Review open PRs that need attention
4. Re-run `/report` after making changes to track progress

## Notes

- The "Apply?" column is interactive — the agent will ask before modifying any Jira field
- PR detection relies on issues having proper remote links in Jira; unlinked PRs won't appear
- Customer detection heuristics: reporter domain, "customer" label, support case links
- Activity scanning looks at comments, status changes, and field updates from the last 7 days
- Human PR activity filters out bot actions (CI, auto-merge, label bots) to surface real engagement
