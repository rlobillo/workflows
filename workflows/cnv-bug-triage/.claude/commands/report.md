# /report - Generate Comprehensive Triage Status Report

## Purpose

Produces a full triage status dashboard combining data from all other commands: untriaged bugs,
field-gap breakdown, backport candidates, and duplicate candidates.

## Prerequisites

- Jira MCP server must be configured and accessible
- Previous command runs are optional but enhance the report if their artifacts exist

## Process

1. **Fetch current bug state from Jira**
   - Total open CNV bugs
   - Untriaged bug count (missing any of the 5 fields)
   - Bugs per triage field gap (how many are missing assignee? priority? etc.)
   - Bugs by priority distribution
   - Bugs by component

2. **Load existing artifacts (if available)**
   - Read `artifacts/cnv-bug-triage/triage-report.md` for pending suggestions
   - Read `artifacts/cnv-bug-triage/duplicates-report.md` for duplicate candidates
   - Read `artifacts/cnv-bug-triage/backport-report.md` for backport needs
   - Note which analyses have been run and when

3. **Build the full report**
   - **Executive Summary**: high-level health of the triage queue
   - **Triage Completeness**: table of counts by missing field
   - **Priority Distribution**: breakdown of open bugs by priority
   - **Top Untriaged Bugs**: the 10 most critical untriaged bugs needing immediate attention
   - **Backport Status**: count of bugs needing backport (if backport analysis was run)
   - **Duplicate Status**: count of duplicate candidates (if duplicate check was run)
   - **Comment Coverage**: how many bugs have received a [Bug Triage Agent] comment
   - **Recommendations**: top 3 actions the team should take based on the data

4. **Save the report**
   - Write to `artifacts/cnv-bug-triage/full-report.md`
   - Include timestamp and data sources used

5. **Present summary inline**
   - Show the executive summary and key metrics in the conversation
   - Point user to the artifact for full details

## Output

- **Full report**: `artifacts/cnv-bug-triage/full-report.md`
  - Complete triage dashboard with all sections

## Usage Examples

Generate a full status report:

```
/report
```

## Success Criteria

After running this command:

- [ ] Live Jira data fetched for current bug state
- [ ] All available triage artifacts incorporated into the report
- [ ] Triage completeness breakdown provided per field
- [ ] Top 10 critical untriaged bugs highlighted
- [ ] Recommendations section included
- [ ] Report saved to `artifacts/cnv-bug-triage/full-report.md`

## Next Steps

After reviewing the report:

1. Share `artifacts/cnv-bug-triage/full-report.md` with the team or in a standup
2. Run `/triage` to analyze and suggest values for the top untriaged bugs
3. Run `/post-comments` to push suggestions to Jira
4. Re-run `/report` after a triage session to measure progress

## Notes

- This command works best after running `/triage`, `/duplicate-check`, and `/backport-analysis`
- If no prior artifacts exist, the report will be based on live Jira data only
- The report is point-in-time; always check the timestamp before sharing
