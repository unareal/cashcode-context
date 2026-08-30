# CashCode project state

Last updated: 2026-08-30

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
- **Completed phase:** `008-ledger-core`, phase 7 of the accepted financial-core
  redesign program.
- **Implementation:** commits `7225ed046daa0a2b3e5865d1e7a54a88e9d2a354` (the phase)
  and `2365153973d25c9bfe0751b59a3c413061b8ebc7` (a CI repair, below). The private
  branch is clean and synchronized with its remote.
- **Remote verification:** `v2` workflow run `33300831608` on `2365153`, concluded
  **success** across all three jobs (guards and contract, web, crypto).
- **Delivered scope:** the internal double-entry ledger now exists in `services/web`.
  One migration adds exactly the five tables the schema-baseline phase assigned here -
  ledger accounts, materialized balances, postings, entries and holds - with their enum
  types, foreign keys that all refuse cascading deletes, and the fixed platform and
  external counter-accounts seeded. A ledger service is the single path that may change
  a balance or a hold, enforced by a test that fails if any SQL elsewhere in that
  module touches those five tables. Every operation takes the caller's transaction handle, so the deal
  creation of phase 009 can run selection, the availability check, the hold, the
  counterparty slot, the limit spend and the insert as one transaction.
- **Posting legs sum to exactly zero at ledger precision by construction, not by luck
  of rounding.** The two fee values are quantized first and all three legs are derived
  from the quantized values, so no leg is independently rounded. A zero-valued leg is
  omitted rather than written, which means a settlement whose merchant and trader
  percentages are equal records two entries instead of failing; and the platform revenue
  account may go negative, so an inverted tariff still settles rather than stranding a
  confirmed deal. Both of those were found by specification review before any code
  existed - as written first, the schema could not have recorded either case.
- **Available funds now have one definition** - balance minus the sum of active holds,
  clamped at zero - replacing the five mutually inconsistent expressions the legacy
  system maintained, one of which silently omitted deal reservations and was the
  concrete mechanism of balance drift. The availability check and the hold happen in one
  transaction, with balances locked in a fixed order and the hold total read after the
  lock is taken. That ordering is the whole substance of the fix for the legacy race
  where two concurrent deals could both pass a balance check.
- **Every money event is idempotent by its reference.** A repeat returns the existing
  posting and moves nothing; the same reference carrying a different set of legs is
  refused rather than silently overwritten, matching the accepted rule that money is
  never auto-corrected. The discriminator compares the full set of accounts and signed
  amounts, so a repeat that carries the same total but is addressed to a different
  account is caught rather than accepted as a duplicate.
- **Insurance became a hold instead of a transfer.** In the legacy system the unpaid
  remainder of a trader's insurance was moved on-chain into a platform wallet, so it
  left the trader's ownership. The accepted architecture reverses this: the sum stays the
  trader's property and is only blocked. The hold is recalculated after every deposit
  credit, unconditionally - it never consults available funds, so the total held may
  exceed the balance and available simply clamps to zero. What should be *gated* on an
  underfunded insurance hold is deliberately not decided here; it belongs to the deal
  flow.
- **Two primitives ship deliberately unreachable.** The insurance forfeiture and the
  manual adjustment have no caller, no endpoint and no automatic path, so nothing in
  this phase can move a trader's money. Forfeiture in particular has no legacy
  behavior to port and no phase owns it: what event forfeits a trader's collateral and
  who may authorize it is undecided, and the specification obliges the first phase that
  wants to call it to obtain that decision before doing so.
- **A required CI run went red and was repaired.** The failure was in a pre-existing
  test of the merchant authentication suite, unrelated to the ledger: the way it
  constructed an invalid value was not guaranteed to differ from the valid one, so it
  intermittently asserted a rejection against a request that had legitimately
  succeeded. An independent review confirmed the product itself was correct before the
  repair was pushed. The fix is confined to the test, which now fails loudly if the
  value it alters is not actually different.
- **Current/next task:** `009-deal-flow-v2` is the next phase in the accepted master
  plan. It has not been started, and no specification for it exists at this checkpoint.
- **Next action:** run phase 009 - requisite selection, atomic deal creation, the single
  deal-confirmation path, expiry, admin actions, settlement, counterparty slots and
  requisite blocking.

### Phase boundaries deliberately held

Structure exists in the baseline where behavior does not. A table being present
is not evidence that its behavior is implemented.

- The ledger exists from phase 008, but it has no production caller yet. Nothing
  in the running service creates an account, credits a deposit, places a hold or
  settles a deal: the ledger service is not wired into the composition root, and
  its only callers are tests. Balances are therefore all zero and will stay so
  until phase 009 creates deals and phase 012 credits deposits.
- The insurance forfeiture and the manual adjustment are primitives with no
  caller and no endpoint. Do not wire either to anything without the owner
  decision the phase-008 specification requires first - forfeiture moves a
  trader's collateral to the platform and no phase owns the rule for when that
  is allowed.
- Requisites, devices and their parsers are delivered by phase 006. Deal flow is
  not: the notification path stops before deal matching, so nothing in phase 006
  confirms a deal, releases funds or dispatches an IPN.
- The merchant public API exists but cannot create a deal, and neither dispute
  endpoint is ported: atomic deal creation belongs to phase 009, and the money
  beneath it now exists. It cannot create a real deal; only the
  sandbox answers a simulated one. The authentication middleware for that surface is
  mounted and tested while its only business route is still absent.
- No `requisite_block`, no `requisite_contragent_slot`, and no deal-flow policy
  before phase 009: requisite selection, limit spending, expired counters,
  auto-blocking and counterparty binding are all absent, and the tables that
  will carry them have no writer yet.
- The legacy financial-statistics triggers, wallet and queue structures,
  batch/MultiSend, per-deal on-chain settlement and the withdrawal model are not
  present and are not scheduled before their own phases.
- The baseline seeded no reference data. Bank display names were supplied by
  phase 006, and the global fee tiers, the tariff grid and this phase's
  configuration keys by phase 007. Each system configuration key is still defined
  and supplied by the phase that introduces the behavior depending on it; there is
  no bulk import of legacy configuration, and a key the source did not hold stays
  on its built-in default.

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
- Still in force from task 005: a trader or merchant created in v2 keeps the
  legacy insurance minimum instead of zero, as a fixed value, so nobody exists
  without an insurance requirement. The ledger of phase 008 now reads that value
  and holds it.
- Task 006 completed and remotely verified: requisites, devices, device logging and
  the bank-notification parsers are ported with the device cryptographic contract
  unchanged, the bank catalogue is seeded, and every deliberate departure from legacy
  is recorded in the specification with the evidence behind it.
- Task 007 completed and remotely verified: the merchant API, sandbox, widget, fees,
  exchange rates and system configuration are ported, IPN delivery became a durable
  outbox, the reference data was seeded from the recorded historical artifact and
  cross-checked against it independently, and six departures from legacy were
  decided by the owner rather than by the implementation.
- Task 008 completed and remotely verified: the double-entry ledger, holds and
  materialized balances exist behind a single service, every money event is
  idempotent by reference, available funds have one definition instead of five,
  and insurance became a hold the trader still owns. Two owner-level questions
  were identified and deliberately left undecided rather than answered by the
  implementation - what forfeits a trader's insurance, and what deleting a user
  with ledger history should do - each recorded as an obligation on the phase
  that first needs the answer.

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

- Phase 006 added its own list of deliberately carried quirks, each pinned by a
  test: several device-management
  routes have no ownership check at all, so any authenticated user can cancel and
  thereby delete another user's unbound device, read any requisite's notifications,
  and rewrite the status of any notification, while a team lead can read a device
  card outside their own referrals; a missing
  device header answers 400 while a bad credential answers 401; the create and
  rebind responses spell the QR field differently; validation tags on the requisite
  create and update bodies are never evaluated, so almost nothing is validated at
  bind time; the trader requisite list reports the page length where the admin list
  reports a real count; and the SMS-Box slot lookup has no status filter, so a
  blocked or pending requisite still matches. Removing any of them is a defect, not
  a cleanup.
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

- Device authentication depends on a signature over the **original raw bytes of the
  request body**. Re-serializing or normalizing the JSON before verifying the
  signature - including verifying after a framework binding has already consumed and
  re-encoded the body - is incompatible with the Android clients already paired in
  the field, and must not be done without a separate, deliberate protocol migration.
  It breaks every paired device while looking correct in review.
- Four of the notification statuses the SMS pipeline can produce are not values the
  status column accepts, so those rows are silently never stored. That is legacy
  behavior and is carried, which is only safe because the notification write is
  deliberately its own statement outside any transaction that must still commit.
  Put it back inside one and a blocked SMS would stop blocking the requisite.
- The bank catalogue is not decoration. Requisite creation validates the submitted
  bank against it, and one of its flags decides whether a trader may create an SBP
  requisite, so an empty or invented catalogue changes what traders can do.
- Configuration defaults are part of the ported behavior. The legacy backend
  supplies built-in defaults for the Android release descriptor, so "unset" never
  reaches its handler; an empty default in v2 would tell every current device that
  an update is available, with no version and no link. Where legacy has a default,
  carry it, and cite the line it came from.
- Do not add a device-log table because the ingestion endpoint appears to need one:
  the bodies are deliberately not stored (see the checkpoint above).

- Phase 007 added its own carried quirks and traps. Money and rates cross the wire as
  **quoted JSON strings with trailing zeros trimmed**, so a stored `1000.00` is emitted
  as `"1000"`; a test or client expecting the stored scale, or a JSON number, is wrong
  rather than the code. The sandbox settlement amount is computed in floating point and
  only then converted, so it carries float artifacts verbatim - the phase that
  implements real settlement must not inherit that chain when it implements the accepted
  identity of amount, fees and settled sum.
- The typed configuration read path is cached for five minutes in v2 where legacy read
  through on every call. A configuration change made outside the process - by direct SQL,
  or by the admin route on another instance - therefore takes effect up to five minutes
  later, on every key except the three widget ones, which keep an uncached path. The
  writing process refreshes its own cache, so the admin route stays self-consistent.
- The seeded fee tier grid has **deliberate coverage gaps** carried from the source: an
  amount falling between two tiers matches none of them and drops to the configured
  default percent. Closing the gaps would change what merchants are charged and is an
  owner decision, not tidying. The tier validation rejects genuine overlaps only.
- The IPN history endpoint always answers an empty list, exactly as legacy does even
  though v2 now has durable delivery state behind it. Serving real rows from the outbox
  would be new merchant-visible behavior.
- Widget expiry has a write on the refusal path: a read that discovers an expired token
  deactivates it, so the first read answers "not found" and the next answers a server
  error for an inactive token. That write must **commit even though the read is
  refused**. Wrapping the check in a transaction that rolls back on refusal discards the
  deactivation and leaves the token live forever, while looking correct in review - this
  is exactly the defect the first CI run caught.
- IPN retry pauses are quantised to the dispatcher's polling interval, so the shortest
  legacy pauses are unreachable and each pause is the legacy pause rounded up to a tick.
  Delivery is at-least-once by construction: a process that dies after claiming a row
  retries it when the lease expires.
- The admin fee routes render every service error as a server error, including a
  rejected tier overlap, and one legacy error message on the bank-fee routes is
  unreachable because the comparison that would select it never matches. Both are
  carried deliberately. The merchant public API and the sandbox are registered on the
  engine rather than under the panel's route group; moving them under it changes
  observable behavior for live integrations, and middleware coverage for that surface
  is a production-hardening item rather than a porting-phase change.

- Phase 008 added the ledger, and its traps are mostly about how the rest of the
  system must call it. The ledger service is the only path that may change a
  balance or a hold, and a guard test fails if any SQL elsewhere in that module
  touches the five ledger tables - the fix for a "just this once" direct update is to add the
  operation to the service, not to widen the guard's owner list.
- Every ledger operation takes the **caller's** transaction handle and does not
  commit. That is required so deal creation can run selection, the availability
  check, the hold, the counterparty slot, the limit spend and the insert as one
  unit - but it means the caller owns the commit, and a forgotten one silently
  discards money movement that looked applied.
- Every deposit, settlement and withdrawal takes an exclusive row lock on a shared
  counter-account balance for the duration of the caller's transaction, so all
  deposits serialize against each other, as do all settlements and all
  withdrawals. Do not wrap slow work - a network call, an external API, a long
  scan - around a ledger call.
- Posting references are unique, and one of them is an identifier minted in the
  custody service's own database. If that side ever re-issues an identifier, the
  event is silently accepted as a duplicate and the money is never credited. The
  phases that build the deposit path must make the reference globally unique or
  prove re-issue is impossible.
- The insurance hold is recalculated **unconditionally** after a deposit credit and
  never consults available funds, so the total held can exceed the balance and
  available simply clamps to zero. Making it respect available would change how
  much a trader may withdraw and is an owner decision, not a tidy-up. What should
  be blocked when insurance is underfunded is deliberately still undecided.
- A settlement does not always have three legs: a zero-valued leg is omitted, so
  equal merchant and trader percentages produce two entries. The platform revenue
  account is also allowed to go negative, so an inverted tariff still settles. Code
  or tests that assume three legs, or a non-negative platform balance, are wrong.
- Once a trader has a ledger account, deleting that user answers a conflict rather
  than succeeding. The phase that first creates accounts for real must obtain the
  owner's decision on what deleting a user with ledger history should do; deleting
  the ledger rows to preserve the old answer is deletion of financial records
  through user management.
- Balance rows are created lazily by the first operation that locks an account, not
  when the account is created. A test that asserts against a balance row without
  running a real operation first updates zero rows, sees no error, and proves
  nothing, while appearing to assert something.
- A test that constructs an "invalid" value must guarantee it actually differs from
  the valid one. A pre-existing authentication test altered a value in a way that
  was occasionally a no-op, and so intermittently asserted a rejection against a
  request that had legitimately succeeded. It failed a required CI run
  intermittently.
- Before recording an acceptance check as unrunnable in this environment, re-read
  the project instructions rather than concluding from a failed connection attempt.
  Phase 007 deferred its database-backed verification to CI on the belief that the
  integration database was out of reach; phase 008 found it reachable and verified
  every criterion locally before committing.

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
