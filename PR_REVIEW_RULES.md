# PR Review Rules

These are the rules and workflow used when reviewing and preparing Dhee LMS pull requests.

## 1. Start with a clean branch

- Check the current branch and working-tree status.
- Confirm the feature branch tracks the expected remote branch.
- Pull the latest feature branch changes.
- Do not continue if unrelated uncommitted changes are present.

```bash
git status
git pull origin
```

## 2. Rebase onto the target branch

Rebase the feature branch onto the branch that the PR will merge into, usually `master` or the configured environment branch.

```bash
git rebase <target-branch>
```

After rebasing, divergence from the remote feature branch is expected. Do not use `git pull` to merge the old remote history back into the rebased branch.

## 3. Resolve conflicts deliberately

For every conflict:

1. Inspect both sides of the conflict.
2. Understand the business and security intent of each change.
3. Preserve compatible behavior from both branches where possible.
4. Check for syntax errors and remaining conflict markers.
5. Continue the rebase only after the resolution is reviewed.

```bash
git status
git diff -- <conflicted-file>
git add <conflicted-file>
git rebase --continue
```

Never resolve conflicts by blindly choosing `ours` or `theirs`.

If more than one technically valid conflict resolution could change behavior, stop and ask the user to choose. Do not assume the intended business, security, or compatibility outcome.

## 4. Compare the feature branch with the base branch

Review the feature-only diff, not the entire rebased history:

```bash
git diff --stat <target-branch>...HEAD
git diff --name-status <target-branch>...HEAD
git diff --check <target-branch>...HEAD
git log --oneline <target-branch>..HEAD
```

Use `git range-diff` to confirm that rebasing did not lose or materially alter feature commits:

```bash
git range-diff <target-branch>..<remote-feature> <target-branch>..HEAD
```

## 5. Classify the changes

For each changed file, determine whether the change affects:

- Domain business logic.
- API routing or response contracts.
- Authentication, authorization, tenant isolation, or privacy.
- Reliability, rate limiting, caching, or infrastructure.
- Database schema or migrations.
- Tests and documentation only.

Do not describe a security or infrastructure change as having no behavioral impact. Security controls can intentionally change request outcomes, response codes, or failure behavior.

## 6. Review business logic explicitly

Check whether the PR changes:

- Course, batch, enrollment, assessment, payment, or certificate rules.
- State transitions and lifecycle behavior.
- Eligibility, visibility, permission, or tenant-scoping rules.
- Calculations, ordering, pagination, filtering, or defaults.
- Retry, fallback, timeout, and partial-failure behavior.

If no domain logic changed, state that clearly and identify the actual scope of the PR.

## 7. Finding severity

Classify review findings consistently:

- `P0`: security breach, cross-tenant data exposure, data loss, or a release-blocking outage.
- `P1`: incorrect business behavior, a breaking API contract, or a serious correctness regression.
- `P2`: reliability, performance, deployment, or significant test-gap risk.
- `P3`: maintainability, documentation, or low-impact cleanup.

Prioritize `P0` and `P1` findings before approval. Explain the evidence and affected users or services for every finding.

## 8. Review API compatibility

Check for:

- Removed or renamed response fields.
- New required request fields.
- Changed field types or nullability.
- Changed status codes or error envelopes.
- Changed authentication requirements.
- Changed pagination, filtering, sorting, or default behavior.
- New `429`, `401`, `403`, or `5xx` paths.

Classify compatibility as:

- Backward-compatible: additive or ignored changes.
- Conditionally compatible: clients must tolerate new errors or nullable fields.
- Breaking: removed fields, renamed fields, changed types, or changed required behavior.

## 9. Review cross-service consistency

When multiple services participate in one change:

- Compare producer response schemas with consumer types.
- Search all consumers of removed or renamed fields.
- Confirm fallback behavior between services.
- Confirm shared secrets, URLs, headers, and configuration names.
- Confirm tenant scope and authorization assumptions match across services.
- Merge coordinated backend and frontend changes together when required.

## 10. Coordinated merge and release checklist

When a change spans multiple repositories:

- Merge backend API changes before dependent frontend changes, unless the API remains backward-compatible.
- Configure required environment variables and shared secrets in every affected service.
- Apply and verify database migrations before exercising dependent endpoints.
- Run deployment smoke tests for the affected services.
- Confirm a rollback path for code, configuration, and migrations.
- Recheck the PR after each dependent repository is updated.

## 11. Validate proportionally

Run the most relevant checks available:

```bash
git diff --check
python3 -m compileall -q <source> <tests>
pytest -q <relevant-tests>
npm run type-check
npm run lint
```

If the full suite cannot run, record:

- The exact command attempted.
- Why it could not run.
- Which narrower checks passed.
- The remaining risk.

Do not claim a test passed when only static inspection was performed.

## 12. Approval comment format

An approval comment should summarize:

- Rebase and conflict-resolution status.
- Business-logic impact.
- API response and compatibility impact.
- Security, privacy, reliability, or infrastructure impact.
- Cross-service configuration requirements.
- Tests and checks run.
- Any remaining risks or follow-up actions.

Approval should be conditional when deployment configuration or coordinated PRs are required.

## 13. Push rebased branches safely

After the changes are committed and verified, update the remote branch with:

```bash
git push --force-with-lease origin <feature-branch>
```

Use `--force-with-lease`, never plain `--force`, because it refuses to overwrite unexpected remote updates.

Before pushing, verify:

```bash
git status
git log --oneline --decorate -5
git diff --check
```

## 14. Do not commit without approval

Keep changes uncommitted until the user explicitly authorizes the commit. Before committing, summarize:

- Files to be committed.
- Functional and security impact.
- Validation results.
- Proposed Conventional Commit message.

Suggested commit prefixes:

- `feat:` for new behavior.
- `fix:` for corrected behavior.
- `security:` for security hardening.
- `refactor:` for structure-only changes.
- `test:` for test changes.
- `docs:` for documentation changes.
