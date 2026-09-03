# CashCode project state

Last updated: 2026-09-03

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
  Crypto is the only authorization to sign and broadcast a payout.
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
- **Completed phase:** `019-dispute-mechanism`, phase 18 of the accepted financial-core
  redesign program, followed by the bounded pre-cutover cleanup `020-constructor-trader-list`.
- **Implementation:** commit `a126ebb` on top of `1c1377705228506972f34a43deda8db3e3139e88`.
  The private branch is clean and synchronized with its remote.
- **Remote verification:** `v2` workflow run `33775802297` on `a126ebb`, concluded **success**
  across all three jobs (guards and contract, web, crypto), first time, with no repair round.
  The preceding phase was verified the same way by run `33755833210`.
- **What this phase delivers: a disputed deal can now be judged, and judging it moves money.**
  The read surface of the previous phase could show that a dispute existed but nothing could
  open, answer or settle one. The whole mechanism is now on the v2 API: creation from the
  panel, from the merchant public API and from the sandbox; the read lifecycle for every role;
  accept and reject; evidence documents; the dispute-specific merchant notification; and the
  operator and administrator actions. Fifty-three routes and one migration.
- **Accepting a dispute is the only operation here that moves money, and it reuses the deal
  confirmation path rather than a second one.** The deal-flow phase shipped that path
  deliberately without a caller; this phase supplies it. Settlement, holds, limits and
  counterparty slots are not rebuilt, so a dispute settles through exactly the same accounting
  as an ordinary confirmation. Its two refusals are the requisite limit and insufficient
  available funds; a counterparty slot shortage never refuses an accepted dispute. The dispute
  transition, the settlement and the requisite auto-block commit in one transaction, so no
  state exists in which a dispute is accepted while its deal is unsettled.
- **Evidence files live in the database, not on a service's filesystem.** That was an owner
  decision, and it removes a class of failure rather than a class of work: backup and restore
  cover the evidence automatically, a redeployed or moved service loses nothing, and no
  document row can outlive the bytes it points at. Uploads are streamed with an explicit
  per-part bound and a request-size cap, and the ways a web framework silently spills a large
  upload to a temporary file are forbidden by name - the prohibition had to be corrected once,
  because its obvious form missed the one function that actually does the spilling, and what
  enforces it is described under Known traps.
- **Deleting evidence is soft, always.** Bytes, metadata, hash, uploader identity and history
  survive; no route removes a row; a deleted document is simply not found on
  every read path including the administrator's. The uploader may remove its own upload and an
  administrator may remove any, but nobody destroys anything - physical destruction under a
  retention or legal policy would be a separate owner decision. The per-dispute document
  ceiling counts live rows, so a deletion frees a slot.
- **Support staff can open a dispute and attach evidence, and can make no money decision.**
  Accept, reject and any other settling action stay with the administrator and the dispute's
  trader. The guard that used to assert this role has no write route at all was narrowed rather
  than deleted: it now lists the three routes it may have and fails on a fourth.
- **A trader reaching for another party's identifier is told the object does not exist.** Reads
  of disputes by requisite, device or deal are scoped to what the caller owns, and an
  unreachable object answers exactly as a nonexistent one does - the same body, on every
  method including writes - so the answer cannot be used to discover that someone else's
  dispute exists. Closing that hole was an owner decision, taken deliberately even though a
  comparable legacy gap elsewhere was carried forward verbatim: an earlier decision to carry a
  defect does not oblige new code to reproduce it.
- **The sandbox cannot touch production.** Test credentials reach sandbox data only, with no
  fallback to a production lookup, so they can neither create a real dispute nor trigger a real
  merchant notification or auto-block. Today the sandbox holds no deal eligible for a dispute,
  so the route answers "deal not found" - that is the current state of the sandbox, not the
  endpoint's permanent meaning.
- **Two deliberate departures from the old merchant wire are recorded rather than hidden.** A
  deal in the error state now answers a client-side rejection instead of the server error the
  old code produced through a defect in how it matched its own message; and the error envelope
  keeps the three canonical keys instead of a fourth that carried internal validator text. Both
  were owner decisions. Everything else on that wire, including the dispute notification
  payload, is byte-identical to the old one.
- **Current/next task:** `015-cutover-dev-stand` is the next phase in the dependency order,
  with `018-sweep-trx-float` already complete. By owner decision the next phase is never
  started automatically.
- **That cutover gap is now closed, and closing it revealed a dead control.** The deal
  constructor's trader dropdown used to call the payout queue's filter route, which no v2
  phase owns; it now reads the constructor's own list of traders, the symmetric twin of the
  merchant list it already had. No payout machinery was ported for it, no role gained data or
  a capability, and no money path was touched. What the investigation turned up is that the
  filter the dropdown feeds has never filtered anything: the selection is not sent to the
  server and does not narrow the result list, and in the old code the parameter the frontend
  did send was silently dropped, because the request struct had no field for it. Making the
  filter real, or removing the control, changes visible behaviour and is an open owner
  decision; the list itself works either way.
- **A sweep of the web client found no unrecorded call that the cutover will break.** Every
  frontend call that loses its backend at the switch is already named in a phase
  specification: the deferred dashboards and their export, the merchant dashboard statistics,
  the log and message surfaces, the dropped requisite-monitoring routes, insurance, and the
  batch processing action. A second group of calls is already dead against the current backend
  and so is unaffected by the switch - system configuration, notifications, currency
  conversion, and a few device and SMS-device operations that were never served for the role
  the client derives. One surface still needs an owner decision before the switch: the
  insurance administration pages are money-adjacent, are served today, and the specifications
  name them precisely as belonging to no v2 phase.
- **The cutover is wider than the web client, and one known break lives outside it.** Android
  self-update fetches its package from the platform and stops working the moment the stand is
  switched; that obligation is already assigned to the cutover phase itself. Read the sweep as
  covering the web client only.
- **Next action:** none in flight.

### Phase boundaries deliberately held

Structure exists in the baseline where behavior does not. A table being present
is not evidence that its behavior is implemented.

- **Nothing in the v2 core has been exercised against real infrastructure.** The deposit, sweep
  and withdrawal paths are complete and covered by tests, but assembly, broadcast and the owner's
  approval have never run against a live chain or a real bot, and the panel has never spoken to a
  running v2 stand. All of that is first proven at the cutover phase.
- **Relocating custody to cold storage is not built and is not a treasury withdrawal.** It is a
  different operation with a different accounting model; folding it into the treasury path would
  hide a second kind of money movement inside an existing one.
- **A treasury withdrawal draws on the platform's revenue account and is bounded by it.** An
  administrator cannot reach trader or merchant custodial balances through that path, and the
  administrative balance view shows the same account for the same reason.

- The ledger is now complete in both directions. Deal creation places holds,
  confirmation settles, a confirmed deposit credits a balance, and a confirmed
  withdrawal debits one to the outside. Every movement of money in the v2 core now has
  an implemented path.
- The insurance forfeiture and the manual adjustment are primitives with no
  caller and no endpoint. Do not wire either to anything without the owner
  decision the phase-008 specification requires first - forfeiture moves a
  trader's collateral to the platform and no phase owns the rule for when that
  is allowed.
- The device-push and SMS notification pipelines now confirm deals; phase 006 had
  deliberately stopped them at the first access to the deal table.
- Deal reads exist for every panel role, and the transaction and dashboard pages call them.
  The dispute mechanism is now ported too, and the two pieces the deal-flow phase shipped
  without callers - the money-moving deal acceptance path and the pair of dispute-driven
  requisite block and unblock operations - have them at last, without being rebuilt. A deal
  read reports whether a dispute is open on that deal, derived from the dispute table rather
  than stored on the deal, so the flag cannot go stale the way the legacy one did. Every panel
  route that touches money now exists; what has never run is any of it against real
  infrastructure.
- Requisite selection, limit spending, expired counters, auto-blocking and
  counterparty binding all exist from phase 009, with their two tables. The
  administrative routes for trader priorities now exist as well, so the priority branch of
  selection is reachable rather than merely implemented.
- The legacy financial-statistics triggers, wallet and queue structures,
  batch/MultiSend, per-deal on-chain settlement and the withdrawal model are not
  present and are not scheduled before their own phases.
- An inbound event applier performs whatever business write and ledger effect the event
  carries, together with the applied marker, in one transaction, so nothing can be
  applied twice or left half-done.
- The Web-to-Crypto transport now carries every command and event the wire defines. The
  outbox has three producers: the deposit-address command, the withdrawal command and the
  cancellation command; a sweep remains absent from it, because a sweep is decided by the
  custody side and is not commanded across the wire at all. The event journal applies all
  five event types, withdrawal status and cancel rejection included. The swept event
  still records the fact and deliberately posts nothing to the ledger, since the money
  was already credited at confirmation. No event type is stored-and-parked any more.
- The custody service derives deposit addresses, watches the chain, tracks deposit
  confirmations, signs and broadcasts on its own for the two internal custody operations,
  and now signs payouts as well - but only after an explicit owner approval delivered
  through the Telegram bot, which is the only trigger that exists for it. Both the
  deposit-specific and the daily cross-system reconciliation run on a schedule, and the
  reconciliation's in-flight term is now populated rather than fixed at zero.
- The two limitations that phase carried forward are closed, and closing them was the
  substance of an owner decision. A pre-signature refusal now always reaches a landing
  state instead of repeating without bound or falling silent, and a crash between
  approval and signature is re-driven - with the policy applied again before anything is
  signed. One accepted residual remains: while the policy still passes, a node that
  cannot assemble a transaction is retried without a repeat budget, which costs journal
  rows and node calls but cannot pay twice, cannot pay under a changed policy, and keeps
  the hold. Bounding those repeats would be a new rule about when money stops being held
  and was left to a future owner decision.
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
  review and CI; the read surface became `017-deal-read-and-disputes`, and the dispute surface was split off
  again during that phase and is now `019-dispute-mechanism`.
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
- Task 018 completed and remotely verified: the custody side sweeps and funds itself.
  Six owner decisions are recorded in its specification. Three shaped the phase before a
  line was written - that empirical cost evidence had to be gathered before any threshold
  was accepted, that the sweep-side classes get their own explicit safety budgets rather
  than an exemption from the payout ones, and the exact structural restriction on where
  each class may send. Two facts from it a cutover session will want: the chain resource
  actually consumed by each outbound transaction is recorded alongside the fee, so real
  operation can be compared against the baseline; and every threshold there is
  configurable and recorded as a phase baseline to revisit, never as a business constant.
  The independent reviews earned their place twice over: the
  specification review caught, before any code existed, that an exhausted global budget
  would have suspended every address at once instead of the one at fault and that a
  transient failure during assembly could permanently and silently disable an address,
  and the code review then caught a real defect in the implementation - each top-up in a
  batch checked the custody reserve floor against the balance read at the start of the
  pass, so a batch could individually pass and collectively breach a floor the owner had
  declared inviolable.
- Task 013 completed and remotely verified: the withdrawal path exists end to end, and
  with it the last money path of the v2 core. Seven owner decisions are recorded in its
  specification, covering who may approve, what a treasury withdrawal is allowed to
  move, how long consent waits before it expires, whether a payout may be reassembled
  under a policy that has changed, and whether a completed withdrawal shows the
  transaction that paid it. The independent reviews were the phase's most valuable
  activity by a wide margin. Before any code existed, the specification review caught an
  approval rule that would have required more approvers than the legacy behavior it was
  meant to carry over, an owner-facing shortfall figure that was arithmetically too small
  so that topping up by the amount shown would still have failed, and a guarantee the
  specification asserted but could not deliver, since the state it reasoned from does not
  imply what it claimed - and the acceptance test for it could not have failed, because
  the test double always supplied the missing value. After implementation, the code review
  caught a payout that could be signed under a stale authorisation long after the owner
  pressed, which became an owner decision and then a design change; and, in the fix for
  it, an accounting error that counted a request against its own daily limit twice, which
  no test in the suite could see because every fixture used amounts far below the cap.
- Task 014 completed and remotely verified: the v2 core became operable by a person. Eleven owner
  decisions are recorded in its specification, covering whether an
  administrator may start a treasury withdrawal from the panel at all, what an administrator's
  balance means once custody is omnibus, whether support keeps its view of a subject's funds,
  whether the public payment widget shares the panel's HTTP client, how far the client
  consolidation reaches, and whether a page whose calls have never been authenticated should start
  working as a side effect of that consolidation. The specification reviews earned their place
  before any code existed: four rounds caught an acceptance criterion resting on a lint command
  that is broken in the baseline and would have blocked the commit regardless of code quality; a
  structural guard that failed on correct code, making the phase's own gate unreachable; a single
  line of client configuration that satisfied every guard rule while sending the operator's token
  to the public payment page; a migration that would have replaced a merchant's own API credentials
  with the panel session token; and a rationale that licensed dropping a merchant's request
  signature. After implementation the code reviews found a filter that returned an empty approval
  list while requests were waiting, a failed request that rendered as "nothing to approve", and a
  translation key collision that printed a diagnostic sentence where the word "Status" belonged -
  none of which any build, type check, guard or test in this project can see.
- Task 017 completed and remotely verified: the panel can read what the platform did. Ten owner
  decisions are recorded in its specification - seven substantive, the other three being the
  annulment and renewal of the commit authorization and the shape of the final fix round -
  among them the split of the dispute mechanism
  into its own phase, which dashboards may exist at all in v2, what the requisite turnover and
  profit figures mean, and - the one raised by a reviewer after the specification was already
  accepted - whether unifying the deal response shape may hand every role the union of what any
  role sees. The owner ruled it may not, which turned a structural read phase back into what it
  was meant to be instead of letting it widen who sees whose money. The code review then caught
  a defect no test could: the phase renamed four contragent fields into a nested object and
  migrated only one of the three cards, so an administrator's contragent button would have
  called copy where the operator meant transfer. It survived because the client method was
  typed as an untyped response; giving it a real type surfaced four further reads of fields
  this phase had removed.

- Task 019 completed and remotely verified: the dispute mechanism exists end to end, and
  accepting a dispute settles it through the same path an ordinary confirmation uses. Nine
  owner decisions are recorded in its specification - eight substantive, the ninth being the
  commit authorization - the load-bearing one being that evidence files live in
  the database rather than on a service's filesystem. The independent reviews earned their
  place repeatedly. Before any code existed they caught that the prohibition meant to keep
  uploads out of the filesystem did not name the one standard-library function that actually
  spills them, that reading a large upload inside an open transaction had no time bound and
  could hold a database connection indefinitely, and that the order of parts in a multipart
  upload had become significant while remaining unspecified. After implementation they caught
  a merchant-visible error body that differed between the production and sandbox routes, and -
  on the last pass before the commit, after three earlier rounds had missed it - a navigation
  target that compiled and type-checked but sent the user to the home page instead of the deal
  card. That last one is the standing lesson: the criteria this project verifies by code review
  alone are exactly where a defect survives longest, and a fix should make the mistake
  unexpressible rather than merely absent.

## Known traps

- **The order of parts in a multipart dispute upload is load-bearing.** `deal_id` and `reason`
  must be sent before the first file part; a non-file part after it is refused with a 400. That
  is a deliberate narrowing, forced by streaming the parts straight into the database instead of
  buffering them, and the single client appends the fields in that order today. Reorder those
  lines and the surface breaks with a 400 that no build, type check, guard or test will see.
- **A deal may have at most one active dispute, and this is now a database constraint, not
  only a check in the creation path.** It already forced a test fixture from an earlier phase to
  spread its disputes across several deals; any future fixture that stacks two active disputes
  on one deal will fail for a reason the code does not explain. The rule the owner set: never
  drop a database invariant to preserve a fixture representing a state the invariant makes
  impossible.
- **The body of a multipart upload is read inside an open database transaction**, bounded only
  by a server-side deadline that the implementation makes injectable for tests. Remove or
  lengthen that deadline and a slowly-sent request holds a pooled connection for as long as it
  likes; the pool is small, so a handful of them stall every money path in the service.
- **The scan that keeps uploads off the filesystem runs at review time, not in CI.** The
  structural guards this project runs automatically do not include it, and the automated
  temp-file test needs a database and therefore skips without one. Several other criteria of
  the dispute phase - the single API client, the untouched earlier tests, the client-side
  part order - are likewise verified by a reviewer running a pattern, not by a pipeline. Treat them as
  standing obligations of whoever reviews next, not as things already guaranteed.
- **Evidence is never destroyed by an ordinary delete, and the sandbox's current answer is not
  its contract.** Deleting a dispute document hides it everywhere, including from the
  administrator judging the dispute, while the bytes and the history stay; physical destruction
  would be a separate owner decision. The sandbox dispute route today answers "deal not found"
  only because the sandbox holds no deal eligible for a dispute - that is the state of the
  sandbox, not the meaning of the endpoint.
- On a surface whose fields come from optional joins, `null` already means "the joined row is
  missing". A field withheld from a role must therefore be **absent** from the response, not
  null: nulling it makes "withheld" and "no such row" indistinguishable, and no test can then
  prove the withholding still works.
- A daily-window sum plus the amount being evaluated is only correct while that request
  is not already inside the sum itself. Re-checking a request that has already been
  approved counts it twice; simply dropping the added amount is wrong in the other
  direction, because an approval that has aged out of the window would then be counted
  zero times. Exclude the request being evaluated, and keep the addition.
- A consent lifetime must key on "the owner has not decided anything yet", never on a
  list of states. A request can return to the owner's queue after a decision, and a
  state-list rule silently starts expiring it a second time.
- A request approved for send that carries no stored transaction id provably has no
  signature behind it, which is the only reason it can be safely handed back to the
  owner. That proof rests entirely on the transaction id being written by the signing
  transition and by nothing else - preserve that property or the safe exit disappears.
- The custody daemon refuses to start without its owner-approval channel configured,
  the same class as the mTLS material. Supplying it is a cutover precondition, not a
  runtime nicety.
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
- Deposit addresses, ledger balances and the custody-status view now have HTTP routes and a
  panel that consumes them; that boundary is closed. What remains is that nothing in the panel
  has been exercised against a running stand, because the stand still serves the legacy backend
  until the cutover phase.
- A test that renames a table away to simulate a failure is safe only because every
  fixture runs on its own ephemeral database that is dropped afterwards. Do not copy that
  pattern into a suite that shares one database.
- **Never assert an ordering between a timestamp your service wrote and one a database
  trigger wrote.** They come from different clocks. A test in this phase compared an
  operation's service-clock creation time against a trigger-written update time; it
  passed for hours and then began failing deterministically, with nothing changed on
  disk, the moment real time crossed the fixture's pinned epoch. Nothing was wrong with
  the product. Two lessons: derive age only from timestamps your own code supplies, and
  treat a suite that passes now as unproven against a clock - the failure had been
  waiting to surface in CI at an arbitrary hour. Pinning a fixture epoch safely in the
  past, rather than to today, makes the whole hazard unconstructible.
- A per-item safety check that reads a shared balance once per batch and never subtracts
  what the batch has already committed will let each item pass while the batch as a whole
  breaks the limit. Accumulate within the pass, from committed state, rather than
  re-reading an external source per item.
- Reserve floors are directional. A check that constrains an operation draining a balance
  is arithmetically vacuous for one that fills it, and writing it as if it applied to both
  hides which limit is actually doing the work. State per operation class which floors
  bind and which are vacuous.
- A test double and the code it feeds can share the same wrong assumption and agree
  forever. Where an external format matters, pin it with a real captured sample and prove
  the tests are not vacuous by deliberately breaking the mapping and watching them fail;
  encoder-and-decoder symmetry alone proves nothing about absolute correctness.

- Phase 014's traps are about defects that no automated check in this project can see. A
  translation key that is a **string in one language and an object in another** resolves, in the
  language where it is an object, to a diagnostic sentence rendered straight into the UI - and the
  usual "default value" argument does not save it, because that fallback applies only when the
  resolved value is empty or missing, and an object is neither. Build, type check and structural guards are all
  blind to it. The only way to know is to resolve every key through the real translation library
  against the real files; do that whenever a phase touches a surface's captions.
- A structural guard is only worth what it detects, and the gap is never where you look. A guard
  written to keep authenticated traffic on one HTTP client missed the callable form of the client,
  a lower-case header name, the browser's older request object, a second client constructed in an
  allowed file, and a re-introduced call that took the session credential
  directly instead of obtaining it from the canonical client. Worse, its first version
  counted violations inside a shell pipeline subshell and reported success while printing them.
  Both classes were caught only because the acceptance criterion required **demonstrating the
  guard failing on an injected violation**, not merely writing it. Require that demonstration for
  every guard.
- The mirror-image hazards on an approval screen are equally dangerous and only one is obvious. A
  filter that silently matches nothing shows "no requests" while requests wait; a filter that is
  not applied shows every subject while the control reads as filtered. Both end with an operator
  acting on a list they misread. Derive the "is this filter actually applied" condition and the
  "what do we send" condition from one predicate, and say plainly on screen when a filter is
  incomplete.
- A failed request must never render as an empty result on a surface where "empty" means "nothing
  to do". A query that returns no data on error, plus a default of "empty list", plus no error
  branch, turns every network blip into a false all-clear. Where rows were already loaded, keep
  them under an explicit staleness warning rather than blanking the screen - that is safe here only
  because the server re-reads the request under lock and refuses a stale action.
- Comparing a type-checker's error count between two checkouts produces false positives: some
  messages embed absolute paths, which differ per checkout. Normalise the paths and compare as a
  multiset, or a clean run will look like a regression.
- An acceptance criterion pointed at a path that does not exist passes silently forever. One in
  this phase asserted that database migrations were untouched, against a directory path that does not
  exist; the command returned success whatever the migrations did. A criterion that cannot fail
  is not a criterion.
- Consolidating clients changes behavior beyond the transport. Call sites that had no session
  handling inherit it, and any endpoint that answers "unauthorized" for a **wrong credential**
  rather than a dead session would then log the operator out on a mistyped password. No such
  endpoint is reachable through the shared client today; check that again before routing a
  credential-checking call through it.
- A public payment page is not the operator panel. Folding it into the panel's authenticated
  client attaches an operator token to a request a paying customer's browser makes, and gives that
  customer the panel's session-expiry redirect mid-payment. Express that boundary as a separate
  component, not as a flag on a shared one.

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
