# Issue tracker: GitHub

Specs and tickets for this repository are stored in GitHub Issues.
Use the gh CLI with the rafsidqi/BRUTAL-raf repository.

## Operations

- Read: gh issue view <number> --repo rafsidqi/BRUTAL-raf --comments
- List: gh issue list --repo rafsidqi/BRUTAL-raf --state open
- Create: gh issue create --repo rafsidqi/BRUTAL-raf --title "..." --body-file <file>
- Comment: gh issue comment <number> --repo rafsidqi/BRUTAL-raf --body-file <file>
- Label: gh issue edit <number> --repo rafsidqi/BRUTAL-raf --add-label "..."
- Resolve: gh issue close <number> --repo rafsidqi/BRUTAL-raf --comment "..."

For multiline bodies, save the text to a UTF-8 file and use --body-file.

## Specs, tickets, and dependencies

- Publish each spec as an issue.
- Create one issue per ticket and link it to the spec issue.
- Record blocking relationships using native issue dependencies.
  If unavailable, use a "Blocked by: #<number>" line.
- A ticket is ready for work once all blockers are resolved.
- Close tickets after their acceptance criteria are met.
  Include validation evidence and commit or PR references.
- Close the spec once all its tickets are resolved.

## Pull requests as a triage surface

PRs as a request surface: no.

## Availability

Ticket operations require Issues to be enabled and appropriate GitHub permissions.
If an operation is denied, report the blocker; do not claim that a ticket
was successfully created or closed.
