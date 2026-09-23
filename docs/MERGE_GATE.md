# Merge gate: what actually has to pass

> **Source of truth.** The *required* column below mirrors the branch protection
> rule for `main`. Only a repository admin can read that rule (Settings →
> Branches → `main`). If you add or rename a check, update both this table and
> the rule in the same pull request — a check that is required but renamed
> silently blocks every PR, and a check that only exists in this table blocks
> nothing.
>
> Workflows that run on a schedule, on `workflow_dispatch`, or only to label
> issues do **not** gate a merge.

## Required checks

| Check (workflow / job) | Trigger | Gates the merge (verify in Settings → Branches) | Run it locally |
| --- | --- | --- | --- |
| `1️⃣2️⃣ Security — Secrets (Gitleaks)` — `ci.yml / secrets-scan` | `pull_request` → `main` | Yes | `gitleaks detect --config .gitleaks.toml --verbose --redact` |
| `2️⃣ Frontend — TypeCheck (tsc)` — `ci.yml / typecheck` | `pull_request` → `main` | Yes | `npm ci && npx prisma generate && npx tsc --noEmit` |
| `3️⃣ Frontend — Unit Tests (Vitest)` — `ci.yml / unit-tests` | `pull_request` → `main` | Yes | `npm ci && npx prisma generate && npx vitest run` |
| `5️⃣ Backend — Contracts (WASM + Tests)` — `ci.yml / contract-wasm` | `pull_request` → `main` | Yes | `rustup target add wasm32v1-none` then `cd contracts/ophirpay && cargo build --target wasm32v1-none --release`, same for `contracts/emitter` |
| `Validate schema, replay migrations, and test invariants` — `prisma-ci.yml / prisma` | `pull_request` touching `prisma/**`, `scripts/validate-prisma-migrations.sh`, `src/__tests__/prisma-schema-ci.test.ts` | Yes, when the workflow runs (path-filtered) | `npm ci && npx prisma validate && npx prisma generate && bash scripts/validate-prisma-migrations.sh && npx vitest run src/__tests__/prisma-schema-ci.test.ts` |
| `contract-regression` — `contract-regression.yml` | `pull_request` touching `contracts/**`, `scripts/check-contract-regressions.mjs`, or that workflow | Yes, when the workflow runs (path-filtered) | `cd contracts && cargo build --manifest-path ophirpay/Cargo.toml --target wasm32v1-none --release && cargo build --manifest-path emitter/Cargo.toml --target wasm32v1-none --release && node scripts/check-contract-regressions.mjs` |

Everything in `npm run ci` is the closest single-command mirror of CI:

```bash
npm run ci   # typecheck → lint (0 warnings) → vitest → next build → validate-deploy-config.sh
```

Two caveats worth knowing before you trust it:

- `npm run ci` does **not** include the Gitleaks scan, the WASM contract builds,
  the Prisma migration replay, or the contract-regression check. Those are the
  checks most likely to surprise you.
- `deploy-config` validation only runs through `npm run ci`; it is not a job of
  its own in `.github/workflows/ci.yml`.

## Advisory: runs, but does not gate a merge

| Workflow | Trigger | Why it is advisory |
| --- | --- | --- |
| `pr-labeler.yml` (`pull_request_target`) | PR opened / edited / synchronised / reopened | Labels the PR only. It never fails a build. |
| `db-backup.yml` | `schedule` (03:00 UTC daily) + manual | Scheduled infrastructure job; never runs on a PR. |
| `dependency-scan.yml` | `schedule` (03:17 UTC nightly) + manual | Findings arrive as a scheduled report, not as a PR check. |
| `scheduled-payments-cron.yml` | `schedule` (every 5 minutes) + manual | Production cron sweep; never runs on a PR. |
| `scorecard.yml` (OpenSSF) | `branch_protection_rule`, `schedule` (Mondays 06:00 UTC) | Produces a security score for the default branch, not a merge gate. |
| `stale.yml` | `schedule` (Mondays 08:00 UTC) | Housekeeping on issues and PRs. |

## Path filters change which checks even appear

`prisma-ci.yml` and `contract-regression.yml` only start when a PR touches their
paths. That has two consequences for reviewers:

- A PR that edits only application code will show a shorter check list. That is
  expected, not a skipped check.
- A PR that touches `contracts/**` will additionally show
  `contract-regression`, which no other PR sees. If it is red, the PR is not
  mergeable, even though the generic list looks green.

`ci.yml` itself filters nothing: its four jobs run on every PR to `main`, and
the first three must be green before review is worth starting, because
`typecheck` and `unit-tests` fail fast on the most common mistakes.

## Documented checks vs. what actually runs

`CONTRIBUTING.md` describes a "15-job CI/CD pipeline". Only four of those jobs
exist as pull-request checks today, plus the two path-filtered workflows above.
The rest were either never implemented, were removed, or run on a schedule
instead. Until that table and this one agree, this section is the honest
mapping — please correct whichever side is wrong.

| Documented (`CONTRIBUTING.md`) | Reality in `.github/workflows/` | Local command |
| --- | --- | --- |
| 1. Lint — ESLint | No PR job runs ESLint. `npm run lint` exists and is part of `npm run ci`. | `npm run lint -- --max-warnings 0` |
| 2. TypeCheck — tsc | `ci.yml / typecheck` | `npx tsc --noEmit` |
| 3. Unit Tests — Vitest | `ci.yml / unit-tests` | `npx vitest run` |
| 4. Coverage — Vitest | No PR job. `npm run coverage` is local only. | `npm run coverage` |
| 5. Contract WASM Build | `ci.yml / contract-wasm` (+ `contract-regression.yml` for `contracts/**`) | see the table above |
| 6. Next.js Build | No PR job runs `next build`. | `npm run build` |
| 7. E2E — Chromium | No PR job. Playwright exists (`npm run test:e2e`) and E2E was removed from CI deliberately. | `npm run test:e2e` |
| 8. E2E — Firefox | No PR job. No Firefox project is configured. | — |
| 9. Prisma Validate | `prisma-ci.yml` (path-filtered to `prisma/**` and friends) | `bash scripts/validate-prisma-migrations.sh` |
| 10. Docker Build | No PR job. | `docker build .` |
| 11. K8s Validate | No PR job. `k8s/` manifests ship without a validating check. | — |
| 12. Helm Lint | No PR job. `helm/` ships without a validating check. | — |
| 13. Secret Scan — Gitleaks | `ci.yml / secrets-scan` (job name is `1️⃣2️⃣ Security — Secrets (Gitleaks)`) | `gitleaks detect --config .gitleaks.toml` |
| 14. npm Audit | No PR job. `dependency-scan.yml` runs on a nightly schedule + manual dispatch. | `npm run audit:deps` |
| 15. PR Auto-Label | `pr-labeler.yml` (labels only, never fails a build) | — |

Two further gaps worth recording, because the DoD references both:

- The DoD says "No typos (`typos` spell-check CI job passes)". There is no
  `typos` workflow. Only the `.typos.toml` configuration exists, so the check
  has to be run by hand: `typos` (with that config) — or the DoD sentence should
  stop claiming a CI job.
- The DoD and `CONTRIBUTING.md` both say "11 required CI checks". Four PR jobs
  gate a merge today (`typecheck`, `unit-tests`, `contract-wasm`,
  `secrets-scan`) plus the two path-filtered ones when they run. That number
  needs to come from **Settings → Branches**, not from this file.

## How to reproduce the gate before pushing

```bash
npm ci
npx prisma generate
npm run typecheck
npm run lint -- --max-warnings 0
npm test
npm run build
bash scripts/validate-deploy-config.sh
```

If your change touches the DB layer or the contracts, add:

```bash
bash scripts/validate-prisma-migrations.sh
cd contracts && cargo build --manifest-path ophirpay/Cargo.toml --target wasm32v1-none --release
```

Finally, check whether the repository's branch protection lists a check that
this document does not. That mismatch is a documentation bug: fix it in the same
PR that introduces the new check.