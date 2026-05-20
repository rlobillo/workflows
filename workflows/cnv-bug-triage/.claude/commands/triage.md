# /triage - Analyze All Untriaged CNV Bugs

## Purpose

Fetches open, untriaged CNV bugs from Jira, checks which of the 5 triage fields are missing for each,
and generates suggestions for each gap. Produces a triage report artifact summarizing findings.

## Prerequisites

- Jira MCP server must be configured and accessible
- User should have confirmed the target component (default: "CNV Install, Upgrade and Operators")

## Process

1. **Fetch untriaged bugs**
   - Query Jira with the default JQL for untriaged CNV issues:
     ```
     project = "OpenShift Virtualization" AND component = "CNV Install, Upgrade and Operators" AND (type = Bug OR type = Vulnerability OR type = Weakness) AND status not in (Closed, Verified) AND (assignee is EMPTY OR "QA Contact" is EMPTY OR sprint not in (openSprints(), futureSprints()) OR priority = Undefined OR fixVersion is EMPTY OR assignee = 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9 OR "QA Contact" = 712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9) ORDER BY createdDate ASC
     ```
   - The account ID `712020:0a621ff3-50ea-43eb-ab16-4b09475e57d9` is a placeholder user that counts as unassigned
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
   - Show a brief table: bug key (as clickable link) | missing fields | top suggestion
   - Format every issue key as a Markdown link: `[CNV-XXXXX](https://atlassian.redhat.net/browse/CNV-XXXXX)`
   - Invite user to run `/post-comments` to post suggestions to Jira

## Output

- **Triage report**: `artifacts/cnv-bug-triage/triage-report.md`
  - Full list of analyzed bugs with missing fields and suggestions

## Usage Examples

Run full triage sweep:

```
/triage
```

Triage a different component:

```
/triage component = "Network"
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
