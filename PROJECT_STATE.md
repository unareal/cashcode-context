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
- **Completed phase:** `005-port-auth-users`, phase 4 of the accepted
  financial-core redesign program. The specification went through seven review
  rounds before acceptance; the implementation through three.
- **Implementation:** commit
  `f459d1552effd3290a88580fbf5b8c69f29f3066`. The private branch is clean and
  synchronized with its remote.
- **Remote verification:** `v2` workflow run `33173651582` on that exact commit,
  concluded **success** across all three jobs (guards and contract, web,
  crypto).
- **Delivered scope:** the auth, MFA, session, users, roles, invites, team-lead
  referral and trader-permission surface now lives in `services/web`, on the
  phase-004 schema. The thirteen `/auth/*` routes and the admin, support-admin,
  team-lead and trader routes reproduce the legacy wire contract - status codes,
  error codes and message strings - with whole-body assertions in tests. New in
  the module: Argon2id password hashing, HS256 access tokens with a refresh
  token that is not a JWT, TOTP with replay protection, the admin IP allowlist,
  the first repository layer and transaction boundary, the user and invite audit
  path, and the auth middleware and rate limiters. The WAF wrapper, the
  Redis-backed session service, the development bypass group and the dead
  handlers the master task lists for removal are not ported, and a test scans
  the module to keep them out. No migration was needed: every table the phase
  uses already exists in the schema baseline.
- **Deliberate deviations from legacy, all three decided by the owner:**
  "log out from all devices" now actually revokes the stored sessions, where the
  legacy endpoint returned success and revoked nothing; a trader or merchant
  created in v2 keeps the legacy insurance minimum instead of zero, so the
  future ledger does not find them with no requirement at all - now as a fixed
  value, because the legacy configuration table it used to come from is not part
  of the v2 schema; and a failure to
  persist a session during login, registration, refresh or MFA confirmation
  fails the request instead of returning a token that would never work.
  Everything else is carried verbatim, including a documented list of legacy
  quirks that an idiomatic rewrite would remove by reflex. Two further
  observable differences follow from the master plan itself rather than from a
  choice: the IP-level rejections disappear from `/auth/*` (see the
  traps below), and a merchant registered in v2 gets no default tariff rows
  until phase 007 supplies them.
- **Current/next task:** `006-port-requisites-devices` is the next phase in the
  accepted master plan. It has not been started, and no specification for it
  exists at this checkpoint.
- **Next action:** run phase 006 - requisites, devices, device logging and the
  SMS/push parsers, with the device HMAC and QR contract unchanged.

### Phase boundaries deliberately held

Structure exists in the baseline where behavior does not. A table being present
is not evidence that its behavior is implemented.

- No ledger, accounts or holds before phase 008.
- Requisites, devices and their parsers belong to phase 006; phase 005 touches a
  requisite only to carry over the one cascade that revoking an inside-bank
  permission has always performed.
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
- Task 005 completed and remotely verified: the non-financial auth and user
  surface is ported, the removals the master task requires are done and
  enforced by a test, and the three behavior changes worth making were decided
  by the owner rather than by the implementation.

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
- In a porting phase the reflex to tidy is the main hazard. Several legacy
  behaviors are deliberately preserved and individually recorded: a refresh
  response whose "expires in" field carries an absolute timestamp; two invite
  endpoints answering in different key styles; an empty list serialized as null;
  an administrator search by id that can never match; invalid enum values
  answering 500 rather than 400; and an access token issued at registration
  that every protected route rejects. Removing any of them is a defect, not a
  cleanup - and deciding to fix one is an owner decision, because the accepted
  defect-fix list covers only deals, requisites, the ledger and withdrawals.
- The audit path never serializes a whole row. It writes an explicit allowlist
  of safe columns, and a test proves password hashes, MFA secrets and unconsumed
  invite codes never reach the audit table. Do not "simplify" it into a row
  dump, and do not add a read-back to make an audited value look tidier.
- Web has no Telegram bot token by accepted architecture, so the alerts legacy
  sent on account lockout and on a blocked administrator login have no transport
  in v2. The controls themselves are preserved; only the notification is gone.
- Redis must fail in two different directions, and the
  difference is the point: an unconfigured client validates one-time codes
  without replay protection, while a configured but failing one rejects them.
  With it unreachable, no account with MFA can sign in - administrators
  included - while the per-account lockout quietly stops applying, because it
  fails open in the same situation. That is faithful to legacy and is an availability question for the
  cutover, not a bug to paper over.
- The administrator address allowlist is empty by default, and an empty
  allowlist means the restriction is simply skipped, with nothing warning that
  it is missing. Carrying the deployed list over is a precondition of the
  cutover phase, not an optional step. When only per-login rules are used, a
  login with no rule of its own is allowed through, so either every
  administrator is listed or the global list is used - a non-empty global list
  applies to everyone.
- Removing the legacy request filter has consequences beyond the filter: request
  bodies on registration and on the administrator routes are no longer bounded
  by the application, passwords may now contain characters the filter rejected,
  and its length limits counted bytes where the replacement validation counts
  characters. With the IP-level protections gone, the `403 ip_blocked` and
  `429 too_many_attempts` responses no longer exist on `/auth/*` either; the
  per-account lockout and the route rate limiters remain.

## AI development workflow

The workflow is now autonomous phase orchestration. The owner is not a message
relay between AI sessions: the main session orchestrates a phase end to end,
delegates bounded work to subagents, and returns to the owner only for a
decision that is genuinely the owner's.

A phase runs as `/run-phase <phase>`, or `/run-phase <phase> --ship`. The
lifecycle is: load project state, resolve the phase against the accepted master
plan, bounded investigation, internal-first grill, owner gate, draft the phase
specification, independent specification review, automatic fixes and focused
re-review until clean, owner gate, implement, acceptance checks, independent
code review, classify findings, automatic fixes and focused re-review until
clean, commit and push if preauthorized, required/applicable remote CI, phase
closeout, then stop. The next phase never starts automatically.

Every uncertainty is classified **AUTO**, **OWNER** or **ESCALATE**. AUTO is
ordinary engineering work the session must settle itself - internal structure,
naming, migration mechanics, indexes, constraints implied by accepted
invariants, tests, fixtures, factual corrections, objectively correct reviewer
fixes. OWNER is reserved for money movement and availability, merchant, trader
or user-visible behavior, who may perform or approve an operation, custody and
withdrawal semantics, trust boundaries, whether to preserve a legacy rule where
that was never decided, policy thresholds, compatibility breaks, infrastructure
trade-offs, accepted architecture, and material scope expansion; unresolved doubt in those
categories resolves to OWNER, not to AUTO. ESCALATE is a conflict between
authoritative documents, which is presented rather than silently resolved.

Discovery is **internal-first**: an uncertainty is answered from the phase task
and master plan, the ADRs, newer accepted phase artifacts, current v2 code,
recorded legacy evidence, or bounded investigation, in that order. Only OWNER
and ESCALATE items reach the owner, batched rather than one at a time.

Review is independent and the fix loop is autonomous. The agent that writes an
artifact never reviews it; a reviewer never edits what it reviews; a finding is
never closed as invalid by its author, and a reviewer's OWNER or ESCALATE
classification cannot be downgraded by another agent into an ordinary fix. Objective findings
are fixed and re-reviewed automatically, with bounded cycles and a stagnation
rule, so the owner is not consulted between normal iterations.

Remote CI is a closeout gate when it is **required and applicable**: the final
diff touches a path the workflow filter selects and the branch matches its
trigger. Otherwise it is recorded as N/A with the reason. An unrelated workflow
is never triggered to manufacture a green check.

Commit and push in the private repository always require explicit owner
authorization. The one preauthorization is phase-level: `--ship`, typed
literally in the invocation, authorizes committing and pushing the bounded
result of that phase only, after every gate is clean. It is never inferred,
never inherited from an earlier run, and becomes invalid if an owner decision
changes the accepted scope, an ADR must change, or the phase expands.

This public context is the documented exception to the push rule: once a phase
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
