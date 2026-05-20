# /triage-bug - Deep Analysis of a Single Bug

## Purpose

Performs a comprehensive triage analysis of a single Jira issue: checks all 5 triage fields,
suggests values for missing ones, analyzes backport needs, and flags potential duplicates.

## Prerequisites

- A valid Jira issue key must be provided (e.g., `CNV-12345`)
- Jira MCP server must be configured and accessible

## Process

1. **Fetch issue details**
   - Retrieve all fields for the specified issue key
   - Confirm the issue exists and is accessible; report an error if not
   - Show the user: key, summary, status, component, description (truncated)

2. **Triage field check**
   - Evaluate each of the 5 triage fields:
     - **Assignee**: populated and is a known developer?
     - **QA Contact**: populated and is a known QE engineer?
     - **Sprint**: assigned to an active or future sprint?
     - **Priority**: not Undefined?
     - **Fix Version**: populated?
   - Report a completeness score (e.g., "3/5 fields populated")

3. **Generate field suggestions**
   - For each missing field, reason through a suggestion:
     - **Priority**: analyze severity keywords (crash, data loss, regression, performance, cosmetic)
     - **Assignee**: match component/area to developer expertise
     - **QA Contact**: match component to QE team coverage
     - **Sprint**: suggest current active sprint or next upcoming sprint
     - **Fix Version**: consider priority and current release timeline
   - Provide reasoning and confidence (high/medium/low) per suggestion

4. **Backport analysis**
   - Review fix versions and release status (dev/ga/maintenance/eol)
   - Evaluate if the issue affects older supported releases
   - Recommend: backport needed / investigate / not needed
   - Provide reasoning based on severity and release age

5. **Duplicate detection**
   - Fetch other open CNV bugs with similar title keywords or same component
   - Compare descriptions semantically for overlap
   - List any potential duplicates with:
     - Issue key and summary
     - Confidence percentage
     - Reasoning for the match

6. **Save artifact**
   - Write full analysis to `artifacts/cnv-bug-triage/bug-{ISSUE-KEY}.md`

7. **Present findings**
   - Show inline: triage score, suggestions table, backport recommendation, duplicates list
   - Ask if user wants to post this as a Jira comment (or run `/post-comments`)

## Output

- **Bug analysis**: `artifacts/cnv-bug-triage/bug-{ISSUE-KEY}.md`
  - Full field analysis, suggestions with reasoning, backport status, duplicate list

## Usage Examples

Analyze a specific bug:

```
/triage-bug CNV-12345
```

## Success Criteria

After running this command:

- [ ] All 5 triage fields checked and reported
- [ ] Suggestions provided for each missing field with confidence level
- [ ] Backport recommendation given with reasoning
- [ ] Potential duplicates identified (or "none found")
- [ ] Analysis artifact saved to `artifacts/cnv-bug-triage/bug-{ISSUE-KEY}.md`

## Next Steps

After analyzing the bug:

1. Run `/post-comments` to post the analysis as a Jira comment
2. If duplicates found, investigate and consider linking/closing them
3. Run `/backport-analysis` for deeper backport coverage across all releases

## Notes

- Never modify Jira fields — this command is read-only
- If the issue key is invalid, ask the user to verify and retry
- Duplicate detection is semantic approximation — flag with confidence, not certainty
