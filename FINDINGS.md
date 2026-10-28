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

## 4. Creating types and fields through the API

`createIssueType` and `createIssueField` with the personal account as
`ownerId` return `FORBIDDEN` for a token with `repo`, `workflow` and
`admin:org` scopes. Whether the `user` scope is enough is still to be tested.

## Open questions

- Will `GITHUB_TOKEN` be able to read and set issue types on personal
  repositories? If not, could the API return an error instead of success?
- Which token scope creates personal issue types and fields?
- The field-value input already accepts `suggest`, `rationale` and
  `confidence`. Will suggested values be visible to the issue author before
  they are applied?
