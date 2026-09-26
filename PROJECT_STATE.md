# CashCode project state

Last updated: 2026-09-26

This is a compact restart checkpoint, not a diary or full specification.
Accepted ADRs and task specifications in the private project repository remain
the canonical engineering source of truth.

## Project and direction

CashCode is a payment and trading platform with Go services, PostgreSQL, a
React/TypeScript web client, and a Kotlin/Compose Android device application.

CashCode v2 is a **selective rewrite**, not a full rewrite and not an in-place
refactor. The legacy backend served as a frozen reference while v2 was being
ported; after v2 acceptance it was physically removed from the repository
(phase `016-legacy-removal`). The only code is `services/web`, `services/crypto`
and the minimal wire-only `services/contract` module, plus the web client and
the Android application. Old behavior is read from git history at the parent
of the removal commit and from the recorded legacy evidence under
`docs/platform/`; the legacy module path must never be imported again, and a
structural guard keeps it out.

Kept with cleanup: clients and non-financial product behavior, including auth,
users and permissions, requisites and devices, disputes, merchant API and
sandbox contracts, IPN, widget, fees, and exchange rates - with the merchant
API and IPN authentication envelopes since replaced by the pre-production
hardening task, while their business payloads stayed as they were. Rewritten: backend
wiring and transaction boundaries, the financial data model, deal lifecycle,
ledger and holds, deposits, withdrawals, custody, and the financial UI. The
legacy wallet, queue, batch, MultiSend and fragmented withdrawal mechanisms
were removed together with the legacy tree after v2 acceptance.

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
- **Current stage: the production-readiness package is complete up to a
  mandatory stop - the owner's architecture review.** Production has NOT been
  deployed, no infrastructure has been rented, and nothing proceeds until the
  owner has gone through a plain-language architecture review in the private
  repository and explicitly confirmed that they understand and accept the
  resulting architecture; silence is not consent. The review covers the
  components, what changed against the legacy system, each machine, where
  money lives in accounting and where real keys live, deposits, deals, the
  ledger, withdrawals and the owner's approval step, failures and full
  reboot, backups, who controls what, the accepted and not-yet-accepted
  residual risks, what will be rented, and what the external validation gate
  must still check.
- **Owner decisions recorded this cycle (final for this stage).** The SMS-box
  channel is not used in the first production release (kept in code, off on
  the server). The secret store is unsealed manually by a 2-of-3 threshold,
  and all three shares - like every other piece of infrastructure and custody
  control, including release signing - stay with the owner alone; there is
  no escrow and no third party in the chain of control, and the customer
  receives application roles only. The custody service's access to the
  secret store keeps the model already proven on the stand (periodic renewal
  by a scheduled job) with an alert on failure. Nothing is compiled on
  production hosts: releases are built elsewhere, pass CI and review, and
  carry checksums and provenance. The phone app keeps one stable production
  name compiled in. Money limits stay as they are and will be shown to the
  customer in one table before launch.
- **Done this cycle, each with independent specification and code review:**
  `059` - small production-readiness fixes (TLS in the proxy template, the
  log relay without root, phone release builds refuse plain HTTP, the parser
  corpus harness ready for real samples); `060` - the SMS-box channel is
  switched off on the server fail-closed, so it cannot be activated by
  bypassing the web client, and production refuses to start with it exposed;
  the merchant sandbox now returns a working widget address; `061` -
  second-factor secrets are encrypted and session refresh tokens are stored
  only as hashes in the web database; `062` - a release bundle script
  (reproducible service builds, manifest with checksums, CI evidence, offline phone
  signing by the owner, verification on the target host) and templates for
  renewing the custody service's access with an alert on failure. `v2` CI
  **success** for the code changes (runs `36224918616` at `2e3fb8e`,
  `36226909151` at `04eb6e7`); the last commit `2e67164` is outside the CI
  path filter. One check was only partly possible here: building the phone
  release through the script ran out of memory on the development host and is
  to be repeated on a larger build machine. None of this was deployed to the
  stand in this cycle; it changes nothing there until the next web release.
- **Previously completed and deployed:** `056-rub-only-bank-events`, parts A **and B**.
  By owner decision the platform serves only rouble deals and payments. After
  the owner answered the four blocking questions, the rule is now complete: the
  parsers were tightened so that a payment auto-confirms only when it
  unambiguously indicates roubles, and any payment that names or implies another
  currency, an unrecognised currency, or comes from an older phone build is sent
  to manual review instead of confirming. This holds for both the phone channel
  and the SMS-box channel. Proven by unit and integration
  tests and by a mechanical comparison of the old and new parsers over every
  test message - engineered messages only - and passed independent
  specification and code review with no blocking findings. An accepted,
  to-be-measured cost of the chosen rule is that a minority of genuine rouble
  payments may be sent to review by mistake; the rate will be measured on the
  stand before the external validation gate. **Deployed to the development
  stand:** migrations applied, the new server released, and the new phone build
  installed first (phone build before server, as required), with the device
  binding preserved; the updated phone re-connected to the new server, its
  event queue empty with no losses, and money state unchanged. What is NOT part
  of this task and remains open is validating the rule against **real** bank
  messages - that is the external gate below, because a real message cannot be
  injected on the phone. Task closed. `v2` and `frontend` CI **success**.
- **Custody host (Crypto VM) - both problems fixed and verified.** After the
  owner granted the privileged access, two confirmed defects were resolved.
  (1) Deposit discovery had stopped: a connection-handling defect in the custody
  service caused scans to stall rather than complete, even though the network and
  chain node were healthy. It is fixed; on the stand deposit scans now run with
  zero errors over many cycles. (2) A credential-lifetime issue on the custody
  service that would eventually have blocked it from restarting is fixed:
  automatic renewal is in place and verified, and the service restarts cleanly.
  `v2` CI green.
- **Divergence-alert false positive (`057`) - fixed (owner-approved).** The
  owner approved the change; the custody-reconciliation channel no longer raises
  a false critical alert in that edge case, and the real-divergence guarantee is
  preserved and tested. Reviewed, `v2` CI green, deployed to the stand.
- **Also in an earlier cycle - SMS-box channel:** established read-only that the
  channel is switched off on the stand and nothing depends on it, and that
  whether the third-party sender's signature is compatible with the server
  cannot be settled from the repository - it needs the sender's specification
  or a captured real request. No change was made; its production fate stays an
  owner decision.
- **Also in an earlier cycle:** a question about when a device's pending
  requisites become active was closed by owner decision with the existing
  behaviour kept.
- **Last completed and deployed:** `054-bank-fee-update-lost-row` - an
  administrator's edit of a merchant's bank-specific fee that another
  administrator deletes at the same moment now answers "not found" instead of
  reporting success; nothing is written. Fee rules, roles, the API contract
  and existing deals are unchanged, no money is involved. Deployed to the
  development stand (Web only) on 2026-09-25 without a schema change; the
  phone, the custody exchange, the money state and the fee tables were
  unchanged. Proven by integration tests that fail without the change and by
  CI - the race itself was not exercised live, which would have meant
  changing tariffs. Head `ad0eac4`; `v2` run `35989725410` (at `66b2853`),
  **success**.
- **Previously completed:** `053-merchant-bank-fee-concurrency` — concurrent edits
  of a merchant's bank-specific fee ranges are serialised, so two overlapping
  ranges of one merchant can no longer both be accepted. A bounded review of
  the remaining fee writes tied to the same non-overlap rule found no other
  unprotected one. Fee rules, fee calculation, tier selection, roles, the API
  and existing deals are unchanged. Deployed to the development stand (Web
  only) on 2026-09-24 without a schema change; the phone, the custody
  exchange, the money state and the fee tables were unchanged, and the stand's
  fee tables still have no overlap. The serialisation was proven by
  integration tests, including checks that fail when it is removed, and by
  CI - not exercised live, which would have meant changing tariffs. Head
  `77ccb7f`; `v2` run `35983911940`, **success**.
- **Previously completed:** `052-default-fee-grid-concurrency` — concurrent edits
  of the default fee schedule are serialised, so two overlapping ranges can no
  longer both be accepted; the non-overlap rule now holds under concurrency as
  it did for one edit at a time. Fee rules, tier selection, roles, the API and
  existing deals are unchanged. Deployed to the development stand (Web only) on
  2026-09-24 without a schema change; the phone, the custody exchange, the
  money state and the fee tables were unchanged, and the stand's default
  schedule still has no overlap. The serialisation was proven by integration
  tests, including checks that fail when it is removed, and by CI - not
  exercised live, which would have meant changing tariffs. Head `b5ea9af`;
  `v2` run `35977039152`, **success**.
- **Previously completed:** `051-trader-fee-replacement-atomicity` — saving a
  trader's individual fee schedule is now one unit: either the whole new
  schedule applies or the previous one stays exactly as it was, and resetting a
  trader to the default schedule is one unit too. Concurrent changes to the
  fee schedule of one trader or one merchant are serialised and no longer mix.
  Fee rules, range checks, roles and the API are unchanged, and deals already
  created are not affected, because a deal's fee is fixed when it is created.
  Deployed to the development stand (Web only) on 2026-09-24 without a schema
  change; the phone, the custody exchange, the money state and the fee tables
  were unchanged. Atomicity and serialisation were proven by integration tests
  and CI, not exercised live - that would have meant changing tariffs. Head
  `93c16a2`; `v2` run `35966255496`, **success**.
- **Previously completed:** `050-frontend-lint-restoration` — the web client's linter
  runs again and checks the code. It had been aborting on its first file
  because of an incompatible pair of pinned tooling versions; one of them was
  moved within its declared range, the production build stayed byte-identical,
  and the checker's configuration was not weakened. It immediately found two
  kinds of real defect, both fixed: one silently disabled a client-side
  consistency check on fee settings, the other was a scoping error. The
  remaining historical findings - 464 errors - are **recorded as a baseline, by
  owner decision, not fixed**: every rule stays enabled; a new rule violation in
  a file, or growth in a file's count for a suppressed rule, fails the run; the
  baseline is counted per file and rule, not per finding, and can only shrink. It is technical debt,
  not evidence that the existing code is clean; 27 warnings stay visible. A
  dedicated CI workflow now installs, lints and builds the web client on every
  change to it. The web client's type-check errors (385) remain a separate
  baseline; the build does not type-check. Head `ec2f222`; `frontend` run
  `35961620941`, **success**.
- **Previously completed:** `049-device-requisite-authorization-gaps` — the
  authorisation and validation gaps inherited from the legacy platform on some
  device and requisite routes, carried on purpose since the porting phase, are
  closed under four owner decisions: a team lead reads only what belongs to
  their own referrals; a trader's write to another trader's object answers
  exactly as a write to a missing one, so it discloses neither existence nor
  content; the field rules the legacy requests declared but never enforced are
  enforced, without inventing new ones and without locking edits of existing
  records; and the administrator address restriction now applies on every
  request of the support administrator's surface, not only at sign-in.
  Deployed to the development stand on 2026-09-24 without a schema change; the
  phone, the custody exchange and the money state were unchanged. The role
  behaviour was proven by integration tests and CI, not exercised live - the
  stand has no users of the affected roles, and no account was created or
  altered for it. Criteria: all passed. Head `04bae80`, `v2` run
  `35956932107` success.
- **Implemented, not closed:** `048-production-config-fail-closed` — a production
  or staging configuration now refuses to start when mandatory infrastructure
  or settings are missing, instead of silently running in an incomplete or
  development mode, and it reports the loss of a required dependency after
  start. Two inherited degradations of authentication controls were removed
  for production by owner decision; the development stand keeps the inherited
  behaviour on purpose. Deployed to the development stand (Web only) on
  2026-09-24. The task is **not closed**: **one criterion stays open** - the behaviour against real
  production infrastructure has not been checked, and tests on isolated
  processes are not that check. The custody side was not updated on the stand;
  its compatibility with the new build is not yet confirmed and is checked
  before its next update. Head `f31a2fc`, `v2` run `35950482699` success.
- **Previously completed:** `047-device-key-lifecycle` — a key a device has
  retired is not accepted again by a fresh enrollment code, a transient
  key-store or network failure never destroys a key, bank events left from a
  previous binding are never sent under a new one without the person's explicit
  confirmation, and a stuck unbound phone offers a confirmed key reset.
  Deployed to the development stand on 2026-09-24; criteria 17 of 17. Head
  `bab2986`, `v2` run `35938820508` success.
- **Previously completed:** `046-device-offline-recovery` — after an automatic
  offline switch-off the server now remembers what it switched off, and when
  the phone is back the panel offers to return exactly that, one click per
  step through the ordinary routes; nothing returns by itself, and details
  that were off or blocked are never offered. The stand episode that raised it
  was a server-side failure to record heartbeats whose cause is **not
  established**. Deployed to the development stand and run live on
  2026-09-24; criteria 21 of 21, one accepted by the owner with a named
  limitation (the administrator-panel view was not checked visually). Head
  `a13f3d7`, `v2` run `35925843114` success.
- **Previously completed:** `045-bank-source-and-name-catalogue` — the three
  directories that name a bank event's source are now checked against the
  catalogue the payment details take their name from, and a path by which a
  sender could name a bank other than the one that sent the money is closed in
  the repository; one restriction lives in the handset only and is not a server
  guarantee. Some legitimate sender spellings are no longer recognised
  automatically - a recorded loss in the safe direction, open until the real
  corpus. It does not establish that any sender or application belongs to
  the bank it is listed under; five criteria wait for the real corpus (gate item
  **G-4**). Head `ffbddf4`; `v2` run `35776873900`, **success**; deployed to the
  development stand on 2026-09-22.
- **Previously completed:** `044-blocking-notice-classification` — bank
  notifications that a parser wrongly read as a block of the payment details. A
  notification now counts as a block only when it states unambiguously that a
  card or an account is blocked or restricted now; other notices that merely
  mention a block no longer block, and explicitly worded blocks keep blocking.
  The same rule applies in both parsers. 8 of 8 criteria met; deployed to the
  development stand on 2026-09-22 with the handset updated in place. **Not
  proof that false blocks are gone for real bank notifications**: the corpus
  holds no real bank message, and its residual risks stay open until **G-4**.
  Head `e204fdd`, `v2` run `35711045969`, **success**.
- **Previously completed:** `043-deal-attribution-ambiguity` — the part of the
  task proving which deal a bank payment belongs to that needed no real bank
  messages: one open deal of a given amount per handset, enforced by the
  database; no choosing between candidates, so an event that fits several open
  deals confirms none; and an amount no longer read as a card number. This is
  not proof that a payment belongs to a deal; residual risks stay open and need
  the real bank messages. 13 of 13 criteria met. Deployed to the development
  stand on 2026-09-22; the handset application was not changed. Head `e24fee0`,
  `v2` run `35701164674`, **success**.
- **Previously completed:** `042-bank-message-corpus` — the technical part of
  the parser-corpus task: one corpus format in which every sample declares
  whether it is real, engineered or of unknown origin, and test harnesses that
  run every sample through the actual parsers of both channels, keeping what
  the code answers apart from what the message means. **Real samples: 0.** A
  green run proves the code still answers what is recorded, not how a real bank
  writes. Collecting the real anonymised corpus is item **G-4** of the external
  validation gate. Head `e71c1c1`, `v2` run `35679026299`, **success**.
- **Previously completed:** `041-bank-event-durable-delivery` — an observed bank
  event now survives on the handset in an encrypted durable queue until the
  server has decided it, or until the binding ends, which discards it with the
  loss counted; the server records each event under its own
  identity before deciding, in the same transaction as any money it moves, so
  one delivered event settles at most once. These guarantees cover the handset
  channel only; the separate SMS-box channel was deliberately left unchanged,
  and money rules were otherwise unchanged. Automatic confirmation is refused in
  the cases that task recognised as unsafe; that narrows the matching risk and
  does not remove it. Live-validated on the development stand and the physical
  handset, 64 of 64. Head `b5b1c5c`, `v2` run `35646229651`, **success**.
- **Previously completed:** `040-requisite-block-bypass` — a blocked requisite
  could be returned to deal matching without the unblocking procedure; it was a
  class rather than one route, and is now closed by a single invariant evaluated
  where the change is written. Money behaviour was held unchanged. A structural
  check guards the rule's decision point; it raises the cost of a future bypass
  and is recorded as a fence, not a proof. Head `8b6cc7a`, `v2` run
  `35514360889`, **success**.
- **Previously completed:** `039-device-diagnostics-and-operator-health` — an
  operator can read the state of a physical handset from the panel instead of
  picking the phone up. The device sends a periodic structured snapshot of
  itself and the server, not the phone, turns it into one verdict with
  enumerated reasons, each carrying what the operator should do about it. A
  phone that has quietly stopped working is now distinguishable from one that
  simply had a quiet day. The snapshot carries no message content, no personal
  or payment data and no secret, and that is structural rather than filtered.
  One verdict names the common case rather than a fault: a handset can be
  entirely healthy and still be out of deal matching, because binding proves
  identity while activation is a separate human act; the panel offers that
  activation after a binding and never performs it on its own. Verified on the
  physical handset. One criterion is deliberately left open — how power
  management behaves on handsets from other manufacturers — and it joins the
  external validation gate below as its third item. Head `f395950`, closeout
  `d2201a6`; `v2` run `35431798216`, **success**.
- **Previously completed:** `037-android-preproduction-hardening` — the second of
  the pre-production tasks. The Android application no longer carries a shared
  enrollment secret: every phone now has an identity of its own that the server
  can verify and revoke individually, so compromising one handset no longer
  says anything about any other. Binding a phone is an operator-issued one-time
  code that the phone must answer with proof that it holds the private half of
  the identity it presents; the code alone binds nothing. Revocation is a
  durable fact kept separate from whether a device is merely switched off: no
  automatic path sets or lifts it, and a revoked phone returns only through a
  new binding. Repeated and replayed device requests are refused, the update
  path refuses a release it cannot verify, cleartext traffic is confined to the
  debug build type, and notification content no longer reaches logs or the
  diagnostic channel.
  The whole device lifecycle was exercised on a physical Android handset:
  binding, an application update with the binding surviving it, revocation with
  the phone erasing its own key and going silent, re-binding, and a real backup
  run confirming the backup does not carry the credential store. The other half
  of that check - restoring onto a different handset - is in the gate below and
  has not been run.
  Three follow-up items are recorded in the private repository. They concern
  device key lifecycle and enrollment recovery, none of them blocks the closure
  of this task; the two that needed an owner decision were settled and closed
  by `047`.
- **Previously completed:** `036-merchant-api-hardening`. The
  merchant public API now has a single authentication scheme, the one the
  platform will launch with, replacing the one carried over from the legacy
  platform. Two behaviours are worth knowing before integrating against it: a
  captured request re-sent unchanged is refused, and every authentication
  failure answers one identical refusal, so the error text does not tell one
  cause from another while debugging an integration. The old scheme is not kept as a compatibility mode: the owner
  confirmed there is no integrated merchant estate that has to be preserved
  byte-for-byte, and chose a clean contract over carrying the old one
  indefinitely. An integration is built against the contract document, which
  the task added to the private repository.
- **What else that task closed.** A merchant credential can now be rotated and
  revoked by an operator — rotation with a transition window, so changing a key
  costs the merchant no downtime, and revocation taking effect at once. The
  sandbox now has a credential of its own, issued to the merchant in the
  panel, and sandbox calls are signed with it. Outgoing notifications are held
  rather than discarded while a merchant has no usable credential, and they
  carry an identifier stable across retries, so a merchant can tell a
  redelivery from a new event. The platform refuses to deliver a notification
  to an address inside its own network, on every hop. The surface has its own
  request-size, idle and rate limits. A blocked merchant is now refused on
  every merchant route; before, it was turned away only on deal creation, and
  softly. The merchant settings routes are restricted to the merchant role, and
  the shop identifier is fixed once a credential has been issued.
- **One accepted architecture decision changed, deliberately and in writing.**
  The earlier decision froze the merchant notification contract. Changing its
  authentication envelope was therefore an owner decision, not an engineering
  one; the owner took it before the first production deployment and required
  that the decision record be updated rather than left contradicting the code.
  The notification body and its business meaning are unchanged — only the
  envelope that proves who sent it and which delivery it is.
- **A test guard, and what it uncovered.** The integration tests used to skip
  silently when CI ran without a database, so a lost setting would have
  reported success while testing nothing. They now fail instead. Making that
  true exposed four more tests that were skipping through a different door,
  including the one that proves the two services will only talk to each other
  over their protected channel.
- **Documentation.** The merchant contract now has a document of its own, and
  the merchant panel describes and reproduces only the current scheme.
- **`037`'s implementation:** head `a36de13`, on top of `37abbde`; 128 files;
  `v2` run `35414081962` on that exact head, **success**. **`036`'s:** head
  `37abbde`, on top of `baa180e`; 87 files; `v2` run `35140516415`, **success**
  across all three jobs (guards and contract, web, crypto).
- **Previous phase:** `016-legacy-removal` — the repository no longer carries the
  legacy platform. The frozen legacy Go backend (289 files) is deleted, and so
  are the legacy how-to documents that described running it, the residual web
  client code with no working purpose after the switch (27 modules nothing
  imports, dead service and store members, the merchant dashboard cards fed by a
  statistics route that v2 deliberately does not serve, a stray script at the
  web client's root), and the instruction rules that applied only to it. Repository tooling, guards, CI comments,
  ignore rules and the instruction documents now describe a v2-only tree; every
  structural guard check is kept, including the ones that refuse the legacy
  module path and a root Go workspace.
- **What is deliberately retained.** The recorded legacy evidence -
  `docs/platform/legacy-deal-rules.md` and the frozen SQL snapshot under
  `docs/platform/legacy-sql/` - stays, because v2 code and a dozen accepted
  specifications cite it by rule identifier and line, and the export script that
  reproduces the snapshot stays with it. Provenance comments in `services/**`
  that cite a legacy file and line are left as they are; they resolve against
  the parent of the removal commit.
- **One role-visible change, decided by the owner, and it is not a permission
  change.** The support role's "Edit" control on a requisite could never save
  (no write route exists for that role) but was the role's only way to see a
  set of requisite settings and the counterparty list that v2 deliberately
  serves it. The owner ruled that a dead write control must not survive as a
  false affordance and that the replacement must not narrow what the role
  already saw: it is now an explicit read-only "view" control showing the same
  fields as text plus the counterparty list, issuing no write request; the
  administrator's edit flow is untouched. The earlier ruling that the support
  role never creates or owns trusted bank-notification devices stands.
- **Repository-only, by owner decision.** Nothing on the development stand was
  touched: its remaining legacy leftovers (an old database, a disabled
  reverse-proxy site, stale binaries and logs, an old secret-store process, an
  out-of-tree source copy) are recorded in the phase specification and need
  their own authorisation before anyone removes them. Production and mainnet
  were not touched, and the phase authorises no production cutover.
  Its implementation was head `baa180e` on `bd99154`; 357 files, 1528
  insertions, 124409 deletions; `v2` run `35037785540`, **success**.
- **Before that:** `035-live-cutover-validation` — the live half of the stand
  cutover, and the first phase whose evidence is a real chain rather than a fake
  node. It is what makes "the new financial path is verified end to end" a
  statement about observation instead of about tests. Its acceptance set is 73
  criteria: **71 passed**, and **two are owner-accepted deviations that are not
  counted as passes** — an administrator's second factor was configured after the
  reverse-proxy switch rather than before it, and two intermediate work sessions
  left no preflight record. Both are historically unrecoverable; the owner
  disposed of them explicitly, and the record says so rather than rounding up to
  "all passed".
- **What the live run proved.** A deposit discovered and credited exactly once; a
  deal carried through the production merchant route to settlement; a payout
  signed only after the owner pressed the button and confirmed by the network;
  automatic sweep and top-up running unattended; reconciliation agreeing. A
  second, small payout exercised recovery between "signed" and "broadcast" and
  landed in the genuinely uncertain state — body signed and stored, send outcome
  unknown — where recovery re-verified the body, broadcast nothing twice, and
  left exactly one transfer on the chain. The refusal paths were exercised too:
  an address outside the allow-list refused before the owner was ever asked, an
  administrator login refused on an infrastructure failure path, and the log relay
  delivering exactly one of seventy-eight records.
- **Four defects only a live run could find.** Two of them stopped deposits
  outright and neither was reachable by any test, because the fake node answers
  in a different address representation and never drops a connection: the node
  client kept idle connections longer than the upstream does, and the contract
  address in an event log was compared in the wrong representation, so every
  transfer record was skipped silently. A third made the merchant API impossible
  to use on a clean database, since nothing could issue the first API secret. The
  fourth was alert noise with teeth: an ordinary command handover raised a
  critical divergence alert, six times, on the one channel reserved for "the two
  sides disagree about whose money is where".
- **Its implementation:** head `bd99154`; `v2` run `35014753274`, **success**.
- **Before that:** `015-cutover-dev-stand` — the repository half of the stand
  cutover, split from the live half by owner decision so that repository checks
  could accept it on their own: alert delivery on the custody side (nine
  conditions, one owner channel, no operator text crossing to the messenger),
  the device self-update route, deployment templates guarded against
  host-specific values, configuration examples pinned by tests, the operator
  command-line tools, and the rule that a node's "found" is never trusted
  before a body has been verified against the signed intent. Head `caf80ad`
  on `232097b`; `v2` run `33986241821`, **success** across all three jobs.
- **Current/next task:** `056-rub-only-bank-events` parts A and B are developed,
  reviewed and **deployed to the stand and closed** (phone build first, then
  server; binding preserved; money state unchanged); `054-bank-fee-update-lost-row`,
  `053-merchant-bank-fee-concurrency`,
  `052-default-fee-grid-concurrency`,
  `051-trader-fee-replacement-atomicity`,
  `050-frontend-lint-restoration` and
  `049-device-requisite-authorization-gaps` are CLOSED;
  `048-production-config-fail-closed` is implemented and deployed to the
  development stand but not closed, awaiting its real-infrastructure
  criterion; `047-device-key-lifecycle` is CLOSED, and so are
  `046-device-offline-recovery`,
  `045-bank-source-and-name-catalogue` (in its autonomous part),
  `044-blocking-notice-classification`,
  `043-deal-attribution-ambiguity`, `042-bank-message-corpus` (technical
  part), `041-bank-event-durable-delivery`,
  `040-requisite-block-bypass`, `039-device-diagnostics-and-operator-health`,
  `037-android-preproduction-hardening` and `036-merchant-api-hardening`
  before it. The financial-core redesign program's
  phase list was already closed by `016-legacy-removal`; what remains are the
  separate pre-production tasks listed at the end of this file. The real
  parser corpus waits for the external gate. The task proving which deal a
  bank payment belongs to is only PARTLY DONE: its autonomous part is closed,
  and its remainder waits for the real corpus. The carried authorisation gaps
  are closed (`049`); the production-configuration item is implemented
  (`048`) and waits only for its check against real infrastructure; the
  default-fee-schedule and bank-specific-fee items are closed (`052`, `053`).
  No other item there has started. By owner decision each is a bounded task
  of its own, none starts automatically, and none of them is a production
  deployment.
- **Nothing breaks at the switch any more.** Device self-update was fixed in the
  cutover preparation; the merchant dashboard cards that could only go blank -
  their figures came from a withdrawal model that no longer exists, an exclusion
  the owner took earlier - were removed with the legacy tree. The support-role
  question that stayed open after the dead-surface phase is closed by the owner
  decision described above.
- **The cutover that was validated is the development stand's, not production's.**
  Production has not been cut over, and nothing in this phase authorises it. The
  live run happened on a test network with test funds, on a disposable private
  stand, under an authorisation that explicitly excluded production, mainnet and
  production secrets.
- **Next action:** `056` parts A and B are developed, reviewed and **deployed to
  the stand and closed**. Several things are
  outstanding that are not optional. The mandatory external validation gate
  below now holds four checks and none of them has been run; production
  readiness cannot be declared until all four are. Further items are recorded
  as mandatory before production readiness, listed at the end of this file -
  among them proving which deal a bank payment belongs to - plus two records
  there that the owner deliberately did not declare mandatory. The
  payment-attribution task's remainder cannot start before real bank messages
  exist. Tasks `048` to `054` are done (`048` except its real-infrastructure
  check). The two custody-host problems (deposit discovery stalling, the
  unrenewed secret-store credential) are now fixed, deployed and verified on the
  stand. No pre-production task starts by itself. The remaining owner questions
  from the autonomous cycles of 2026-09-24/26 are pending.
  PRODUCTION READINESS IS NOT CONFIRMED and PRODUCTION DEPLOYMENT HAS NOT
  STARTED: nothing so far has touched production, its hosts, mainnet or
  production secrets.

### Phase boundaries deliberately held

Structure exists in the baseline where behavior does not. A table being present
is not evidence that its behavior is implemented.

- **The v2 core has been exercised against real infrastructure once, on a test network.** Until
  the live cutover phase the deposit, sweep and withdrawal paths were covered by tests only; that
  phase ran them on a live test-network stand with a real bot and a real owner approval. It proves
  the development stand, not production: production has never run any of it.
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
  to a surface it must never reach; a migration that would have replaced a merchant's own API credentials
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

- Tasks 020-034 completed and remotely verified: a bounded pre-cutover programme of
  thirteen implementing tasks and two owner decision records, run across five gates, after
  which a systematic sweep of every call the web and device clients make found no live
  surface without a backend behind it, beyond the exclusions the owner accepted on purpose
  and which are named above. That was a point-in-time finding: nothing in the build keeps it
  true, so the next client change can falsify it silently. The durable results, all still in force: an
  administrator's edit of a trader's insurance requirement moves no money - it changes the
  size of a hold, so what changes is how much of the trader's own balance is reserved, and
  merchants are refused outright because no merchant insurance hold exists; support keeps
  its diagnostic reads but receives no raw bank message body and no full payment number
  anywhere, and cannot provision a device, because a device is part of the trusted channel
  through which deals are confirmed; a trader cannot read another trader's device,
  requisite or notifications, and an unreachable object answers exactly as a nonexistent
  one does, on every method, because a different answer is itself the disclosure; evidence
  that a deal was confirmed survives the deletion of the device that produced it; and one
  rule runs through the panels - a money figure the panel does not have is never rendered
  as zero, so a failed read says so instead of reporting that the platform holds nothing.
- Task 015 completed and remotely verified: the repository half of the stand cutover, split
  from the live half by owner decision. Nine owner decisions are recorded in its
  specification. Four of the five entries under Known traps come from it, and two of
  those cost several review rounds each: a guard that refuses a secret must not quote it, and a
  safety rule that enumerates bad inputs will be defeated by the next input nobody
  enumerated. The independent reviews earned their place here as they did earlier: before
  any code existed they caught an acceptance criterion resting on a command that cannot run
  in this baseline - the second time that same criterion had to be removed - and after
  implementation they caught a freeze that held a user's money and reported it to nobody,
  a concurrent rebuild that would have raised a false alarm about a healthy request, and a
  guard whose failure message published the password it had just refused.

## Known traps

- **The legacy backend is gone; read it from history, not from the tree.** Provenance
  comments in `services/**`, the legacy rules document and older specifications cite
  legacy files by path and line. Those resolve only at the parent of the commit that
  deleted the legacy tree (`git log -1 --diff-filter=D -- backend` finds it). Do not
  "fix" such a citation by pointing it at v2 code, and do not recreate a legacy file to
  satisfy it. The recorded evidence under `docs/platform/` is the frozen copy the rule
  identifiers and the SQL snapshot refer to; its line numbers are stable because the
  files are.
- **A service token that is a child of the bootstrap root token dies with it.**
  Revoking a bootstrap root token is ordinary hygiene, and it silently revoked a
  long-lived service credential that had been verified as *separate* — different
  value, own policy, own file. Separate is not the same as independent: the
  check that matters is whether the token is an orphan. The failure is delayed
  and therefore deceptive, because the service keeps running on what it already
  read and only fails at its next restart, when nobody connects the two events.
  A long-lived service token must be created as an orphan.
- **Unseal material stored beside the data it protects protects nothing at
  rest.** Whoever reaches the host reaches both. Acceptable on a disposable
  stand, never in production, and the distinction has to be written down where
  the next reader will look rather than assumed.
- **Compile-time configuration makes every environment a separate build.** The
  device application still takes its server address at build time, so an
  artifact built for one environment cannot be pointed at another, and every
  environment needs its own build. What changed is that the address now has no
  built-in default at all: a build that does not supply it fails instead of
  quietly pointing a phone somewhere. The shared secret that used to travel in
  the artifact alongside it is gone. Discovering the per-environment build
  property when the hardware arrives is expensive; it is a thing to check
  before a phase depends on it.
- **A negative observation without a positive control proves nothing.** "No
  alerts were delivered" and "the alerting is dead" look identical. Every
  selectivity check in this phase carries a deliberate positive delivery in the
  same interval, and the same discipline caught a divergence-alert fix that
  could otherwise have been mistaken for working.
- **An exclusion keyed on an identifier can silence more than it means to.** The
  fix for the false divergence alerts was first keyed on the request identifier,
  which also masked divergence that existed for an unrelated reason, because the
  open-request list is a union of independent sources. Keying it on the row
  actually being handed over is the narrow form; the wide form passed every test
  until a reviewer constructed the case.

- **A guard that reports a secret must not quote it.** The template guard refused a
  connection string carrying a password and printed the password in the refusal,
  into a log that outlives the refused commit. It took four rounds to close,
  because each round blacklisted the character that had just been demonstrated -
  first the credential prefix, then a slash in the password, then seven more
  separators. What finally worked was inverting the rule: print the address only
  when it matches a safe shape **and** the line carries no credential separator at
  all, which is a syntactic property rather than a list of characters. The lesson
  generalises past this guard: a check that decides safety by enumerating bad
  inputs is a check that will be defeated by the next input nobody enumerated.
- **A test that asserts only "not the success code" proves almost nothing.** The
  path-traversal criterion was satisfied by a status check that a plain routing
  mismatch also satisfies, so it would have passed if the guard it exists to pin
  had been moved ahead of the check that actually answers. Assert the body.
- **Routing decides more than it looks like it does.** A parent reference reaches
  the handler as an ordinary parameter value, while anything carrying a path
  separator - encoded or not - is answered by the router before any handler runs,
  because matching happens on the unescaped path. Both are safe here, but they are
  safe for different reasons, and a criterion that assumes one mechanism for both
  describes something that does not happen.
- **Detection is not delivery.** An alert condition can be implemented, tested and
  still reach nobody, because the state it produces is not the state the notifier
  watches. The freeze introduced by the recovery check was exactly that: correct,
  covered, and silent. Check the path from condition to person, not the condition.
- **The frontend lint command works again, against a recorded baseline** (task
  `050`). A green lint means no file gained a violation of a new rule and no suppressed
  per-file, per-rule count grew - not "no findings", and not that each finding
  is the same one as before (one suppressed finding swapped for another of the
  same rule in the same file is not detected):
  464 historical errors are suppressed on purpose and remain debt. Fixing one
  requires shrinking the baseline in the same change; regenerating the baseline
  to hide a new finding defeats it. The build still does not type-check.

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
- **The body of a multipart upload is read inside an open database transaction**, and a
  server-side deadline on that read is load-bearing rather than defensive: without it, a
  request that is slow to send holds a pooled database connection, and the pool is shared with
  every money path in the service. The deadline is deliberately injectable so tests can prove
  it, and weakening or removing it is a change that must be refused in review.
- **Several criteria of the dispute phase are verified by a reviewer, not by the pipeline.**
  Which ones, and how each is checked, is recorded in the private repository. The public point
  is the standing obligation: they are commitments of whoever reviews next rather than
  guarantees the build already enforces, and moving them into automation is open work.
- **Evidence is never destroyed by an ordinary delete, and the sandbox's current answer is not
  its contract.** An ordinary delete hides a dispute document from every role that can
  see the dispute, the one adjudicating it included, while the bytes and the history
  stay - so nothing is destroyed, but nothing is visible either. That is deliberate,
  carried behaviour and it is **not** a resolved question: physical destruction, and
  whether an adjudicator should keep seeing what a party removed, are separate owner
  decisions that have not been taken. The sandbox dispute route today answers "deal not found"
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
- Do not reintroduce the removed legacy module path into v2 - the legacy backend
  is gone from the repository and a guard refuses that import path - and do not
  expand a scaffolding phase into later financial phases.
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
  test. They spanned three kinds: authorisation gaps inherited from the legacy
  platform on some device and requisite routes, input validation weaker at
  bind time than the field definitions suggest, and cosmetic inconsistencies in
  responses and counts. **The authorisation gaps and the bind-time validation
  are closed** by task `049`, under owner decisions. The cosmetic
  inconsistencies, and a few recorded behaviours that are not authorisation
  questions, remain as carried; each is enumerated in the private
  specification, and removing one without the owner's decision is still a
  defect rather than a cleanup.
- The audit path never serializes a whole row. It writes an explicit allowlist
  of safe columns, and a test proves password hashes, MFA secrets and unconsumed
  invite codes never reach the audit table. Do not "simplify" it into a row
  dump, and do not add a read-back to make an audited value look tidier.
- Web deliberately holds no Telegram bot token by accepted architecture, so
  some notifications the legacy platform sent on authentication events have no
  transport in v2. The controls themselves are preserved; only the notification
  is gone, and restoring that visibility is open work.
- **Parts of the authentication hardening are configuration-sensitive, and that
  is inherited rather than new.** Some of these controls depend on external
  infrastructure and on deployment configuration being present, and the
  behaviour when that configuration is absent is not uniformly the same as when
  it is present and healthy. Which direction each one takes, and under what
  conditions, is recorded in the private repository and is not published here.
  Task `048` made a production configuration refuse to start without them and
  removed the silent degradations there, by owner decision; the development
  stand keeps the inherited behaviour on purpose. What is still open is the
  check against **real production infrastructure**: tests on isolated processes
  are not that check, and a stand behaving well is not either.
- Replacing the legacy request filter moved where input limits and character
  rules are enforced, and the set of refusal responses on the authentication
  routes changed with it. Per-account lockout and route rate limiting remain.
  The exact before-and-after, and which limits now live where, is in the private
  repository; the public point is that this boundary moved and must be
  re-verified against a production configuration rather than assumed.

- Device authentication is sensitive to **how the request body is handled before it is
  verified**. Re-serializing or normalizing it on the way in - including anywhere a
  framework binding has already consumed and re-encoded it - breaks device
  authentication while looking correct in review, and must not be done without a
  separate, deliberate protocol migration. The pre-production hardening task reset
  every existing pairing deliberately, so an installed fleet is no longer a reason
  not to change this; the verification rule is unchanged, still load-bearing, and
  recorded in the private repository.
- Four of the notification statuses the SMS pipeline can produce are not values the
  status column accepts, so those rows are silently never stored. That is legacy
  behavior and is carried, which is only safe because the notification write is
  deliberately its own statement outside any transaction that must still commit.
  Put it back inside one and a blocked SMS would stop blocking the requisite.
- The bank catalogue is not decoration. Requisite creation validates the submitted
  bank against it, and one of its flags decides whether a trader may create an SBP
  requisite, so an empty or invented catalogue changes what traders can do.
- Configuration defaults are part of the ported behavior. The legacy backend
  (now removed from the repository) supplied built-in defaults for the Android
  release descriptor, so "unset" never reaches its handler; an empty default in v2 would tell every current device that
  an update is available, with no version and no link. Where legacy has a default,
  carry it, and cite the line it came from.
- Do not add a device-log table because the ingestion endpoint appears to need one:
  the bodies are deliberately not stored. The device-health store added by
  `039` is a different thing - typed health facts, never message bodies.

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
  engine rather than under the panel's route group, so they do not inherit the panel's
  middleware. That was a gap until the hardening task, which gave the surface its own
  limits and controls instead of moving it under a group whose behavior it does not
  want; the placement itself is unchanged and still has to be remembered when reading
  the routing.

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
  is not a transient event**, so a validation rule added carelessly becomes an
  availability problem rather than a rejected message, and the operational visibility
  of that situation is itself open work. The settled split of what is validated, and
  how strictly at each layer, is recorded in the private contract; review moved it
  twice before it was right, and the principle that survived is that a wrong amount on
  a money event must never be accepted with a log line, while forward compatibility is
  preserved where it costs nothing.
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
  issuing the material. One administrative tool is outside that requirement for an
  ordering reason recorded privately, and that exception is deliberate, narrow and
  written down rather than discovered. An environment-keyed bypass on a custody
  boundary is the defect class this project has already been bitten by.
- Three limits bind the phases that come next, and each is enforced by a refusal on a
  wire where a refusal is not transient, so exceeding one is an availability problem
  rather than a rejected message. They are the event batch size, the reconcile request
  list, and a deposit-address command whose request id must be derived deterministically
  from the trader so that a retry cannot mint a second address. The values are in the
  frozen contract in the private repository.
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
  the timeout that feeds it is ordinary configuration, and nothing in the code ties it to
  the lifetime it has to outlast. Validating that relationship against a production
  configuration, rather than setting the two independently, is part of the
  configuration review the pre-production work owes.

- Phase 012's traps are about not losing money that is already on the chain. **A
  discovery source must declare its completeness.** The index either answers completely
  for the requested interval or marks its answer truncated and says how far it is
  complete, and the scan cursor advances only to that boundary minus an overlap - never
  to the newest timestamp it happened to see. A paginated, newest-first source plus a
  cursor that trusts it silently loses the oldest transfers in a busy window, and the
  only thing that would ever notice is a balance check.
- **Measure confirmation depth against the block the current receipt names**, and rewrite
  the stored block while the deposit is still unconfirmed. Freezing the block discovered
  first can credit a deposit before the accepted depth has actually been reached under
  chain reorganisation, and the secondary control that catches a vanished transaction does
  not cover that shape. The payout tracker already had this right; the deposit path had to
  be brought into line with it, and the rule above is what keeps it right.
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
  panel that consumes them; that boundary is closed. The panel was exercised against the running
  v2 stand in the live cutover phase; the legacy backend that the stand used to serve no longer
  exists in the repository.
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
  client puts operator credentials and panel session behaviour on a surface built for a paying
  customer's browser, which is a trust-boundary error rather than a styling one. Express that
  boundary as a separate component, not as a flag on a shared one.

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

None of this is a production deployment. Classes: **A** can be prepared in the
repository; **B** needs production infrastructure; **C** needs real bank
messages and phones; **D** needs an owner decision. Class A is done for this
stage. Deferring an item is not a pass, a waiver or an accepted risk.

- **Owner architecture review (D) - the next mandatory step.** Decisions the
  owner is asked to take there: which application role the customer uses day
  to day, given how broad the top administrative role's powers in the panel
  are; the second-factor policy for each administrative role;
  where releases are built; the restore and failover policy that must not
  re-bind the phone fleet or re-sign an already-signed transfer; whether the
  secret store's storage is backed up or recovery relies on the owner's
  offline copies.
- **Execute the deployment package on real infrastructure (B)** after the
  review and a separate permission, including the fail-closed configuration
  check of task `048`, which stays **not passed** until run there, and the
  unseal ceremony.
- **External validation gate (C).** One organised session with prepared
  procedures and pass criteria: a genuine bank notification on a bound
  handset, restore onto a second physical handset without cloning its
  identity, handsets from other manufacturers, and a real anonymised corpus
  of the banks and channels used at launch. Not run; real samples: 0. It is
  not closed with synthetic data and does not block the documentation work;
  what is needed just before the session is listed.
- **Proving which deal a bank payment belongs to (C, then D).** The matching
  is still not declared safe; residual risks of confirming a deal a payment
  was not made for, and of settling one payment twice, stay open. The
  decision is taken only after the real messages show what they contain; a
  decision tree is prepared. The SMS-box channel is not part of it for the
  first release. Also dependent on the corpus: which applications may be a
  source of a payment notification, and how many rouble payments are wrongly
  sent to review.
- **Recorded, deliberately NOT declared mandatory by the owner:** the
  structural check protecting the blocking rule is a fence rather than a
  proof; an operator refused a forbidden operation on a blocked requisite sees
  the wording of a different refusal.
- **Recorded and left open, neither closed nor accepted:** a silent phone
  keeps receiving new deals until the offline threshold passes, and why the
  server once failed to record heartbeats is not established; accounting
  integrity is enforced by the code rather than by the database; idempotency
  of withdrawal creation is an open item (every withdrawal still requires the
  owner's approval). These are listed for the owner's verdict in the review.

## Update policy

Rewrite this file after completion of a phase, a changed accepted architecture
decision, a branch or current-task change, or discovery/resolution of an
important known trap. When something stops being true, replace or remove it.
Do not preserve contradictory old decisions as history, and do not turn this
document into a changelog.
