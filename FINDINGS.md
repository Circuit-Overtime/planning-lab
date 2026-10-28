# Findings: issue types and fields on a personal account

Tested 28 October 2026 on `Circuit-Overtime/planning-lab`, a public repository
owned by a personal account enrolled in the issue types and fields preview.

## 1. Defaults

A personal account starts with three enabled issue types: **Task**, **Bug** and
**Feature**. It starts with no issue fields; `viewerCanSeeIssueFields` is
`true`.

## 2. Setting a type with a personal token works

`PATCH /repos/Circuit-Overtime/planning-lab/issues/1` with `type=Task`, sent with
the owner's OAuth token (`repo` scope), set the type. See
[#1](https://github.com/Circuit-Overtime/planning-lab/issues/1).

## 3. `GITHUB_TOKEN` cannot see or set types, and fails silently

The [classify workflow](.github/workflows/classify.yml) reads the issue form's
Kind and sets the type, with `permissions: issues: write`.

- GraphQL `repository.issueTypes` returns `null` under `GITHUB_TOKEN`, with no
  error, while the owner's token returns Task, Bug and Feature.
- `PATCH /issues/3` with `type=Bug` returns success, but the response and the
  issue both show no type. The request is accepted and ignored.

Evidence: [#3](https://github.com/Circuit-Overtime/planning-lab/issues/3) and
[run 40700871326](https://github.com/Circuit-Overtime/planning-lab/actions/runs/40700871326).

Impact: Actions-based triage on personal repositories cannot classify issues,
and nothing in the response tells the workflow author why.

## 4. Creating types and fields needs the `user` scope

`createIssueType` and `createIssueField` with the personal account as
`ownerId` return `FORBIDDEN` for a token with `repo`, `workflow` and
`admin:org` scopes. After adding the `user` scope, both succeed: the account
now has a **CFP** type and **Deadline** (date), **Effort** (number) and
**Stage** (single select: Drafting, Submitted, Accepted, Rejected, Delivered)
fields. The error does not name the missing scope.

## 5. Reserved field names

`Status`, `Type`, `Assignee`, `Labels`, `Milestone` and `Title` are rejected
with "Name cannot have a reserved value". `Priority` is accepted. `Status` is
the most natural name for a planning field, so `Stage` is used here; the
reserved list is not shown in the API error.

## 6. Field values through the API

On [#5](https://github.com/Circuit-Overtime/planning-lab/issues/5), the owner's
token set type `CFP` (REST) and Deadline `2026-11-21`, Effort `6` and Stage
`Drafting` (GraphQL `createIssueFieldValue`). All four read back correctly, and
the timeline records one `IssueTypeAddedEvent` and three
`IssueFieldAddedEvent`s.

## 7. Suggested values wait for review

`createIssueFieldValue` with `suggest: true`, a `rationale` and
`confidence: MEDIUM` proposing Stage = Submitted returned success but left Stage
at Drafting. The proposal is stored in `issue.pendingSuggestions` as a
`PendingFieldSuggestion` with its rationale, author and suggested value, and
adds no timeline event. The schema also defines pending suggestions for types,
labels, assignees and closing, with `applyPendingIssueSuggestions` and
`rejectPendingIssueSuggestions`.

## Open questions

- Will `GITHUB_TOKEN` be able to read and set issue types on personal
  repositories? If not, could the API return an error instead of success?
- Could the `FORBIDDEN` error name the missing `user` scope, and the reserved-name
  error list the reserved names?
- Where are pending suggestions shown to the issue author in the UI?
- Will automation be able to propose types and fields as pending suggestions
  with `GITHUB_TOKEN`, so maintainers review bot classifications instead of
  receiving them silently?
