# /duplicate-check - Identify Potential Duplicate Bugs

## Purpose

Fetches open CNV bugs and uses semantic analysis to identify pairs that may be duplicates.
Reports findings with confidence percentages and reasoning to help the team reduce bug clutter.

## Prerequisites

- Jira MCP server must be configured and accessible
- For best results, run after `/triage` so context on open bugs is already available

## Process

1. **Fetch open bugs**
   - Query Jira for all open CNV bugs (not Closed, Resolved, or Won't Fix)
   - If a previous `/triage` run already fetched bugs, reuse that data
   - Report total count to the user

2. **Build comparison candidates**
   - Group bugs by component to narrow comparison scope
   - Within each component group, compare each pair of bugs for:
     - Title keyword overlap (Jaccard similarity on key terms)
     - Description semantic overlap (key phrases, error messages, affected components)
     - Same affected versions or environment
     - Same reporter or same test/scenario described

3. **Score and filter**
   - Assign a confidence percentage to each candidate pair (0-100%)
   - Only report pairs above a minimum threshold (default: 60%)
   - Confidence tiers:
     - 90%+: Very likely duplicate — recommend linking/closing one
     - 70-89%: Probable duplicate — recommend human review
     - 60-69%: Possible duplicate — flag for awareness

4. **Build duplicates report**
   - For each candidate pair:
     - Bug A key + summary
     - Bug B key + summary
     - Confidence %
     - Reasoning (what overlaps)
     - Recommendation (link as duplicate / investigate / monitor)
   - Group by confidence tier
   - Save to `artifacts/cnv-bug-triage/duplicates-report.md`

5. **Present findings**
   - Show count by tier (e.g., "3 very likely, 7 probable, 12 possible")
   - Display top 10 pairs inline in the conversation
   - Point user to artifact for full list

## Output

- **Duplicates report**: `artifacts/cnv-bug-triage/duplicates-report.md`
  - All candidate pairs with confidence scores, reasoning, and recommendations

## Usage Examples

Run duplicate check across all open CNV bugs:

```
/duplicate-check
```

## Success Criteria

After running this command:

- [ ] All open CNV bugs fetched and compared
- [ ] Candidate duplicate pairs identified with confidence scores
- [ ] Only pairs above 60% threshold reported
- [ ] Duplicates report saved to `artifacts/cnv-bug-triage/duplicates-report.md`
- [ ] Summary presented inline in conversation

## Next Steps

After reviewing duplicates:

1. For high-confidence pairs: manually link one as duplicate of the other in Jira
2. Run `/triage` again to update the triage report after duplicates are closed
3. Run `/report` for an updated status overview

## Notes

- This command is read-only — it never links, closes, or modifies any Jira issue
- Duplicate detection is probabilistic; always requires human judgment to confirm
- Cross-component duplicates may be missed if bugs describe the same issue differently
- Consider running with a smaller scope (single component) for large projects
