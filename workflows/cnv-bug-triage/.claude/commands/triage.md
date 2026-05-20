# /triage - Analyze All Untriaged CNV Bugs

## Purpose

Fetches open, untriaged CNV bugs from Jira, checks which of the 5 triage fields are missing for each,
and generates suggestions for each gap. Produces a triage report artifact summarizing findings.

## Prerequisites

- Jira MCP server must be configured and accessible
- User should have confirmed the target Jira project (default: CNV)

## Process

1. **Fetch untriaged bugs**
   - Query Jira with the default JQL for untriaged CNV bugs:
     ```
     project = CNV AND issuetype = Bug AND status not in (Closed, Resolved, "Won't Fix") AND (priority = Undefined OR assignee is EMPTY OR fixVersion is EMPTY)
     ```
   - Report total count found to the user before proceeding
   - If count is large (>50), ask user if they want to limit scope

2. **Check triage completeness**
   - For each bug, evaluate which of the 5 fields are missing:
     - Assignee (empty)
     - QA Contact (custom field, empty)
     - Sprint (not assigned to an active/future sprint)
     - Priority (value is "Undefined")
     - Fix Version (empty)
   - Track a completeness score (0-5) per bug

3. **Generate suggestions**
   - For each missing field, reason through a suggestion based on:
     - Bug title and description content
     - Component and labels
     - Reporter and past similar bugs
     - Known team members and their expertise areas
   - Provide a confidence level (high/medium/low) per suggestion
   - Flag any suggestion that cannot be made with high confidence

4. **Build triage report**
   - Summarize: total bugs fetched, fully triaged count, untriaged count
   - List each untriaged bug with: key, summary, missing fields, and suggestions
   - Highlight critical/blocker bugs that are missing priority
   - Save to `artifacts/cnv-bug-triage/triage-report.md`

5. **Present summary in conversation**
   - Show a brief table: bug key | missing fields | top suggestion
   - Invite user to run `/post-comments` to post suggestions to Jira

## Output

- **Triage report**: `artifacts/cnv-bug-triage/triage-report.md`
  - Full list of analyzed bugs with missing fields and suggestions

## Usage Examples

Run full triage sweep:

```
/triage
```

Triage with custom JQL:

```
/triage project = CNV AND component = "Network" AND priority = Undefined
```

## Success Criteria

After running this command:

- [ ] Total bug count reported to user
- [ ] Each untriaged bug analyzed for all 5 triage fields
- [ ] Suggestions provided with reasoning for each missing field
- [ ] Triage report saved to `artifacts/cnv-bug-triage/triage-report.md`

## Next Steps

After completing triage analysis:

1. Review suggestions in the triage report
2. Run `/post-comments` to post structured suggestions to Jira
3. Run `/report` for a full status dashboard
4. Run `/duplicate-check` to find potential duplicates among untriaged bugs

## Notes

- Never modify Jira fields — this command is read-only
- If Jira returns no results, confirm the JQL filters and project key with the user
- Large bug lists may take time; keep the user informed of progress
