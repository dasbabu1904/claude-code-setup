# Issue Resolution Process

This is the required gated process for resolving issues. The purpose is to
make the business behavior, API contract, security rules, and implementation
scope explicit before code changes begin.

## Gate 1 — Inspect the current system

Perform read-only investigation first:

- understand the reported behavior and business context;
- identify the owning service and architectural boundaries;
- inspect routes, schemas, models, configuration, consumers, and tests;
- use Graphify/JCodeMunch before broad source searches where applicable;
- distinguish confirmed facts, assumptions, and unknowns;
- inspect security, tenant isolation, data integrity, performance, and failure
  behavior.

No business logic, API behavior, schema, workflow, or production configuration
may be changed during this gate.

For an issue-resolution request that asks to follow this process, prepare both
planning artifacts as one proposal: the Gate 2 problem explanation and the
corresponding Gate 4 solution document. Preserve matching issue and decision
numbering between them. Before writing either file, present both exact paths
and their scope together and obtain the user's approval for those document
changes. Do not ask for a second approval between the two files when the
approved scope is unchanged. This approval covers planning documents only;
business behavior and implementation remain gated by the later decision and
approval steps below.

## Gate 2 — Write the problem explanation

Create a problem explanation before proposing code. Write it issue-by-issue;
do not provide only a consolidated read-only report. Every material finding
must use this structure:

```text
## N. Short problem title

Problem explanation:
  What is happening now, with evidence and affected code or configuration.

Business logic impact:
  What rule, workflow, role, tenant, state transition, invariant, or data
  lifecycle is affected.

API behavior impact:
  What request/response, status code, error code, validation, authentication,
  authorization, side effect, or downstream behavior changes or can fail.

Decision N — Explicit question

Possible choices:
  1. Option with consequences.
  2. Option with consequences.

Recommended: the proposed choice and the reason.
```

The problem explanation must also identify:

- affected users, roles, tenants, services, and data;
- current business rules and where they are enforced;
- security, compatibility, migration, performance, and failure risks;
- explicit exclusions and non-goals.

If the current behavior is ambiguous, document the alternatives instead of
silently selecting an interpretation.

## Gate 3 — Analyze behavior changes before designing a solution

Before proposing implementation, explicitly identify whether the issue changes:

### Business logic

- eligibility or permission rules;
- role, tenant, ownership, or state transitions;
- calculations, limits, defaults, or invariants;
- data lifecycle, retention, or deletion behavior;
- retry, idempotency, ordering, or concurrency behavior.

### API behavior

- request fields, response fields, or schemas;
- status codes and error codes;
- authentication and authorization requirements;
- validation rules and backward compatibility;
- pagination, filtering, sorting, or response timing;
- side effects, events, webhooks, or downstream calls.

For every proposed behavior change, state the old behavior, new behavior,
affected callers, migration requirement, and acceptance criteria.

## Gate 4 — Present possible solutions

Create a solution document with alternatives before selecting an approach. It
must correspond to the numbered findings and decisions in the problem
explanation. For each option, explain:

- architecture and ownership;
- business-logic effect;
- API and data-contract effect;
- security and tenant implications;
- compatibility and migration behavior;
- operational cost, performance, and failure modes;
- testing and rollback strategy;
- why the option is recommended or rejected.

The solution document must not silently resolve a decision. Each material
business, API, security, data, compatibility, rollout, or operational choice
must be marked `Approved`, `Rejected`, or `Pending` after the user decides.

Do not present only one implementation when a meaningful product or
architecture choice exists.

## Gate 5 — Confirm decisions one by one

Ask the user to confirm each material decision separately. Confirm decisions
covering, as applicable:

- business rules and workflow behavior;
- API request/response changes;
- authorization, roles, tenants, and security boundaries;
- data model and migration behavior;
- compatibility and legacy behavior;
- rollout, monitoring, and rollback;
- documentation and operational ownership.

Record each decision as `Approved`, `Rejected`, or `Pending`, including the
reason and its consequence. Do not treat general approval of an issue as
approval of unspecified business behavior.

Continue investigating and asking focused questions until the logic is
decision-complete and no material gap remains.

## Gate 6 — Strict logic validation

Before implementation, review the final proposal as a contract and validate:

- every input has defined validation and failure behavior;
- every output and status code is defined;
- authorization is checked at the correct boundary;
- tenant scope and ownership cannot be bypassed;
- state transitions preserve invariants;
- old clients and data have a defined outcome;
- retries and duplicate requests are safe or explicitly rejected;
- downstream failures have bounded and documented behavior;
- tests cover happy paths, invalid inputs, unauthorized access, edge cases,
  races, partial failures, and migrations;
- monitoring can detect the main failure modes.

The implementation plan must identify any remaining unknown. If a material
unknown remains, stop and ask the user; do not guess.

## Gate 7 — Approval and implementation boundary

Wait for explicit user approval of the decision-complete proposal.

Most important rule:

> Do not modify business logic before the user's decisions are made and the
> proposed business behavior is explicitly approved.

Before approval, only read-only investigation and planning documents requested
by the user may change. Do not modify business rules, authorization behavior,
workflows, API semantics, database behavior, migrations, tests that encode
unapproved behavior, or production configuration based on assumptions.

Approval applies only to the exact behavior and file scope described in the
proposal. If either changes, stop and obtain approval again.

## Gate 8 — Implement incrementally

After approval:

- implement only approved behavior;
- preserve unrelated user changes;
- follow existing architecture and naming conventions;
- update tests for the approved behavior;
- avoid unrelated refactors;
- keep each business-logic change traceable to a recorded decision.

## Gate 9 — Verify and review

Run proportionate validation:

- syntax, formatting, type, and static checks;
- unit, integration, contract, and end-to-end tests;
- security and tenant-isolation tests;
- migration and rollback checks;
- deployment or service-level checks where applicable.

Review the final diff for correctness, regressions, race conditions, security,
performance, scope creep, and accidental API changes. If a check cannot run,
record the exact reason, risk, and follow-up.

## Gate 10 — Document and hand off

Update the issue explanation, solution/decision record, API documentation,
README, architecture, deployment, and migration notes as needed. Refresh the
central Graphify index after code or documentation changes.

The final handoff must state:

- what changed;
- which decisions were implemented;
- what was deliberately not changed;
- API/business behavior changes;
- validation performed and results;
- remaining risks or decisions;
- whether commits were intentionally left uncreated.

## Gate 11 — Handling PR Review Feedback (Review Rounds)

When a pull request receives review feedback requesting changes (`CHANGES_REQUESTED` or material technical/business comments):

1. **Treat PR Review Feedback as a Gated Issue Iteration:**
   Do not jump directly to code changes. Address reviewer feedback using the same gated discipline as an issue.
2. **Paired Planning Documents for Review Rounds:**
   Create a dedicated paired set of planning documents in `Documentation/issues/` following the naming pattern:
   - `issue-<id>-pr-review-round<N>-problem-explanation.md`
   - `issue-<id>-pr-review-round<N>-solution-design.md`
   where `<id>` is the issue number and `<N>` is the 1-based review iteration round (e.g., `round1`, `round2`).
3. **Structured Review Problem Explanation:**
   Document each finding from the reviewer using the Gate 2 structure:
   - Short finding title
   - Problem explanation (what was identified in the review with code/contract citations)
   - Business logic impact
   - API behavior impact
   - Explicit numbered decision question
   - Possible choices with consequences
   - Recommended choice
4. **Structured Review Solution Design & Decision Record:**
   Document the corresponding solutions preserving matching numbering (Gate 4):
   - Technical design, affected files, and planned diffs
   - Repository business rules updates (e.g., `docs/BUSINESS_RULES.md`)
   - Decision records marked `Approved`, `Rejected`, or `Pending`
   - Gate 6 strict logic validation
5. **Decision & Approval Gate:**
   Present the proposed decisions and paired document scope to the user and obtain explicit approval (Gate 5 & Gate 7) before editing business logic, implementation code, tests, or configuration.
6. **Implement, Verify & Push:**
   Implement approved changes incrementally (Gate 8), run validation and regression tests (Gate 9), document changes and update central Graphify index (Gate 10), and push updates with `git push --force-with-lease`.

