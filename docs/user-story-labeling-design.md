# Organization-wide user-story labeling design

## Scope and meaning

This is a proposed design, not deployed automation. The user-story form allows
omitting goals, success criteria, and their readiness confirmations without
blocking issue creation. Its first readiness checkbox remains required; the UX
checkbox and goals textarea remain optional. Labels are not added statically to
the form: completed submissions must not start with missing-content labels.

Reconcile only issues carrying the existing `user-story` label, regardless of
issue type or title. Provision this scope label in consuming repositories as well
as the form's existing `backlog` label. Legacy or manually created issues without
`user-story` are ignored; maintainers must explicitly opt them in after review.
Manually labeled stories without recognizable form sections receive the relevant
missing labels rather than being assumed complete.

Manage exactly two labels, independently:

| Label | Section heading | Exact readiness checkbox text |
| --- | --- | --- |
| `missing-goals` | `💡 Goals & Motivation Context` | `Goals are clearly defined and provide sufficient context to understand what we want to accomplish.` |
| `missing-success-criteria` | `✅ Success Criteria` | `Success criteria provides clarity to understand what should happen for known scenarios.` |

For each criterion, **apply its label unless both** of these are true:

1. The corresponding section contains substantive text.
2. Its exact corresponding checkbox is checked within
   `🏁 Definition of Ready Criteria`.

An absent section, empty or whitespace-only section, or section containing only
GitHub's `_No response_` placeholder counts as missing content. A missing or
unchecked readiness checkbox counts as not confirmed even when text is present.
These labels mean **“missing or not yet confirmed,” not an automated judgment of
quality**. A checked box cannot substitute for content. Optional unchecked boxes
must never block creation; automation runs after submission. The first readiness
checkbox and optional UX checkbox do not affect either label.

When an issue loses `user-story`, remove only these two automation-owned labels
to reconcile scope. Reapplying `user-story` evaluates the latest body again.
Never replace the issue's full label list or alter any unrelated label.

## Recommended architecture: reusable workflow with opt-in callers

Inherited issue forms do **not** cause workflow inheritance. Publish a central
reusable workflow here, for example
`.github/workflows/reconcile-user-story.yml` with `workflow_call`, then install a
thin caller in `.github/workflows/` on the **default branch of each opted-in
repository**. For `ux-fortress`, that branch is `develop`, not `main`. This proposal
does not install callers or make changes in other repositories.

The caller should:

- Subscribe to `issues` events `opened` and `edited`. Also subscribe to `labeled`
  and `unlabeled` to support late opt-in, scope removal, and correction of manual
  changes to the managed labels. Duplicate events converge to a no-op. Do not
  filter solely using the event's possibly stale label list.
- Grant only `issues: write` to the called job via the repository-scoped
  `GITHUB_TOKEN`; unspecified permissions are disabled. The reusable workflow
  cannot elevate permissions beyond the caller. No org-wide PAT or inherited
  secrets are needed. Confirm repository Actions policies permit the call.
- Call
  `psu-pennstateit-swe/.github/.github/workflows/reconcile-user-story.yml@<reviewed-full-commit-SHA>`
  at job level. This is a reference pattern, **not a runnable caller**: the
  workflow must first be implemented and reviewed, and the placeholder replaced
  with its immutable commit SHA. Do not pin to a moving `main` ref.
- Pass only the issue number as data; use the caller's repository context as the
  API target, not the central repository. Validate the issue number and permitted
  event before writing. Avoid arbitrary repository/body inputs.

The central implementation should use the API to provision labels and reconcile
the issue, not check out or execute consuming repository code. A caller checkout
does **not** automatically fetch scripts from the called repository. Prefer
reviewed logic embedded in the reusable workflow; if separate central scripts
are needed, fetch the central repository explicitly at the same reviewed
immutable revision with credentials not persisted. A private target must not
accidentally check out its own scripts instead. Any checkout adds a justified
`contents: read` permission; an API-only implementation needs only `issues: write`.
Pin any third-party actions to reviewed full commit SHAs as well.

Treat issue text as untrusted data. Fetch it through an API client and parse it
in memory. Never interpolate issue titles, bodies, or labels into shell commands,
GitHub Actions script expressions, executable code, or file paths. Do not execute
code blocks, follow content-supplied URLs, or log entire issue bodies.

### Alternative for automatic all-repository onboarding

An organization-installed GitHub App can receive issue webhooks for all selected
repositories (including future repositories if installed for all repositories)
without distributing caller files. Give it only repository Issues read/write
and necessary metadata access, use short-lived installation tokens, validate
webhook signatures and deliveries, and apply the same parser and reconciliation
rules. This requires a hosted service, secret management, retries, monitoring,
and installation governance. Prefer the reusable workflow for explicit opt-in;
choose the App if automatic organization-wide onboarding is a firm requirement.

## Parsing contract

Issue-form IDs are not retained as structured metadata on an issue. Parse the
rendered Markdown headings from the form labels, not `story-goals-context`,
`success-criteria`, or `definition-of-ready` as if they were API fields.

The implementation and fixtures should follow these rules:

1. Normalize CRLF to LF and trim surrounding whitespace when comparing headings,
   checkbox text, and placeholder content. Preserve the exact wording and emoji
   of known form headings and readiness labels.
2. Use a Markdown-aware block scanner/parser that tracks fenced code blocks,
   including backtick and tilde fences and their matching closing lengths.
   Ignore headings and checkbox-looking strings inside fences. An unclosed fence
   must not allow apparent later headings or checkboxes to satisfy a criterion.
3. GitHub renders the form fields as level-three headings. Locate known form
   boundaries using the complete current heading set: `🔗 Parent Feature Link`,
   `📋 The User Story`, `💡 Goals & Motivation Context`, `✅ Success Criteria`,
   `👤 Target Persona`, `Target Persona Additional Context (Optional)`,
   `🏁 Definition of Ready Criteria`, and
   `🛠️ Technical Notes / Implementation Hints (Optional)`.
   A section ends at the next recognized level-three form heading outside code.
   Allow nested headings in goals content (including the form's suggested
   `## Goals` and `## Motivation & Goal Context`); do not split on every heading.
   Unknown headings are content, not proof that a known section exists.
4. Within each content section, remove structural headings, blank lines, and
   standalone `_No response_` placeholder lines before testing for substantive
   content. Heading-only, formatting-only, or fence-delimiter-only content is not
   substantive; actual nonblank prose, list items, or code content can be.
   This is presence detection only, not validation of Given/When/Then or quality.
5. Search for Markdown task-list items **only within the recognized Definition
   of Ready section**, outside fenced code. Accept checked markers `[x]` and
   `[X]`, and unchecked `[ ]`; match the full corresponding label text, not
   a substring. A checked copy in goals, notes, or a code sample does not count.
6. Fail closed for each affected criterion: missing/duplicate relevant sections,
   duplicate matching checkboxes, conflicting states, malformed markers, or
   ambiguous boundaries cannot clear its label. Missing/ambiguous readiness
   sections affect both criteria. On parser failure, retain/apply missing labels
   for affected criteria and report a bounded diagnostic, not a false success.
   Version heading/checkbox changes together with fixtures and rollout plans.

## Provisioning and race-safe reconciliation

Provision `missing-goals` and `missing-success-criteria` in **each consuming
repository**, with descriptions explaining “missing or not yet confirmed.”
The repository-scoped `issues: write` token can create missing labels. Treat a
concurrent create's “already exists” response as success after re-fetching;
do not overwrite an existing label's color or description without agreement.
Surface permission, API, or provisioning failures for retry; do not clear labels
based on incomplete reads.

For each reconciliation:

1. Serialize work per repository and issue with a shared caller concurrency
   group (including repository and issue number), `cancel-in-progress: false`.
   All live-event and backfill paths must use the same serialization mechanism.
   Do not rely on event ordering or concurrency alone to prevent stale writes.
2. Re-fetch the current issue body and labels after acquiring the slot, including
   for queued older events. Re-check scope from that response.
3. Compute the desired membership of only the two owned labels. Immediately
   before mutations, re-fetch and compare body and scope label with the evaluated
   snapshot; restart if either changed.
4. Add missing owned labels with the additive labels endpoint and remove an
   owned label by its individual delete endpoint only when appropriate. Skip
   already-correct membership; an already-absent delete is a no-op. Never use the
   replace-all-labels endpoint. Retry rate limits/transient failures with bounded
   backoff; a partial failure is safe to retry idempotently.
5. Re-fetch after writes and recompute from the latest body, scope, and managed
   labels. Repeat until converged, or fail visibly after bounded retries and
   arrange an explicit retry through the same issue-level path. Label changes
   made with `GITHUB_TOKEN` do not trigger new workflow runs, so do not rely on
   those events for recovery. Human label changes and subsequent edits reconcile
   again. GitHub issue/label APIs offer no atomic body-and-label
   transaction, so a brief stale label window is possible; this is eventual
   convergence, not a claim of atomicity.

GitHub Actions concurrency can replace pending runs and does not guarantee FIFO.
That is acceptable only because every surviving run reads current state, not an
event-body snapshot. Backfill must enqueue the same issue-level reconciliation,
not mutate labels in an independent bulk writer.

## Deployment and rollout

1. Merge the optional-field template change to this repository's `main`. Verify
   inherited forms in consuming repositories without local template overrides;
   confirm blank success criteria and unchecked optional confirmations submit.
   No workflow deployment is necessary for this form change.
2. Implement and test the central reusable workflow against the matrix below.
   Review code and action pins; publish it on `main`, then choose its immutable
   SHA. Provide a reviewed caller artifact and explicit installation instructions;
   a central workflow file alone does not distribute callers.
3. Pilot in one opted-in repository: provision labels, install the caller on its
   default branch, verify Actions access/permissions, then exercise edits and
   scope changes. For a future `ux-fortress` rollout, use `develop`.
4. After approval, distribute callers through separate reviewed changes in each
   opted-in repository. Track installed pins and update them through reviewed PRs.
   Roll back by disabling/removing callers or restoring a prior reviewed SHA.
   Any cleanup must remove only the two owned labels.
5. Perform a one-time, paginated backfill of existing `user-story` issues,
   including closed issues if historical labels are desired (record the chosen
   scope). Exclude pull requests, dry-run first, then enqueue each issue through
   the same latest-state reconciliation and concurrency path. Checkpoint
   pagination, respect rate limits, and make retries safe. Review unlabeled legacy
   stories separately rather than guessing their type from titles or content.

## Acceptance test matrix

Before deploying automation, implement parser fixtures and mocked API tests:

| Case | Expected managed labels |
| --- | --- |
| Neither criterion has text or confirmation | Both |
| Both have substantive text and checked exact confirmations | Neither |
| Only goals have text and checked confirmation | `missing-success-criteria` |
| Only success criteria have text and checked confirmation | `missing-goals` |
| Checked boxes with empty, whitespace-only, or `_No response_` text | Corresponding missing labels |
| Populated text with unchecked or absent confirmations | Corresponding missing labels |
| UX checkbox checked, unchecked, or absent | No effect on either label |
| First readiness checkbox changed after creation | No effect on either label |
| Missing content section or missing readiness section | Affected label, or both for missing readiness |
| Duplicate sections/checkboxes or malformed recognized story | Fail closed for affected criteria |
| Goals with nested `##`/`###` headings and real content | Goals complete if its confirmation is checked |
| Heading-only goals content | `missing-goals` even with a checked confirmation |
| CRLF, surrounding whitespace, `[x]` and `[X]` | Same result as normalized input |
| Fake headings/checkboxes inside backtick or tilde fences | Cannot satisfy or terminate sections |
| Matching checkbox outside readiness; unclosed fence hiding readiness | Cannot satisfy confirmation |
| Real code content in a valid content section | Counts as content, not as readiness confirmation |
| Existing unrelated labels | Preserved on all add/remove paths |
| No `user-story` label | No managed labels; remove stale owned labels |
| `user-story` added to a legacy body without sections | Both |
| Repeated delivery, out-of-order edit, edit during a write, concurrent backfill | Idempotent convergence to latest body/scope |
| Label creation race, partial API failure, rate limit, missing permission | Safe retry or visible failure, no unrelated-label loss |

Also manually test the inherited form on GitHub: all newly optional fields may
be omitted, but the first readiness checkbox and persona remain required.

## Validation for this change

This repository has no existing test runner, build, or linter configuration.
Validate the YAML with an available YAML parser and compare the parsed template
against the pre-change version: the only differences must be `required: false`
for `success-criteria` and the two specified readiness options. Review the textual
diff as well to ensure all other content, labels, issue type, and validations are
preserved. No automation parser or deployment is included here, so the matrix is
an acceptance specification, not a claim of executed automation tests.
