# CashCode project state

Last updated: 2026-08-28

This is a compact restart checkpoint, not a diary or full specification.
Accepted ADRs and task specifications in the private project repository remain
the canonical engineering source of truth.

## Project and direction

CashCode is a payment and trading platform with Go services, PostgreSQL, a
React/TypeScript web client, and a Kotlin/Compose Android device application.

CashCode v2 is a **selective rewrite**, not a full rewrite and not an in-place
refactor. The legacy backend is a frozen reference that keeps the development
stand working and documents existing behavior; it is not the v2 foundation.
New work belongs in independent `services/web`, `services/crypto`, and the
minimal wire-only `services/contract` module. None may import the legacy
backend.

Kept with cleanup: clients and non-financial product behavior, including auth,
users and permissions, requisites and devices, disputes, merchant API and
sandbox contracts, IPN, widget, fees, and exchange rates. Rewritten: backend
wiring and transaction boundaries, the financial data model, deal lifecycle,
ledger and holds, deposits, withdrawals, custody, and the financial UI. Legacy
wallet, queue, batch, MultiSend, and fragmented withdrawal mechanisms are to be
removed after v2 acceptance.

## Target financial architecture

The platform uses omnibus custody with an internal double-entry, append-only
ledger on Web. Trader, merchant, and platform balances are ledger accounts with
explicit holds. A successful deal settles as atomic internal ledger postings;
it does not create a per-deal TRON transfer. `transaction_queue` is a legacy
on-chain mechanism and must never become the new ledger.

On-chain activity is limited to trader deposits, sweeps, and outbound
withdrawals. A confirmed trader deposit credits the internal balance using a
durable unique identity, independently of whether its later sweep succeeds.

Web owns users, deals, the ledger, holds, business approval, durable work, and
inbound Crypto events. It never holds custody keys, signs, broadcasts, changes
Crypto policy, or exposes a path that can send a blockchain transaction.
Crypto is the sole automated signing and broadcast plane. It owns custody,
chain access, deposit monitoring and sweep, its request journal, Telegram owner
approval, payout whitelist, and hard payout limits.

The Web-to-Crypto transport is a durable pull model: Crypto claims work from
Web and sends acknowledgements, statuses, deposit events, and reconciliation
data back to Web's private API. Crypto has no inbound business API. Delivery is
idempotent and designed to survive retries, restarts, lost responses, and long
disconnects.

## Withdrawal invariants

- One immutable request ID and payload represent one merchant, trader, or
  treasury withdrawal. A changed payload under the same ID is a conflict.
- Web approval is a business gate; explicit owner approval in Telegram on
  Crypto is the only authorization to sign and broadcast.
- A withdrawal hold remains active until a terminal state proves Crypto can no
  longer execute. A cancellation request alone never releases it.
- `WAITING_FOR_FUNDS` is indefinite and retains the hold. Funds becoming
  available may prompt the owner again but never authorize an automatic send;
  a new explicit **SEND** action is required.
- Unknown broadcast outcomes remain unresolved with the hold active. Completion
  occurs after chain confirmation, not merely after broadcast.
- Payout whitelist and hard per-request, per-subject, global, type-specific,
  and hot-wallet safety policy belong to Crypto and cannot be changed through
  Web's business API.

## Current checkpoint

- **Branch:** `architecture/financial-core-redesign`
- **Completed phase:** `004-web-schema-baseline`, phase 3 of the accepted
  financial-core redesign program. An independent review and four focused
  re-reviews closed every accepted finding; the last re-review found nothing
  new.
- **Implementation:** commit
  `5acf3023f58416cfa1d152f089db6ced49f97c27`. The private branch is clean and
  synchronized with its remote.
- **Remote verification:** `v2` workflow run `33140604625` on that exact commit,
  concluded **success** across all three jobs (guards and contract, web,
  crypto).
- **Delivered scope:** the Web database now exists as versioned SQL in the
  repository. The disposable phase-002 `0001_bootstrap` migration is replaced by
  `0001_baseline` at the same migration version: **12 enum types, 27 tables, 3
  trigger functions and 10 trigger declarations**, with `deal` at **28 columns**
  and `requisite` at **29 columns**. The migration applies to an empty database
  and rolls back completely without `CASCADE`, carries no data, and creates no
  PostgreSQL extension. Integration tests pin the schema as an inventory -
  tables, enum labels and their order, columns with type and nullability,
  primary keys, unique and check constraints, index definitions, foreign-key
  targets and referential actions, and the trigger allowlist with its negative
  counterpart - so a later phase cannot add or lose an object silently.
- **Migration transition:** the replacement is at migration version 1, and the
  migration runner records version numbers rather than file contents. A
  development database that already recorded phase-002's version 1 will
  therefore **not** rerun it and will silently report a baseline it does not
  have. Every existing disposable phase-002 Web development database must be
  recreated with `make db-reset-web`. This is deliberate: there is no automatic
  guard, and the readiness path stays read-only and non-mutating.
- **Current/next task:** `005-port-auth-users` is the next phase in the accepted
  master plan. **It has not been started**, and no task 005 specification exists
  at this checkpoint.
- **Next action:** agree the task 005 specification - porting auth, MFA,
  sessions, roles, invites and permissions onto the new baseline, with the
  `/auth/*` contract unchanged - before implementation.

### Phase boundaries the baseline deliberately holds

Structure exists in the baseline where behavior does not. A table being present
is not evidence that its behavior is implemented.

- No ledger, accounts or holds before phase 008.
- No `requisite_block`, no `requisite_contragent_slot`, and no deal-flow policy
  before phase 009: requisite selection, limit spending, expired counters,
  auto-blocking and counterparty binding are all absent, and the tables that
  will carry them have no writer yet.
- The legacy financial-statistics triggers, wallet and queue structures,
  batch/MultiSend, per-deal on-chain settlement and the withdrawal model are not
  present and are not scheduled before their own phases.
- The baseline seeds no reference data. Ownership is fixed: bank display names
  belong to phase 006; tariff rates and entity fees belong to phase 007; each
  system configuration key is defined and supplied by the phase that introduces
  the behavior depending on it, with no bulk import of legacy configuration.

Completed architecture/specification milestones:

- ADR-001 accepted: selective rewrite, omnibus custody, Web/Crypto trust
  boundary, pull transport, ledger and hold invariants, deposit/withdrawal
  behavior, policy ownership, fresh databases, and phased delivery.
- Master task 001 accepted: component map, keep/rewrite/remove boundaries,
  detailed financial flows, phase order, acceptance criteria, and known legacy
  defects.
- Task 002 completed and remotely verified: its bounded v2 scaffolding and
  checks are implemented.
- Task 003 completed: the legacy deal and requisite rules, and the defects in
  them, are recorded under stable identifiers with the original SQL definitions
  committed as evidence. Later phases port rules from that record instead of
  reading a live legacy database.
- Task 004 completed and remotely verified: the non-financial Web schema
  baseline is versioned in the repository, and the decisions that shaped it are
  recorded in its specification - the user table rename, money precision
  separating quantities from rates, the deliberately narrow trigger allowlist,
  and the exclusions that keep later phases' architecture out of the baseline.

## Known traps

- Do not treat legacy balance mirrors or `transaction_queue` as an internal
  accounting system, and do not restore per-deal blockchain settlement.
- Do not import from the frozen backend into v2 or expand a scaffolding phase
  into later financial phases.
- Do not release a withdrawal hold based on Web timeout, a cancellation request,
  or any nonterminal/unknown Crypto state.
- Do not turn `WAITING_FOR_FUNDS` into delayed automatic execution.
- Do not place signing, custody material, Telegram approval, whitelist, or hard
  payout policy on Web.
- Legacy documentation can be stale. Verify current code and accepted phase
  documents for scoped work.
- The committed legacy SQL snapshot is historical evidence, not a migration and
  not the source of the v2 schema; the v2 schema is owned by the per-service
  migrations. Nothing in that snapshot is applied to any database or built.
- The rule and defect identifiers in the legacy rules document are referenced by
  later phases. Do not renumber them, and do not assume a rule applies on every
  code path: several are explicitly recorded as bypassed on some branches.
- Known legacy test/typecheck/toolchain failures are not automatically v2
  regressions; compare them with the relevant baseline and changed scope.
- A disposable development database created before the schema baseline will
  report the baseline as applied while lacking its tables. Recreate it rather
  than trying to migrate it forward, and do not add a runtime version guard for
  that one-time transition.
- The presence of a table in the baseline does not mean its behavior exists.
  Several tables ship with no writer at all until their phase arrives.
- Do not add product behavior to a structural or mechanical phase. Legacy shape
  is carried verbatim unless a deviation has recorded evidence; "the database is
  still empty so it is cheap now" is not evidence, and neither is "no current
  code path exercises it".
- A later phase that introduces a new invariant ships its own migration. An
  accepted and pushed baseline migration is not amended retroactively.

## AI development workflow

For substantial or cross-component work: run a no-code discovery/grill, perform
bounded investigation, resolve material questions, write and agree a task
specification, start implementation from fresh context using that spec as the
scope boundary, run proportional checks, obtain an independent code review,
fix accepted findings, and re-check. The agreed task file bridges sessions; old
chat history does not.

Commit and push in the private repository always require explicit owner
authorization. This public context is the one documented exception: once a phase
is fully complete and remotely verified, refreshing, sanitizing, committing and
pushing `PROJECT_STATE.md` is part of phase closeout and needs no further
confirmation, provided the change does nothing beyond bringing the public state
up to date. The authoritative rule is `docs/AI_WORKFLOW.md` in the private
repository.

## Required before production

These are outside the current financial-redesign scope but must not be
forgotten:

- Review and harden merchant public API authentication, signatures, replay
  protection, nonce/timestamp semantics, and compatibility/versioning without
  casually breaking its wire contract.
- Clean up Android secret handling and sensitive logging, remove unsafe
  fallback secret behavior and duplicated cryptographic logic, and add
  regression tests for critical notification/SMS parsing while preserving the
  device API cryptographic contract.

## Update policy

Rewrite this file after completion of a phase, a changed accepted architecture
decision, a branch or current-task change, or discovery/resolution of an
important known trap. When something stops being true, replace or remove it.
Do not preserve contradictory old decisions as history, and do not turn this
document into a changelog.
