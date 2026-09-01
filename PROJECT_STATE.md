# CashCode project state

Last updated: 2026-09-01

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
- **Completed phase:** `012-deposits-sweep`, phase 11 of the accepted financial-core
  redesign program, narrowed by owner decision to the deposit path alone.
- **Implementation:** commit `d8efecee94020a345477a1b62409fee3fa8f69f2`, preceded by the
  documentation commit `d836706a3d77e8b6318f53bae31ade0e14f45ec5` that recorded the split.
  The private branch is clean and synchronized with its remote.
- **Remote verification:** `v2` workflow run `33477125963` on `d8efece`, concluded
  **success** across all three jobs (guards and contract, web, crypto), first time,
  with no repair round.
- **The phase was split by the owner.** The master plan's phase-012 entry covered
  deposits and sweep. It now covers the deposit path only; sweep, TRX float, prefund,
  their signing, sweep thresholds and unswept alerts became a new phase
  `018-sweep-trx-float`, appended with the next free number so the numbers of already
  accepted phases were not shifted. Its dependency position is written out in words
  because a trailing number must not read as low priority: it runs after 012 and is
  mandatory before the cutover phase, before production, and before any point where
  the platform relies on automatic replenishment of the hot wallet from deposit
  addresses. The reason for the split is risk, not size - the deposit half signs
  nothing, while the sweep half introduces the first automated signing and broadcast
  path in the system, and the two should not share one review and one CI run.
- **Three threshold questions were deliberately left open** and carried into that
  phase: the sweep threshold and unswept alert levels, the TRX prefund size and float
  alert level, and the policy boundary for automatic sweep and prefund signing. The
  values suggested while splitting were explicitly **not** accepted and are not
  defaults. That phase first collects empirical cost, energy and resource evidence on
  a stand and only then returns those questions with measurements.
- **Delivered scope:** a trader deposit becomes a balance. Web enqueues the
  address command, Crypto derives and returns the address, the chain is watched for
  incoming USDT, transfers are identified by their on-chain identity, confirmations are
  tracked to the accepted depth of 19, and the confirmed deposit is credited to the
  internal ledger exactly once. Before this phase nothing credited a balance at all, so
  the deal flow existed but was never fed.
- **Nothing here signs or broadcasts.** The deposit tests assert that no transaction is
  built or sent anywhere on the deposit path.
- **The first inbound-event applier exists.** It performs the business write, the ledger
  credit and the applied marker in one transaction, so a credit cannot be applied twice
  or left half-done. The protocol journal had no applier at all before this phase.
- **Ownership follows the destination address.** A confirmed transfer belongs to the
  trader whose deposit address received it, regardless of sender - the platform's own
  hot wallet included. A withdrawal that lands back on the trader's own deposit address
  is credited again, because the withdrawal already debited it; the result is a
  pointless on-chain round trip and its fee, not a lost user balance. By owner decision,
  forbidding such a withdrawal - if ever wanted - belongs to the withdrawal policy layer
  and must never be implemented by withholding a credit for money actually received.
- **Reconciliation targets the two worst outcomes, not the tidy ones:** a deposit nobody
  discovered, surfaced as an unexplained address balance, and a confirmed deposit the
  ledger side refused to credit. A permanently refused event keeps being reported rather
  than dropping out of the checks.
- **Current/next task:** `018-sweep-trx-float` and `017-deal-read-and-disputes` are both
  unstarted and have no specification. By owner decision the next phase is never started
  automatically, so which one runs next is the owner's call.
- **Next action:** none in flight.

### Phase boundaries deliberately held

Structure exists in the baseline where behavior does not. A table being present
is not evidence that its behavior is implemented.

- The ledger is now fed. Deal creation places holds, confirmation settles, and a
  confirmed deposit credits a balance, so the deal path finally has money to work with.
  Nothing yet **debits** a balance to the outside: withdrawals are their own phase.
- The insurance forfeiture and the manual adjustment are primitives with no
  caller and no endpoint. Do not wire either to anything without the owner
  decision the phase-008 specification requires first - forfeiture moves a
  trader's collateral to the platform and no phase owns the rule for when that
  is allowed.
- The device-push and SMS notification pipelines now confirm deals; phase 006 had
  deliberately stopped them at the first access to the deal table.
- The merchant public API can create a deal. Neither dispute endpoint is ported, and
  no deal list, detail or read route exists for the trader, merchant, admin or
  support panels - only the sandbox answers a simulated one. Those belong to
  `017-deal-read-and-disputes`, together with the dispute subsystem beyond the
  acceptance path that confirmation itself needs.
- Requisite selection, limit spending, expired counters, auto-blocking and
  counterparty binding all exist from phase 009, with their two tables. The
  administrative routes for trader priorities remain deferred, while the selection
  semantics that read them are implemented.
- The legacy financial-statistics triggers, wallet and queue structures,
  batch/MultiSend, per-deal on-chain settlement and the withdrawal model are not
  present and are not scheduled before their own phases.
- The Web-to-Crypto transport now carries the deposit half. The outbox has one
  producer (the deposit-address command) and the event journal has one applier, which
  handles the address-created and deposit-confirmed events. The other three event types
  are stored and deliberately parked with no applier - withdrawal status and cancel
  rejection until the withdrawal phase, the swept event until `018-sweep-trx-float` - and
  a stored event nobody applies remains the designed state for those.
- The custody service now derives deposit addresses, watches the chain and tracks
  deposit confirmations, and its pull loop has real work to claim. It still has **no
  signing trigger**: the owner approval that alone authorises a signature belongs to the
  withdrawal phase, and sweep - the other thing that would sign - belongs to
  `018-sweep-trx-float`. Deposit-specific reconciliation now runs on a schedule on both
  sides. What moved to `018` is the daily **cross-system** job and the receiver of the
  hot-wallet and unswept sums, because the second of those sums does not exist until
  that phase introduces it.
- Two known limitations are inputs to the withdrawal phase rather than defects here, and
  both are unreachable while nothing can sign: a pre-signature refusal reaches no landing
  state at all - repeating without bound when it comes from recovery, and falling silent
  after one record when it comes from the send path - and a crash between approval and
  signature strands the request with no automatic re-drive. Both keep the hold, so money is frozen rather than
  lost, and both need the reachable path the withdrawal phase introduces before they
  matter.
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
  that first needs the answer. Phase 009 has since answered the second; the first
  is still open because no phase has yet needed it.
- Task 009 completed and remotely verified: deals can be created and confirmed.
  The phase was split by owner decision so that the money core carries its own
  review and CI; the read and dispute surface became `017-deal-read-and-disputes`.
  Thirteen owner decisions are recorded in its specification - ten substantive, three
  being renewals of the commit authorization - among them that a
  counterparty slot shortage may never refuse an already-accepted dispute, that
  dispute settlement is gated on available funds rather than raw balance, and that
  deleting a user who has ledger history is refused outright.
- Task 010 completed and remotely verified: the Web-to-Crypto protocol exists. The
  wire vocabulary is frozen as v1 in the shared module, the durable outbox and the
  inbound event journal are in place with their idempotency and reclaim rules, the
  private listener requires mutual TLS with pinned keys and has no plaintext mode, and
  each side has a fake of the other for tests, since the two are separate modules and
  no single test can hold both. No owner decision was required: the specification
  review and the code review resolved every question against the accepted
  architecture, and the two that touched money - the erased delivery evidence and the
  UUID demand on a custody-minted deposit identifier - were caught before they
  shipped.
- Task 011 completed and remotely verified: the custody core exists. Secrets have one
  source and no fallback, the two key domains are independent, the transaction id is
  computed locally and persisted before broadcast, and the service verifies the body it is
  about to sign instead of trusting the node that assembled it. The independent review
  found both the unverified-body hole and a window in which one request could be paid
  twice; the owner chose to close both inside this phase rather than defer them.
- Task 012 completed and remotely verified: trader deposits work end to end and become
  ledger balances. The phase was split by owner decision so that the deposit path, which
  signs nothing, would not share a review and a CI run with the first automated signing
  path; the sweep half became `018-sweep-trx-float` with an explicitly written, mandatory
  dependency position. Six owner decisions are recorded in its specification, the
  load-bearing one being that a confirmed transfer belongs to the trader by destination
  address regardless of sender - four substantive, the other two being the withdrawal and
  renewal of the commit authorization. Four independent specification reviews and three
  independent code reviews ran before it shipped, and every fix was checked by
  counterfactual to fail against the implementation it was meant to reject.

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
  system must call it. The ledger service is the only path that may **change** a
  balance or a hold, and a guard test fails if any Go code elsewhere names the five
  ledger tables - the fix for a "just this once" direct update is to add the
  operation to the service, not to widen the guard's owner list. Reads are narrower
  than that since phase 009: requisite selection asks for a trader's available funds
  through one dedicated SQL helper and may not touch the ledger tables itself. That
  helper is the only place outside the ledger's own migration where SQL may name
  them, and a separate guard enforces it, because the Go-side guard scans Go
  sources and cannot see SQL.
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
- Posting references are unique. The deposit phase discharged the obligation this
  once carried: the deposit reference is the transfer's on-chain identity, never the
  identifier the custody side minted, precisely because a custody database rebuild
  could re-issue that one and the credit would then be swallowed as a duplicate.
- The insurance hold is recalculated **unconditionally** after a deposit credit and
  never consults available funds, so the total held can exceed the balance and
  available simply clamps to zero. Making it respect available would change how
  much a trader may withdraw and is an owner decision, not a tidy-up. What an
  underfunded insurance hold blocks was settled by the deal-flow phase: nothing
  extra. Legacy gated only through the trader's available funds, with the unfunded
  remainder subtracted, and the ledger already reproduces that, so selection filters
  on available exactly where legacy did and no insurance-specific refusal exists.
- A settlement does not always have three legs: a zero-valued leg is omitted, so
  equal merchant and trader percentages produce two entries. The platform revenue
  account is also allowed to go negative, so an inverted tariff still settles. Code
  or tests that assume three legs, or a non-negative platform balance, are wrong.
  Amounts are quantised once and the merchant and platform legs derived by
  subtraction; independently rounding four values leaves a residue the ledger
  refuses.
- Once a trader has a ledger account, deleting that user answers a conflict rather
  than succeeding, and by owner decision it stays that way: a user with ledger
  history is not deletable, and the ledger rows are never cascaded or nulled to
  preserve the old answer, because that is deletion of financial records through
  user management. Removing such a user from an operational view is a separate
  archive-or-disable question, not a delete.
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
- Phase 009's traps are mostly about the deal transaction. Creation is **one**
  transaction that owns its commit, and every ledger call takes the caller's handle;
  splitting it, or calling the ledger on the pool instead, silently breaks the
  atomicity the whole financial model rests on. The lock order across creation,
  confirmation and expiry is fixed and deliberate: the ledger call is last, expiry
  locks the requisite row explicitly, and confirmation locks the counterparty row as
  its very first statement. Each of those two statements exists solely to prevent a
  deadlock cycle that no test can construct, so removing one as redundant reopens a
  production-only failure.
- Releasing a counterparty slot must be a state-column update, never delete-and-
  recreate and never a rewrite of its deal or counterparty reference: the database
  re-takes a parent lock only when key columns change, so the "equivalent" rewrite
  restores the very deadlock the ordering removes.
- A limit check and the spend it guards must read the same clock and the same locked
  status. Two clock samples across an interval boundary, or a check keyed on an
  unlocked read, each land the counter one deal above its limit. Both were real
  defects here, not hypotheses.
- A deal can reach success without a counterparty binding, and that fact is recorded
  rather than silent. The money outcome matches legacy, which also failed to bind and
  ignored the failure; what changed is that the absence is now observable. Do not
  "fix" it into a refusal: an owner decision states that a routing capacity limit
  never blocks settlement of an accepted dispute.
- The sandbox and the administrative rate preview keep the old hardcoded conversion
  factor deliberately. The live path never used it - it computed and discarded it -
  so the two are not inconsistent, and aligning the sandbox would change a contract
  the redesign declares unchanged.
- Tests that verify a race must be shown to fail without the fix. Several defects in
  this phase were caught only because a test was mutated to prove it discriminated;
  a green test that has never been made to fail proves nothing about the invariant it
  claims to hold.

- Phase 010's traps are about a wire that is now frozen. **A refusal on this protocol
  is permanent**: the sender retries the same message forever, so every validation rule
  is also a way to stall the queue, with no operator notified anywhere. The settled
  split is three-way, and review moved it twice before it was right: the envelope is
  always checked; the payload of a **known** type is checked narrowly and forward-
  compatibly - required fields only, unknown fields ignored - because a wrong amount on
  a deposit event is money and must not be accepted with a log line; the payload of an
  **unknown** type is not inspected at all. Labels are never checked against a closed
  set, since the phase that owns the withdrawal state machine will mint labels this
  phase has never seen.
- An acknowledgement means **durably stored**, not applied. Anything that treats an
  ack as "handled" silently drops events now that an applier exists, and an event whose
  application is permanently impossible is still acknowledged on purpose, so the queue
  does not stall behind it - it is marked terminal and logged instead. There is no
  alert transport on Web to carry that logged failure; the cutover phase owns it.
- **Do not erase the evidence that work was delivered.** Returning an expired claim to
  the queue must clear only the lease, never the first-claim stamp or the delivery
  counter. The withdrawal phase decides whether it may release a hold by asking
  whether the custody side ever took the command, and a queue state alone cannot
  answer that: the predicate is the first-claim stamp, not the state column.
- The identifier a deposit event carries is minted in the custody database and is
  deliberately **not** required to be a UUID - the ledger phase chose an opaque
  reference for exactly that reason. The on-chain identity travels alongside it, and the
  crediting phase used it to build a reference no database rebuild can re-issue. Demanding a
  UUID there rejects genuine confirmed deposits forever, which is how that rule was
  caught in review rather than in production.
- Mutual TLS on the private listener has **no off switch by design**, so both
  long-running binaries refuse to start without certificate material, and the custody
  client's default endpoint is https. This is deliberate and the cutover phase owns
  issuing the material; the administrative CLI keeps working without it, because
  migrations must run before any of that exists. An environment-keyed bypass on a
  custody boundary is the defect class this project has already been bitten by.
- Three limits bind the phases that come next, and each is enforced by a refusal, which
  on this wire means a permanent stall if the other side exceeds it: the pull loop may
  not build an event batch larger than the frozen maximum, the reconcile request list
  has its own cap, and the deposit-address command must derive its request id
  deterministically from the trader, or a retry mints a second command and a second
  address and breaks "one address per trader".
- The two services are separate modules and a workspace file is forbidden, so **no
  single test can run both sides in one process**. Every cross-service test is one
  real side against a fake of the other; a test that appears to wire the two together
  is wiring a fake, and the two fakes are shared fixtures the later custody phases
  build on.
- **Never retry a broadcast.** Every other node call may be retried; the one that hands
  a signed transfer to the network may not, because a retried broadcast is how a lost
  answer becomes a second payment. A lost answer is resolved by looking the transaction
  up, never by sending it again.
- **Compute the transaction id locally and persist it before broadcasting.** It is the
  hash of the body being signed, so it is knowable before the network sees anything.
  Persisting it first is what makes a crash recoverable without a second transfer, and
  it is the accepted fix for the legacy defect that marked a withdrawal complete at
  broadcast.
- **Never sign a body on the assembling node's word.** Checking that the node's returned
  id matches the hash of the node's returned body only proves the answer agrees with
  itself. The body must be decoded and its operation type, token contract, recipient,
  amount, sender, fee bounds and action count checked against the authorised request,
  because a payout whitelist is checked against the request while the signature binds
  the bytes.
- **A request whose outcome is unknown keeps its hold.** Unknown-outcome and
  in-flight states are non-terminal by design; only a terminal state releases or
  consumes money. Any later manual-resolution tooling must not mint a replacement for a
  request whose previous transaction may still be accepted, and must reconcile the
  previous transaction id first.
- **A body may be replaced only after it can no longer be accepted** - its expiry plus
  the node timeout - and never while its outcome is still in flight. This is the rule
  that keeps two potentially valid payouts for one request from existing at once, and
  the timeout that feeds it is ordinary configuration with no enforced relation to the
  lifetime the node chooses, so raising it far enough reopens the window.

- Phase 012's traps are about not losing money that is already on the chain. **A
  discovery source must declare its completeness.** The index either answers completely
  for the requested interval or marks its answer truncated and says how far it is
  complete, and the scan cursor advances only to that boundary minus an overlap - never
  to the newest timestamp it happened to see. A paginated, newest-first source plus a
  cursor that trusts it silently loses the oldest transfers in a busy window, and the
  only thing that would ever notice is a balance check.
- **Measure confirmation depth against the block the current receipt names**, and rewrite
  the stored block while the deposit is still unconfirmed. Freezing the block discovered
  first credits a deposit early whenever a short reorg re-includes the transaction at a
  higher block, at fewer than the accepted depth - and a short reorg never trips the
  "transaction disappeared" condition, so nothing else catches it. The payout tracker
  already had this right; the deposit path had to be brought into line with it.
- A deposit that was orphaned because the node could not find it is **not** terminal: a
  re-discovered transfer returns it to the unconfirmed state, and the return must refresh
  the block and recount the depth from scratch. Its uniqueness constraint blocks creating
  a second row, so without the return path a real deposit could become permanently
  uncreditable.
- The applier does the business write, the ledger credit and the applied marker in **one**
  transaction. Splitting them, or calling the ledger on the pool, breaks the property the
  whole crediting path rests on. An event that can never be applied is still acknowledged,
  marked permanently failed and re-reported on every reconciliation pass, because the
  money is really on chain and really uncredited - one log line at the moment of failure
  is not enough.
- The ledger reference for a deposit is the transfer's **on-chain identity**, never the
  identifier the custody side minted: a custody database rebuild could reissue that one,
  and the credit would then be silently swallowed as a duplicate.
- The deposit-address command's request id is derived deterministically from the trader.
  A fresh identifier per call mints a second command and a second address, and breaks the
  ordering scope the protocol uses for the resulting event.
- Reconciliation must survive its own reporting rules. A check that reports once and goes
  quiet, or a query that drops an object as soon as it is refused rather than while it is
  still uncredited, produces exactly the silence the check exists to prevent. A check
  gated on a staleness threshold also must not be written as if it reported immediately -
  two comments claiming that were corrected in review because their own tests asserted
  the opposite.
- Deposit addresses have **no HTTP route** yet; that surface belongs to the frontend
  finance phase. The path is complete but not reachable from the panel, so a stand cannot
  demonstrate it end to end until then. This is the accepted boundary, not a gap.
- A test that renames a table away to simulate a failure is safe only because every
  fixture runs on its own ephemeral database that is dropped afterwards. Do not copy that
  pattern into a suite that shares one database.

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
