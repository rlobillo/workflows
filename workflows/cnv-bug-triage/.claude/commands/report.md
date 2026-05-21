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
Flags:    👤 Customer-reported   ⚠️ Stale (no human activity in 21+ days)
```

## Process

1. **Fetch issues from Jira**

   Run three JQL queries to populate each table:

   **Query A — Untriaged issues** (any of the 5 triage fields missing):

   ```
   project = "OpenShift Virtualization" AND component = "CNV Install, Upgrade and Operators" AND (type = Bug OR type = Vulnerability OR type = Weakness) AND status not in (Closed, Verified) AND (assignee is EMPTY OR "QA Contact" is EMPTY OR sprint not in (openSprints(), futureSprints()) OR sprint is EMPTY OR priority = Undefined OR fixVersion is EMPTY OR assignee = 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 OR "QA Contact" = 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9) ORDER BY createdDate ASC
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
   - **Recent activity (last 21 days)**: scan comments, status transitions, and field changes.
     **Ignore all bot-generated activity** (CI bots, auto-labelers, merge bots).
     Only report human actions. For field updates, mention what changed
     (e.g., "Priority: Major → Critical"). If no human activity in 21+ days, mark as stale.
   - **Link activity to Jira**: every activity entry should be a clickable link to the
     specific comment, transition, or changelog in Jira. Use the format:
     `[description](https://redhat.atlassian.net/browse/CNV-XXXXX?focusedId=COMMENT_ID)`
     for comments, or link to the issue activity tab for other changes.
   - **Linked PRs**: **always check BOTH sources** and merge results (deduplicate by URL):
     1. `getTeamworkGraphContext` with `detailLevel: "full"` and
        `relationshipTypes: ["jira-work-item-links-external-pull-request"]`
     2. `getJiraIssueRemoteIssueLinks` filtering for GitHub/GitLab PR URLs
     **You MUST call both APIs for every single issue — no shortcuts, no skipping.**
     If neither source returns any PR URLs, the PRs cell must be exactly "None".
     Never write "Verify in Jira" or any other fallback text.
     For each PR found, report:
     - PR URL (as clickable link)
     - Status: Open / Merged / Closed / Draft
     - Recent human activity only (reviews, comments, commits — ignore bot CI activity)

3. **Build Table 1 — Untriaged Issues**

   | Issue | Summary | Missing Fields | Suggestions | Triage |
   |-------|---------|----------------|-------------|--------|

   - **Issue**: clickable link `[CNV-XXXXX](https://redhat.atlassian.net/browse/CNV-XXXXX)`
   - **Summary**: prepend emoji icons before the title:
     - Priority icon (🔴🟠🟡🔵⚪) — always first
     - 👤 if customer-reported
     - ⚠️ if stale (no human activity in 21+ days)
     - Then the issue title (truncated if long)
     - Example: `🟡 👤 ⚠️ OVN network policy not enforced after upgrade`
   - **Missing Fields**: comma-separated list (e.g., "Assignee, Sprint")
   - **Suggestions**: proposed values with brief reasoning for each missing field
   - **Triage**: show the command to deep-dive and apply: `/triage-bug CNV-XXXXX`

4. **Build Table 2 — Triaged, Current Sprint (pending resolution)**

   | Issue | Status | Summary | Next Action | Activity (21d) | PRs |
   |-------|--------|---------|-------------|----------------|-----|

   - **Issue**: clickable link
   - **Status**: current Jira status (e.g., NEW, ASSIGNED, POST, MODIFIED, ON_QA)
   - **Summary**: same icon-enriched format as Table 1 (priority + 👤 + ⚠️ + title)
   - **Next Action**: who needs to act next, determined by this logic:
     - NEW, ASSIGNED, or POST → show the Assignee name
     - MODIFIED → "Waiting for fix to land"
     - ON_QA → show the QA Contact name
   - **Activity (21d)**: brief description of human activity, linked to Jira.
     For field updates, mention the change. "None" if no human activity.
   - **PRs**: linked PR URLs (clickable) with status and recent human activity,
     or "None"

5. **Build Table 3 — Triaged, Future Sprint (pending resolution)**

   Same columns as Table 2:

   | Issue | Status | Summary | Next Action | Activity (21d) | PRs |
   |-------|--------|---------|-------------|----------------|-----|

6. **Consistency check**

   Run a baseline query to get the total number of open bugs in the component:

   ```
   project = "OpenShift Virtualization" AND component = "CNV Install, Upgrade and Operators" AND (type = Bug OR type = Vulnerability OR type = Weakness) AND status not in (Closed, Verified)
   ```

   Compare: **Total from baseline = Untriaged (Table 1) + Current Sprint (Table 2) + Future Sprint (Table 3)**

   If the numbers don't match, investigate the gap:
   - Identify the missing issues (present in baseline but absent from all 3 tables)
   - Check their fields (sprint, assignee, priority, etc.) to understand why they fell through
   - Report the discrepancy and the root cause in the executive summary

7. **Executive summary**

   Above the tables, show a brief summary:
   - Total issues in scope (from baseline query)
   - Consistency check result: whether the 3 tables cover all issues, and if not, how many are missing and why
   - Untriaged count (👤 X customer-reported)
   - Current sprint count (👤 X customer-reported, ⚠️ X stale)
   - Future sprint count (👤 X customer-reported, ⚠️ X stale)
   - Issues with open PRs awaiting human review

8. **Save and present**
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
- Stale threshold: 21 days with no human activity (bot activity does not count)
- All activity links point to the specific Jira comment or changelog entry
