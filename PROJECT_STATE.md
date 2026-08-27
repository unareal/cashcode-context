# CashCode project state

Last updated: 2026-08-27

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
- **Completed phase:** `003-legacy-deal-rules-doc`, phase 2 of the accepted
  financial-core redesign program. An independent review and two focused
  re-reviews closed every accepted finding; the last re-review found nothing
  new.
- **Implementation:** commit
  `fb656e952739a004ee7dac7c2ab9eabaebd90040`. The private branch is clean and
  synchronized with its remote.
- **Remote verification:** none was triggered, by design. The `v2` workflow is
  scoped to the v2 services, the root Makefile, the guards script and its own
  definition; this phase is documentation-only and touches none of those paths,
  so no run was created. That is the configured behavior, not a failure.
  Acceptance was established locally instead: the full v2 build, vet, unit test
  and structural guard suite passes and no v2 path was modified. The most
  recent successful `v2` run remains run `33035472379` on the phase 002 commit.
- **Delivered scope:** a verified record of how the legacy system actually
  executes deal and requisite rules - 96 rules under stable identifiers in nine
  namespaces, plus 22 confirmed defects - together with a frozen schema-only
  SQL snapshot, a curated per-object slice cut verbatim from that snapshot, and
  a reproducible export script that structurally identifies its source,
  generates its own provenance and publishes atomically. The phase changed no
  product code and fixed no legacy defect.
- **Current/next task:** `004-web-schema-baseline`. It has not started, and no
  task 004 specification exists at this checkpoint.
- **Next action:** agree the task 004 specification - the non-financial Web
  baseline schema and its statistical triggers - before implementation.

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
