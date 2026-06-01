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
Priority:  🔴 Blocker  🟠 Critical  🟡 Major  🟣 Normal  🔵 Minor  ⚪ Undefined
Flags:     👤 Customer-reported   ⚠️ Stale (no human activity in 21+ days)
Release:   🔥 ON_QA + next z-stream GA ≤ 7 days   ❗ Bug open but fix version GA already passed
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
   project = "OpenShift Virtualization" AND component = "CNV Install, Upgrade and Operators" AND (type = Bug OR type = Vulnerability OR type = Weakness) AND status not in (Closed, Verified) AND sprint in openSprints() AND assignee is not EMPTY AND "QA Contact" is not EMPTY AND priority != Undefined AND fixVersion is not EMPTY AND assignee != 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 AND "QA Contact" != 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 ORDER BY status ASC
   ```

   **Query C — Triaged, future sprint** (all 5 fields populated, in a future sprint, not resolved):

   ```
   project = "OpenShift Virtualization" AND component = "CNV Install, Upgrade and Operators" AND (type = Bug OR type = Vulnerability OR type = Weakness) AND status not in (Closed, Verified) AND sprint in futureSprints() AND sprint not in openSprints() AND assignee is not EMPTY AND "QA Contact" is not EMPTY AND priority != Undefined AND fixVersion is not EMPTY AND assignee != 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 AND "QA Contact" != 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 ORDER BY status ASC
   ```

2. **Fetch release schedule**

   Use `WebFetch` to call the Release Console milestones API:

   ```
   GET https://release-console.apps.cnv2.engineering.redhat.com/api/schedule/milestones
   ```

   Parse the JSON response to build a z-stream release lookup:

   - Filter `operators` array for `operator == "CNV"`
   - For each `stream`, extract `version` (major.minor, e.g., "4.21")
   - For each `milestone` with `type == "ga"` and a numeric `z` field, build the
     full z-stream version: `{version}.{z}` (e.g., version "4.21" + z 6 → "4.21.6")
   - Record each z-stream's GA `date` (YYYY-MM-DD format)

   Build a lookup map: `z-stream version → GA date` (e.g., `"4.21.6" → "2026-05-05"`).

   For each bug's Fix Version field (e.g., "CNV v4.20.z"), extract the major.minor
   (e.g., "4.20") and look up the milestones for that stream.

   **CRITICAL**: The Jira Fix Version often contains a generic ".z" suffix (e.g.,
   "CNV v4.20.z"). You MUST resolve this to the **exact z-stream number** from the
   milestones API. Never show ".z" in the Next Release column — always show the
   concrete version number (e.g., "4.20.15", not "4.20.z").

   **Next Release display rules** (for the "Next Release" column in Tables 2 and 3):

   - **Bug is ON_QA**: the fix is already merged and will ship in the next z-stream.
     Find the **next future z-stream GA** for that major.minor stream: the milestone
     with `type == "ga"`, a numeric `z` field, and `date > today`, picking the one
     with the earliest date. Show the resolved version `{major.minor}.{z}` and countdown:
     `4.20.15 (GA in 3d)`
   - **Bug is NOT ON_QA**: find the next future z-stream GA for that major.minor
     (same logic as above). Show the resolved version and days remaining:
     `4.21.8 (GA in 25d)`
   - **Bug is NOT ON_QA, and NO future z-stream GA exists** for that major.minor
     (all GA dates are in the past): the fix missed all target releases. Show the
     most recent past z-stream:
     `4.21.6 (GA passed)`

   **Summary column release icons** (prepended alongside priority/customer/stale icons):

   - `🔥` if bug is ON_QA and next z-stream GA ≤ 7 days
   - `❗` if bug is NOT ON_QA and fix version GA already passed

3. **Enrich each issue**

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
     - Status: Open / Merged / Closed / Draft — **you MUST verify the actual status**.
       Use `WebFetch` on the PR URL to check its real state. Do NOT guess or default
       to "Open". If you cannot determine the status, write "Unknown".
     - Recent human activity only (reviews, comments, commits — ignore bot CI activity)

3. **Build Table 1 — Untriaged Issues**

   | Issue | Summary | Missing Fields | Suggestions | Triage |
   |-------|---------|----------------|-------------|--------|

   - **Issue**: clickable link `[CNV-XXXXX](https://redhat.atlassian.net/browse/CNV-XXXXX)`
   - **Summary**: prepend emoji icons before the **exact Jira summary field** (do NOT
     rephrase, shorten, or summarize — copy it verbatim from the bug's `summary` field):
     - Priority icon (🔴🟠🟡🔵⚪) — always first
     - 👤 if customer-reported
     - ⚠️ if stale (no human activity in 21+ days)
     - Then the **exact** issue summary from Jira (verbatim, never rewritten)
     - Example: `🟡 👤 ⚠️ OVN network policy not enforced after upgrade`
   - **Missing Fields**: comma-separated list (e.g., "Assignee, Sprint")
   - **Suggestions**: proposed values with brief reasoning for each missing field
   - **Triage**: show the command to deep-dive and apply: `/triage-bug CNV-XXXXX`

4. **Build Table 2 — Triaged, Current Sprint (pending resolution)**

   **Row order: sort by Status ascending** (match the JQL ORDER BY status ASC).

   | Issue | Status | Summary | Fix Version | Next Release | Assignee | QA Contact | Activity (21d) | PRs |
   |-------|--------|---------|-------------|--------------|----------|------------|----------------|-----|

   - **Issue**: clickable link
   - **Status**: current Jira status (e.g., NEW, ASSIGNED, POST, MODIFIED, ON_QA)
   - **Summary**: same icon-enriched format as Table 1 (priority + 👤 + ⚠️ + **exact** Jira
     summary, verbatim), plus release urgency icons when applicable:
     - `🔥` if ON_QA and next z-stream GA ≤ 7 days
     - `❗` if NOT ON_QA and fix version GA already passed
     - Example: `🟡 🔥 👤 [4.20] "lowVirtControllersCount" alert firing on Two Node cluster`
   - **Fix Version**: verbatim value from the Jira Fix Version field (e.g., "CNV v4.20.z")
   - **Next Release**: exact z-stream version and GA countdown from Release Console API
     (see step 2 for display rules). Examples: `4.20.15 (GA in 3d)`, `4.21.6 (GA passed)`
   - **Assignee**: developer assigned to the bug
   - **QA Contact**: QE engineer assigned to validate the fix
   - **Activity (21d)**: brief description of human activity, linked to Jira.
     For field updates, mention the change. "None" if no human activity.
   - **PRs**: linked PR URLs (clickable) with status and recent human activity,
     or "None"

5. **Build Table 3 — Triaged, Future Sprint (pending resolution)**

   Same columns and **same row order (by Status ascending)** as Table 2:

   | Issue | Status | Summary | Fix Version | Next Release | Assignee | QA Contact | Activity (21d) | PRs |
   |-------|--------|---------|-------------|--------------|----------|------------|----------------|-----|

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

   **IMPORTANT**: Every issue key mentioned anywhere in the executive summary
   (including notable observations, discrepancies, and any free-text commentary)
   MUST be a clickable Markdown link: `[CNV-XXXXX](https://redhat.atlassian.net/browse/CNV-XXXXX)`.
   Never write a bare issue key.

   Above the tables, show a brief summary:
   - Total issues in scope (from baseline query)
   - Consistency check result: whether the 3 tables cover all issues, and if not, how many are missing and why
   - Untriaged count (👤 X customer-reported)
   - Current sprint count (👤 X customer-reported, ⚠️ X stale)
   - Future sprint count (👤 X customer-reported, ⚠️ X stale)
   - ❗ X bugs with fix version GA already passed (fix missed its target release)
   - 🔥 X bugs ON_QA with next z-stream GA ≤ 7 days
   - Issues with open PRs awaiting human review

   **Sprint progress — Dev side** (current sprint, per Assignee):

   | Assignee | Total | ON_QA (done) | Pending |
   |----------|-------|--------------|---------|

   Count from Table 2: Total = all bugs assigned to this person. ON_QA (done) =
   bugs already in ON_QA status. Pending = Total minus ON_QA.

   **Sprint progress — QA side** (current sprint, per QA Contact):

   | QA Contact | Total ON_QA | Verified/Closed | To verify |
   |------------|-------------|-----------------|-----------|

   Count from Table 2: Total ON_QA = bugs in ON_QA assigned to this QA Contact.
   Verified/Closed = bugs that were ON_QA and have since moved to Verified or
   Closed during this sprint. To verify = Total ON_QA minus Verified/Closed.

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
- [ ] Fix Version column shows verbatim Jira value; Next Release column shows exact z-stream + GA countdown
- [ ] ON_QA bugs with next z-stream GA ≤ 7 days show 🔥 in Summary
- [ ] Bugs whose fix version GA already passed show ❗ in Summary
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
