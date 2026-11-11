---
name: stress-test-user-feature
description: "Prove that a feature actually works end to end, not just that it appears in the UI or returns 200. Use this skill whenever the user asks to test, verify, validate, stress-test, QA, sanity-check, or confirm a feature, or says a feature is done, merged, shipped, or ready and wants to be sure it works. It maps every entry point and behavior, defines proof-of-use criteria, builds a full protocol, executes it, reports evidence, then offers to codify the tests and fix gaps. Trigger it even for short requests like 'teste ça' or 'vérifie que ça marche'."
license: MIT
metadata:
  hermes:
    tags: [testing, qa, verification, e2e]
    category: quality
    argument-hint: <feature à tester>
---

# Stress Test a Feature

Turn "the feature is done" into evidence that it actually works.

## The one rule

**Presence is not proof.**

A feature exists when its output can be consumed and its side effects are verified. Seeing a row in the UI, a `200 OK`, a log line, or a record in the database only proves that something was written somewhere. It does not prove the feature works.

Examples of the gap:

| Naive signal | What it actually proves | Real proof |
| --- | --- | --- |
| The VM appears in the list | A record exists | SSH into it, run a command, reboot it, check it comes back |
| The CSV export returns 200 | A response exists | Download the file, parse it, compare row count and values |
| The signup form shows "success" | The frontend called an endpoint | Receive the verification email, click it, log in, check the session |
| The job is queued | A row was inserted | Wait for completion, check the output, retry, check idempotency |
| The delete button disappears the row | The UI hid it | Query the API and DB, check cascades, check quotas and external resources |

If you cannot name the observable result that only a working feature can produce, you do not yet have a test. Say so instead of inventing confidence.

## Workflow

| Phase | Goal | Output |
| --- | --- | --- |
| 1. Recon | Know what exists and where the truth lives | Feature map |
| 2. Proof criteria | Define what "works" means per capability | Proof table |
| 3. Test matrix | Cover happy, edge, error, state, integration, security, cleanup | Prioritized tests |
| 4. Protocol | Make the run reproducible and safe | Setup, steps, evidence, teardown |
| 5. Execute | Run it for real and capture evidence | Raw observations |
| 6. Report | State what is proven, broken, or untested | Compact verdict |
| 7. Offer | Propose permanent tests and fixes | Actionable next step |

Do not skip phase 2. A long test list built on weak proof criteria produces a confident but useless report.

## Phase 1 — Map the feature

Read the code, not just the feature name. Locate the implementation before designing tests:

- routes, controllers, handlers, components, commands, endpoints
- migrations, schemas, models, indexes
- feature flags, config, environment variables
- queues, workers, cron jobs, webhooks
- external services and credentials
- existing tests and fixtures

Then fill this map:

| Dimension | Questions |
| --- | --- |
| Entry points | How can a user or system trigger this? UI, API, CLI, job, webhook? |
| Preconditions | What must exist first? Account, role, quota, flag, data? |
| State transitions | What states exist? What is legal, what is forbidden? |
| Side effects | What changes elsewhere? DB, files, external API, billing, notifications? |
| Failure paths | What happens on invalid input, timeout, conflict, permission denial? |
| Permissions | Which roles or tenants can do what? |
| Limits | Size, rate, count, timeout, quota? |
| Persistence | What survives a reload, restart, or redeploy? |
| Integrations | What upstream and downstream systems are involved? |
| Cleanup | What must be deleted, released, refunded, or cancelled? |

Look for the places where the feature can *look* successful while being unusable: async work that never completes, records without required relations, resources created but not reachable, state written but not read back.

## Phase 2 — Define proof criteria

For each capability, answer: **what observable result can only happen if this works?**

Good proof criteria are:
- **causal** — the result depends on the feature actually doing its job
- **observable** — a command, response, file, log, or state you can capture
- **reproducible** — someone else can obtain the same evidence
- **last-mile** — it uses the output, not just the creation path

Examples:

- Create VM → reach it over SSH, run a command, reboot, verify persistence, then delete and verify the resource is gone.
- Create user → log in as that user, access a protected page, change the password, log in again.
- Export data → open the file, parse it, compare a known row, check encoding and totals.
- Send notification → receive it on the real channel, click the link, verify the state change.
- Upload file → download it, compare the hash, check permissions and size limits.
- Background job → wait for terminal state, inspect output, trigger a retry, verify no duplicate effect.

Write the proof table before writing the test steps:

| Capability | Naive signal to ignore | Proof action | Evidence |
| --- | --- | --- | --- |

If a capability has no feasible proof in the current environment, mark it `NON TESTÉ` with the reason. Do not replace proof with opinion.

## Phase 3 — Build the test matrix

Cover these categories, but prioritize the tests that would catch a broken feature. A focused P0 set beats a hundred shallow checks.

| Category | What to test |
| --- | --- |
| Happy path | The main user journey, end to end |
| Boundaries | Empty, single item, maximum, oversized, unicode, duplicate |
| Errors | Invalid input, missing auth, forbidden role, conflict, timeout, dependency down |
| State | Reload, restart, redeploy, concurrent access, retry, idempotency, partial failure |
| Integration | Upstream input, downstream effect, webhook, queue, external API |
| Security | Role and tenant isolation, injection, secret leakage, unsafe defaults |
| Performance | Large payload, slow dependency, resource usage, rate limit |
| Cleanup | Delete, cascade, quota release, orphan detection |
| Regression | Existing behavior still works around the feature |

Each test gets an ID, a precondition, an action, an expected observable, the evidence to capture, and a status. Keep the matrix in the report so the reader can see what was and was not covered.

Prefer tests that fail loudly. A test that passes whether or not the feature works is worse than no test.

## Phase 4 — Write the protocol

The protocol must be reproducible by someone who was not in the conversation.

1. **Environment** — commit or version, URL or host, disposable target, accounts and roles, fixtures, flags.
2. **Safety** — confirm the target is not production. For destructive, costly, or auth-related tests, state the target explicitly and get confirmation if there is any doubt.
3. **Setup** — exact commands, seed data, reset steps, dependencies.
4. **Execution order** — tests that depend on each other, stop conditions, what to do on failure.
5. **Evidence capture** — raw command output, HTTP status and body, logs, DB queries, screenshots, file hashes, resource IDs.
6. **Teardown** — delete created resources, revoke tokens, stop jobs, verify quotas and external resources are released.

If setup is impossible, do not fake it. Report `BLOQUÉ` with the missing access, dependency, or environment.

## Phase 5 — Execute and observe

Run the tests. Do not simulate results, do not infer from code, and do not report a test you did not run.

While running:
- capture evidence as you go, not from memory
- record deviations, including environment problems
- distinguish `ÉCHEC` from `NON TESTÉ`
- re-run a suspicious failure once to separate flakiness from a real bug
- do not edit the feature during the run unless the user asked; note the fix instead

If a test is destructive or expensive, confirm the target before running it. If a test would affect real user data, stop and ask.

## Phase 6 — Report

Use this structure, in the user's language. Default to French when the user writes French. Keep it compact: verdict first, evidence second, details only where they change the decision.

```markdown
## Verdict : PROUVÉ | PARTIEL | CASSE | BLOQUÉ

**Feature** : ...
**Périmètre** : ...
**Environnement** : commit ..., cible ..., date ...

| # | Test | Attendu | Observé | Preuve | Statut |
| --- | --- | --- | --- | --- | --- |
| T1 | ... | ... | ... | ... | PASS |

### Écarts
1. **[S1] Titre** — repro : ... — attendu : ... — observé : ...

### Non testé
- ... (raison)

### Prochaine étape
...
```

Verdict meanings:

- `PROUVÉ` — every P0 capability passed, each with evidence.
- `PARTIEL` — the core works, but some paths failed or could not be verified.
- `CASSE` — at least one P0 capability failed.
- `BLOQUÉ` — the tests could not be executed.

Rules:
- Never write "ça devrait marcher". Either you observed it or you did not.
- A screenshot of a list is not proof that the item works.
- List what you did not test. Silence looks like coverage.
- Keep the report under about 40 lines unless the user asks for the full detail.

## Phase 7 — Offer permanent tests and fixes

After the report, propose:

1. **Permanent tests** — detect the existing framework and conventions, map each manual test to an automated test, and say where it should run (unit, integration, E2E, CI).
2. **Fixes** — order gaps by severity, point to `file:line`, and describe the smallest change that closes each gap.
3. **Effort** — rough size and risk for each item.

Then wait for explicit approval before editing code. The user asked for a diagnosis first; do not turn it into an unrequested refactor.

## Guardrails

- Never run destructive tests against production or real user data.
- Never print secrets, tokens, or credentials in the report.
- Always clean up created resources and verify the cleanup.
- Watch cost and quotas for cloud, email, and third-party APIs.
- If the feature touches money, auth, or deletion, confirm the target environment before executing.
- If access is missing, report `BLOQUÉ`; do not lower the proof bar to make the report look green.

## References

- `references/playbooks.md` — domain-specific proof checklists for UI, API, CLI, infra/VM, and async jobs.
- `references/templates.md` — fill-in templates for the feature map, test matrix, protocol, report, and bug entry.

## Final check before reporting

- [ ] Every capability has a proof-of-use test, not a presence check.
- [ ] Negative, error, and permission paths are covered.
- [ ] State, restart, concurrency, and idempotency were considered.
- [ ] Cleanup was executed and verified.
- [ ] Evidence is captured and reproducible.
- [ ] Untested items are listed with reasons.
- [ ] The proposal waits for explicit approval.
