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

Evaluate all changes with respect to the service's configured target working branch (e.g., `shared` for TOAuth, or the branch configured in `dhee-deployment/repos/<env>.conf`):

- **Preservation of Target Branch Business Logic:** Existing business logic on the target working branch must be strictly preserved.
- **Mandatory User Approval:** If any change to business logic, domain rules, data flows, roles, permissions, or access control is proposed, it must be explicitly identified, explained domain-by-domain, and approved by the user before implementation or PR approval.
- **Behavioral Impact Verification:** Whenever business logic or domain modules are touched, explicitly analyze and state whether existing behavior changes or remains identical. Distinguish clearly between behavioral changes (e.g., modified outputs, new constraints, updated state transitions) versus strictly non-behavioral improvements (e.g., performance optimizations, query batching, indexing, or database hygiene that preserve 100% of existing behavior).

Check whether the PR changes:

- Course, batch, enrollment, assessment, payment, or certificate rules.
- Authentication, credential verification, token lifecycle, or session rules.
- Roles, RBAC catalog definitions, permissions, access levels, or role assignments.
- State transitions and lifecycle behavior.
- Eligibility, visibility, access control, or tenant-scoping rules.
- Calculations, ordering, pagination, filtering, or defaults.
- Retry, fallback, timeout, and partial-failure behavior.

If no domain logic changed, state that clearly and identify the actual scope of the PR.
If domain logic or underlying queries were modified, state explicitly whether existing functional behavior changes or is preserved.

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
- Business-logic impact (explicitly confirming preservation of target branch business logic, e.g. `toauth:shared`, or identifying user-approved changes).
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

## 15. Deep Technical Audit Methodology (Tawhid-Style Verification)

To identify critical blockers, silent failure modes, and contract drift before code merges, apply these rigorous audit practices on every PR:

### 1. Cross-Service Contract & Schema Verification
- **Audit Against the Real Remote Source:** Never rely solely on local mocks or assumptions. Inspect the actual upstream/downstream service repository on its target working branch (e.g., `lms-finance-service:master`, `TOAuth:shared`, `dhee-deployment`).
- **Inspect Response Schemas Field-by-Field:** Confirm that fields accessed by the consumer (e.g., `invoice_id`, `gateway_transaction_id`) are genuinely defined in the producer's response schema (e.g. `PaymentListResponse`), not detail-only endpoints. Distinguish dead-code paths from fail-open bypasses.
- **Service Principal & Entitlement Checks:** Trace service-to-service (S2S) authentication end-to-end. Verify whether the remote endpoint's authorization logic (e.g., `enforce_acting_tenant_entitlement`) permits service tokens, or if it requires user tenant memberships that cause S2S calls to fail with `403`.

### 2. Dependency & Library Runtime Behavior
- **Inspect Exact Pinned Library Semantics:** Verify that library calls behave as assumed under the exact version pinned in `requirements.txt` (e.g., checking if `pika` `basic_publish` returns `None` vs `True`, or checking driver version compatibilities).
- **Audit Test Mocks for Concealed Bugs:** Ensure test doubles (e.g., `FakeChannel`, mock HTTP clients) do not return values that mask real driver exceptions or inverted return assertions.

### 3. Concurrency, Transaction Boundaries & Retry Lifecycles
- **Audit Error Handlers for Rollback State Destruction:** Trace exception handlers (`except Exception:`) to ensure `db.rollback()` does not wipe out-of-band state changes (such as `inbox.attempts += 1`) that prevent retry limits from incrementing.
- **Isolate Retry Metadata in Separate Transactions:** Ensure retry counters and message deduplication markers are committed independently from the fallible domain transaction.
- **Audit Infinite Retry & Worker Starvation:** Ensure consumer threads do not enter un-throttled or infinite retry loops that starve prefetched queues.
- **Prevent Silent Failure Swallowing:** Never catch an unhandled failure, log a warning, and mark an event `COMPLETED`. Ensure failures fail closed and are marked `FAILED` or retried.

### 4. Database Schema, Migration Chaining & Data Constraints
- **Single Linear Alembic Heads:** Run `alembic heads` or verify migration metadata to ensure exactly one migration head exists, correctly chained from the current branch head.
- **Validate Constraints Against Existing Data:** Before adding `NOT VALID`, unique, or foreign key constraints, verify that existing rows in staging and production comply (e.g. 0 negative balances, 0 duplicate keys).
- **Collation-Neutral Testing:** Ensure SQL test assertions do not assume ASCII/C-locale ordering where PostgreSQL uses ICU/UTF-8 dictionary collation.

### 5. Review Structure
Format review findings using clear actionable tiers:
- **Blockers:** Release-blocking defects (contract breakages, infinite retries, second migration heads, silent data loss).
- **Major:** High-risk logic drift, state reversions, or missing audit trails.
- **Minor:** Defensive guards, doc updates, dead-code removal.
- **Merge Order & Deployment Dependencies:** Explicit cross-service rollout sequence.

## 16. Standard Review Presentation Structure (Presentation to User)

Whenever a pull request review is presented to the user in conversation, it MUST strictly adhere to this standardized, comprehensive structure:

### 1. PR Overview & Metadata
- **Repository & PR Number:** e.g., `TechOptionsBD/lms-course-management-service#271`
- **Title & Author:** Descriptive title and author GitHub handle.
- **Branch & Target:** Feature branch -> target branch (e.g., `multitenancy`, `master`).
- **Rebase & Working Tree Status:** Rebase state, clean working tree, `git diff --check` result.

### 2. File-by-File Classification & Scope
- Table listing touched files, classified by category (Domain Logic, API Contracts, Infrastructure, Database/Migrations, Tests, Documentation) with concise scope summaries.

### 3. Explicit Business Logic Review & Behavioral Impact Verification (Mandatory - Rule 6)
- **Domain Rules Evaluated:** Exhaustive domain-by-domain breakdown (Course, Batch, Enrollment, Assessment, Payment, Lifecycle, Session, etc.).
- **State Transitions & Error Boundaries:** Explicit analysis of how lifecycle states transition, what triggers them, and how errors are handled.
- **Behavioral Impact Verification:** A clear, unambiguous statement distinguishing:
  - *Functional Business Behavior:* Explicitly state whether existing business behavior changes or remains 100% identical.
  - *Operational / Performance Improvements:* Detail non-behavioral changes (e.g. query batching, indexing, concurrency locking, process models).

### 4. Deep Technical Audit (Mandatory - Rule 15)
- **Cross-Service Contracts:** Verification against actual remote service repositories (e.g. Finance, TOAuth, Gateway).
- **Concurrency & Transaction Boundaries:** Row locking (`FOR UPDATE SKIP LOCKED`), transaction boundaries, exception rollback safety, and infinite retry prevention.
- **Database & Migrations:** Single linear Alembic heads, constraint validation against real staging/prod data, collation neutrality.
- **Library Semantics & Mock Fidelity:** Pinned dependency behavior checks and mock validity.

### 5. API & CLI Compatibility (Rule 8)
- Backward compatibility classification (Backward-compatible, Breaking, Additive).
- Explicit audit of query parameters, request bodies, status codes, and error envelopes.

### 6. Severity-Tiered Findings (Rule 7)
- **Blockers (P0 / P1):** Release-blocking correctness, data loss, security, or contract regressions.
- **Major Findings:** Significant state consistency or performance concerns.
- **P2 Operational & Concurrency Notes:** Capacity limits, memory constraints, worker threading, or deployment sequencing.
- **Minor / P3:** Cleanup, documentation, or low-impact suggestions.

### 7. Empirical Validation Results (Rule 11)
- Table of actual commands executed and exact results:
  - `git diff --check`
  - `python3 -m compileall -q`
  - Unit / integration test suite results (exact passed / failed counts and execution times)
  - Deployment compose verification and data tier generator drift checks.

### 8. Coordinated Release & Merge Order (Rule 10)
- Step-by-step merge and deployment sequence across backend, frontend, gateway, and deployment repositories.

### 9. Clear Recommendation & Next Action
- Explicit verdict: **READY TO APPROVE**, **CHANGES REQUESTED**, or **CONDITIONAL APPROVAL**.
- Clear next step for user approval or merge authorization.

---

## 16. Responding to PR Review Feedback (Review Rounds)

When a pull request receives review feedback with `CHANGES_REQUESTED` or substantive technical/business concerns:

1. **Gated Iteration Process:**
   Treat the review round as an issue-resolution iteration following `ISSUE_RESOLUTION_RULE.md` Gate 11.
2. **Review Round Documentation:**
   Before modifying any code, create the paired planning documents in the target service's `Documentation/issues/`:
   - `issue-<id>-pr-review-round<N>-problem-explanation.md`
   - `issue-<id>-pr-review-round<N>-solution-design.md`
   where `<id>` is the issue number and `<N>` is the 1-based review iteration round (e.g., `round1`, `round2`).
3. **Decision Approval:**
   Explain the reviewer's findings, formulate decision questions with trade-offs, and obtain explicit user approval for each decision before implementing.
4. **Verify, Document & Push:**
   Update tests, update `docs/BUSINESS_RULES.md` if business rules or edge cases were clarified, run validation, and push to the remote branch with `git push --force-with-lease`.



