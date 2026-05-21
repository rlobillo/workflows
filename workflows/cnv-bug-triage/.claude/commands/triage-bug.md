# /triage-bug - Deep Analysis and Triage of a Single Bug

## Purpose

Performs a comprehensive triage analysis of a single Jira issue: investigates similar bugs,
identifies team expertise, checks all 5 triage fields, suggests values with deep reasoning,
and offers to apply confirmed changes.

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
     - **Assignee**: populated and not the placeholder bot user?
     - **QA Contact**: populated and not the placeholder bot user?
     - **Sprint**: assigned to an active or future sprint?
     - **Priority**: not Undefined?
     - **Fix Version**: populated?
   - Report a completeness score (e.g., "3/5 fields populated")

3. **Research similar bugs**
   - Search for recently resolved bugs in the same component with similar keywords:
     ```
     project = "OpenShift Virtualization" AND component = "CNV Install, Upgrade and Operators"
     AND (type = Bug OR type = Vulnerability OR type = Weakness)
     AND status in (Closed, Verified) AND resolutiondate >= -90d
     AND text ~ "<keywords from title>"
     ```
   - For each similar bug found, note:
     - Who was assigned (developer expertise signal)
     - Who was QA contact (QE coverage signal)
     - What priority was set (severity benchmark)
     - What fix version was used (release targeting pattern)
     - What sprint it was in (sprint assignment pattern)
   - This builds a picture of who works on what and how similar bugs were triaged

4. **Research team expertise**
   - From the similar bugs found in step 3, build a frequency map:
     - Which developers are most active in this area
     - Which QE engineers cover this area
   - Also check the current bug's sub-component or labels for more specific matching
   - Use `lookupJiraAccountId` to resolve names if needed

5. **Generate field suggestions**
   - For each missing field, reason through a suggestion using the research:
     - **Priority**: analyze severity keywords (crash, data loss, regression, performance,
       cosmetic) AND compare with how similar bugs were prioritized
     - **Assignee**: suggest the developer who most frequently works on similar bugs
       in this area. Show the top 2-3 candidates with their recent bug count.
     - **QA Contact**: suggest the QE engineer who covers this area based on similar bugs.
       Show the top 2-3 candidates with their recent bug count.
     - **Sprint**: find the active/future sprints from the board whose filter is
       `project = "CNV" AND component = "CNV Install, Upgrade and Operators"`.
       Suggest the current open sprint, or the next future sprint if none is open.
     - **Fix Version**: based on similar bugs' versions and the current release cycle
   - Provide reasoning and confidence (high/medium/low) per suggestion
   - Never hallucinate team member names — only suggest people found in the research

6. **Duplicate detection**
   - Fetch other open CNV bugs with similar title keywords or same component
   - Compare descriptions semantically for overlap
   - List any potential duplicates with:
     - Issue key (as clickable link) and summary
     - Confidence percentage
     - Reasoning for the match

7. **Backport analysis**
   - Review fix versions and release status (dev/ga/maintenance/eol)
   - Evaluate if the issue affects older supported releases
   - Recommend: backport needed / investigate / not needed
   - Provide reasoning based on severity and release age

8. **Present findings and offer to apply**
   - Show inline: triage score, research summary, suggestions table, duplicates, backport
   - For each missing field, present the suggestion clearly:
     ```
     Field: Priority
     Suggestion: Major
     Reasoning: Similar bug CNV-85000 (crash on upgrade) was set to Major.
                Title contains "failure" keyword. No data loss mentioned.
     Confidence: High
     Apply? [yes / modify / skip]
     ```
   - Ask the user which suggestions they want to apply
   - Allow the user to modify a suggested value before applying
   - Process confirmations one field at a time using `editJiraIssue`
   - Report success/failure for each update
   - **Never apply any change without explicit user confirmation**

9. **Save artifact**
   - Write full analysis to `artifacts/cnv-bug-triage/bug-{ISSUE-KEY}.md`
   - Include: research findings, suggestions, what was applied, duplicates, backport

## Output

- **Bug analysis**: `artifacts/cnv-bug-triage/bug-{ISSUE-KEY}.md`
  - Research summary, field analysis, suggestions with reasoning, applied changes,
    backport status, duplicate list

## Usage Examples

Analyze and triage a specific bug:

```
/triage-bug CNV-12345
```

## Success Criteria

After running this command:

- [ ] All 5 triage fields checked and reported
- [ ] Similar bugs researched for expertise and pattern signals
- [ ] Suggestions based on real team data, not guesses
- [ ] User asked for confirmation before any field is modified
- [ ] Applied changes reported with success/failure status
- [ ] Backport recommendation given with reasoning
- [ ] Potential duplicates identified (or "none found")
- [ ] Analysis artifact saved to `artifacts/cnv-bug-triage/bug-{ISSUE-KEY}.md`

## Notes

- This command READS by default and only WRITES when the user explicitly confirms
- If the issue key is invalid, ask the user to verify and retry
- Duplicate detection is semantic approximation — flag with confidence, not certainty
- Team expertise is derived from recent bug history, not hardcoded lists
