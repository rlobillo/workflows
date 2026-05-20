# /backport-analysis - Analyze Bugs for Backport Needs

## Purpose

Reviews recently resolved CNV bugs and evaluates whether their fixes need to be backported
to older supported releases, considering the release status and severity of each issue.

## Prerequisites

- Jira MCP server must be configured and accessible
- Understanding of the current CNV release landscape (dev/ga/maintenance/eol releases)

## Process

1. **Fetch recently resolved bugs**
   - Query Jira for CNV bugs resolved in the last 30 days (configurable):
     ```
     project = CNV AND issuetype = Bug AND status in (Resolved, Closed) AND resolutiondate >= -30d AND resolution = Fixed
     ```
   - Report count to the user

2. **Determine release landscape**
   - Ask the user (or infer from Jira fix versions) which releases are currently:
     - **dev**: In active development (backport unlikely needed, will be included)
     - **ga**: Generally Available and under active support (backport likely needed for critical/major)
     - **maintenance**: Maintenance mode (backport only for critical security/data-loss issues)
     - **eol**: End of Life (no backports)

3. **Evaluate each bug for backport**
   - For each resolved bug, check:
     - **Fix Version**: which release was it fixed in?
     - **Priority**: Critical or Major bugs in dev releases likely need backporting to ga/maintenance
     - **Type**: Security issues, data loss, and crash bugs warrant broader backport consideration
     - **Existing backport links**: Are there already linked backport issues?
   - Determine recommendation:
     - **Backport needed**: High priority, affects users on older supported releases
     - **Investigate**: Medium confidence — needs human review
     - **Not needed**: Low priority or cosmetic, or already backported, or eol releases only

4. **Check for existing backport issues**
   - Search for linked issues or issues with "backport" in the title referencing each bug
   - Flag bugs where a backport is needed but no backport issue exists yet

5. **Build backport report**
   - Summary table: bug key | priority | fix version | recommendation | existing backport?
   - Detailed section per bug: summary, reasoning, recommended target releases
   - Save to `artifacts/cnv-bug-triage/backport-report.md`

6. **Present findings**
   - Show counts: needs backport / investigate / not needed
   - Highlight any critical bugs with no backport issue filed

## Output

- **Backport report**: `artifacts/cnv-bug-triage/backport-report.md`
  - Full table and detailed analysis per bug with backport recommendations

## Usage Examples

Run backport analysis for the last 30 days:

```
/backport-analysis
```

## Success Criteria

After running this command:

- [ ] Recently resolved bugs fetched from Jira
- [ ] Each bug evaluated against the current release landscape
- [ ] Backport recommendation given with reasoning for each bug
- [ ] Bugs flagged where backport is needed but no issue exists
- [ ] Report saved to `artifacts/cnv-bug-triage/backport-report.md`

## Next Steps

After reviewing the backport report:

1. For "Backport needed" items without existing backport issues: file backport Jira issues
2. For "Investigate" items: bring to the team for a triage decision
3. Run `/report` for a full triage status summary including backport coverage

## Notes

- This command is read-only — it never creates Jira issues or modifies fields
- Release status (dev/ga/maintenance/eol) must be provided by the user or inferred from Jira data
- Backport decisions are recommendations; the engineering team makes the final call
- Adjust the resolved-date window (default 30d) based on team cadence
