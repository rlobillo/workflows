# /report - Generate Comprehensive Triage Status Report

## Purpose

Produces a full triage dashboard with three tables covering untriaged issues,
current-sprint work, and future-sprint work. Uses emoji icons instead of extra columns
to keep tables compact.

## Prerequisites

- Jira MCP server must be configured and accessible

## Emoji Legend

Show this legend at the top of every report:

```
Priority: 🔴 Blocker  🟠 Critical  🟡 Major  🟣 Normal  🔵 Minor  ⚪ Undefined
Flags:    👤 Customer-reported   ⚠️ Stale (no human activity in 7+ days)
```

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

2. **Enrich each issue**

   For every issue across all three queries, gather:

   - **Customer flag**: run a separate JQL query to identify customer-reported bugs:
     ```
     project = CNV AND type = Bug AND SFDC_Cases_Counter > 0 AND resolution is EMPTY AND component = "CNV Install, Upgrade and Operators"
     ```
     Cross-reference returned keys with the issues in each table. Mark matches with 👤.
   - **Recent activity (last 7 days)**: scan comments, status transitions, and field changes.
     **Ignore all bot-generated activity** (CI bots, auto-labelers, merge bots).
     Only report human actions. For field updates, mention what changed
     (e.g., "Priority: Major → Critical"). If no human activity in 7+ days, mark as stale.
   - **Link activity to Jira**: every activity entry should be a clickable link to the
     specific comment, transition, or changelog in Jira. Use the format:
     `[description](https://redhat.atlassian.net/browse/CNV-XXXXX?focusedId=COMMENT_ID)`
     for comments, or link to the issue activity tab for other changes.
   - **Linked PRs**: **always check BOTH sources** and merge results (deduplicate by URL):
     1. `getTeamworkGraphContext` with `detailLevel: "full"` and
        `relationshipTypes: ["jira-work-item-links-external-pull-request"]`
     2. `getJiraIssueRemoteIssueLinks` filtering for GitHub/GitLab PR URLs
     For each PR report:
     - PR URL (as clickable link)
     - Status: Open / Merged / Closed / Draft
     - Recent human activity only (reviews, comments, commits — ignore bot CI activity)

3. **Build Table 1 — Untriaged Issues**

   | Issue | Summary | Missing Fields | Suggestions |
   |-------|---------|----------------|-------------|

   - **Issue**: clickable link `[CNV-XXXXX](https://redhat.atlassian.net/browse/CNV-XXXXX)`
   - **Summary**: prepend emoji icons before the title:
     - Priority icon (🔴🟠🟡🔵⚪) — always first
     - 👤 if customer-reported
     - ⚠️ if stale (no human activity in 7+ days)
     - Then the issue title (truncated if long)
     - Example: `🟡 👤 ⚠️ OVN network policy not enforced after upgrade`
   - **Missing Fields**: comma-separated list (e.g., "Assignee, Sprint")
   - **Suggestions**: proposed values with brief reasoning for each missing field

4. **Build Table 2 — Triaged, Current Sprint (pending resolution)**

   | Issue | Summary | Activity (7d) | PRs |
   |-------|---------|---------------|-----|

   - **Issue**: clickable link
   - **Summary**: same icon-enriched format as Table 1 (priority + 👤 + ⚠️ + title)
   - **Activity (7d)**: brief description of human activity, linked to Jira.
     For field updates, mention the change. "None" if no human activity.
   - **PRs**: linked PR URLs (clickable) with status and recent human activity,
     or "None"

5. **Build Table 3 — Triaged, Future Sprint (pending resolution)**

   Same columns as Table 2:

   | Issue | Summary | Activity (7d) | PRs |
   |-------|---------|---------------|-----|

6. **Executive summary**

   Above the tables, show a brief summary:
   - Total issues in scope
   - Untriaged count (👤 X customer-reported)
   - Current sprint count (👤 X customer-reported, ⚠️ X stale)
   - Future sprint count (👤 X customer-reported, ⚠️ X stale)
   - Issues with open PRs awaiting human review

7. **Save and present**
   - Save the full report to `artifacts/cnv-bug-triage/full-report.md`
   - Present all three tables inline in the conversation

## Output

- **Full report**: `artifacts/cnv-bug-triage/full-report.md`
  - Emoji legend + executive summary + three tables

## Usage Examples

Generate the full report:

```
/report
```

## Success Criteria

After running this command:

- [ ] Three tables generated (untriaged, current sprint, future sprint)
- [ ] All issue keys rendered as clickable Jira links
- [ ] Priority/customer/stale icons embedded in Summary column
- [ ] Only human activity shown (bot activity filtered out)
- [ ] Field changes mention what changed specifically
- [ ] Activity entries linked to Jira for one-click access
- [ ] Linked PRs with status and human activity
- [ ] Report saved to `artifacts/cnv-bug-triage/full-report.md`

## Notes

- PR detection relies on issues having proper remote links in Jira; unlinked PRs won't appear
- Customer detection heuristics: reporter domain, "customer"/"CEE" labels, support case links
- Stale threshold: 7 days with no human activity (bot activity does not count)
- All activity links point to the specific Jira comment or changelog entry
