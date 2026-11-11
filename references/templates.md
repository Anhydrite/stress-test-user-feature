# Templates

Fill these in as you go. Keep them short enough to be read.

## Feature map

```markdown
## Feature map — <name>

**What it claims to do:** ...
**Where it lives:** <files, routes, commands, migrations>

| Dimension | Findings |
| --- | --- |
| Entry points | ... |
| Preconditions | ... |
| State transitions | ... |
| Side effects | ... |
| Failure paths | ... |
| Permissions | ... |
| Limits | ... |
| Persistence | ... |
| Integrations | ... |
| Cleanup | ... |

**Suspicious spots:** ...
```

## Proof table

```markdown
| Capability | Naive signal to ignore | Proof action | Evidence |
| --- | --- | --- | --- |
| ... | ... | ... | ... |
```

## Test matrix

```markdown
| # | Category | Precondition | Action | Expected observable | Evidence | Status |
| --- | --- | --- | --- | --- | --- | --- |
| T1 | Happy path | ... | ... | ... | ... | TODO |
| T2 | Boundary | ... | ... | ... | ... | TODO |
| T3 | Error | ... | ... | ... | ... | TODO |
| T4 | State | ... | ... | ... | ... | TODO |
| T5 | Security | ... | ... | ... | ... | TODO |
| T6 | Cleanup | ... | ... | ... | ... | TODO |
```

Status values: `TODO`, `PASS`, `ÉCHEC`, `NON TESTÉ`, `BLOQUÉ`.

## Protocol

```markdown
## Protocol — <feature>

**Target:** <env, URL, host, commit>
**Disposable:** yes/no
**Safety:** <what must not be touched>

### Setup
1. ...

### Execution
1. T1 — ...
2. T2 — ...

### Evidence
- T1: <command, output, screenshot, query>
- T2: ...

### Teardown
1. ...

### Stop conditions
- ...
```

## Report

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
- ...
```

Severity:
- `S1` — feature unusable or data loss/security risk.
- `S2` — major path broken, workaround exists.
- `S3` — minor or cosmetic.

## Bug entry

```markdown
### [S?] <title>

- **Repro:** exact steps or command
- **Expected:** ...
- **Observed:** ...
- **Evidence:** ...
- **Impact:** ...
- **Likely area:** file:line if known
```

## Permanent test proposal

```markdown
| Manual test | Automated test | Level | Where | Effort |
| --- | --- | --- | --- | --- |
| T1 | ... | unit / integration / e2e | ... | S/M/L |
```
