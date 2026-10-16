# African Bitcoin Settlement Layer (ABSL)

**Technical project overview — working draft**  
**Updated:** 16 October 2026  
**Status:** Early-stage design and software prototype. Not audited. Not ready to protect real customer funds.

## 1. Purpose

The African Bitcoin Settlement Layer (ABSL) is a proposed protocol and software system for trades where a customer pays a provider in Bitcoin over Lightning and the provider delivers local currency or an agreed service.

ABSL is intended to make the order explicit before the customer pays: the amount in satoshis, the provider’s promise, the collateral pool that backs the order, the reservation record, and the process for settlement or dispute. The protection is Bitcoin-denominated: if the protected principal is 5,000 sats, the claim is for 5,000 sats, not a fiat-value recalculation.

ABSL is not a marketplace, Lightning network, wallet, or Bitcoin custodian. It coordinates order admission and evidence and is intended to connect to Bitcoin spend paths that enforce refunds, default payouts, and provider exits. Those Bitcoin paths are not implemented yet.

## 2. Relationship to discovery

A discovery service helps a wallet or provider find another participant and inspect a listing. Discovery alone does not prove that a specific order fits within available collateral, reserve that capacity, bind both parties to the same invoice and terms, verify Lightning settlement, or enforce a Bitcoin payout.

ABSL is designed to complement discovery with order-level capacity checks and settlement records. Discovery may remain a separate service; Nostr is an optional future transport for listings and status. A relay transports messages but does not establish collateral, order capacity, or payment truth.

## 3. Architecture

- **Wallet:** Finds a provider, submits an order intent, supplies the required on-chain refund destination privately, verifies the accepted order and invoice package, pays, and keeps its records.
- **ABSL order gateway:** Receives order intents, queues them for the right pool, relays accepted orders to the provider, validates the provider’s invoice package, and privately delivers that package to the requesting wallet.
- **Shared sequencer and public pool ledger:** For the first open-ledger prototype, one shared sequencer commits each provider-pool block. It orders requests for that pool and records accepted or rejected decisions and the resulting capacity totals.
- **Provider node:** Operates the service, creates an invoice only after an order is accepted, sends the invoice package to ABSL, checks its Lightning backend for settlement, and delivers the agreed fiat or service.
- **Lightning backend:** Creates invoices and reports invoice status. The prototype has an adapter boundary, but no production Lightning connector.
- **Bitcoin layer:** Must ultimately verify collateral UTXOs and enforce the reviewed script and transaction paths. It is not connected to the current prototype.
- **Discovery:** Optional directory and status transport, outside the settlement authority.

### Pool boundary

One pool represents one provider collateral pool and its separate collateral set. A provider that wants a second pool must register and fund separate collateral for it. Each pool has its own reservation history and capacity accounting; a request for one pool cannot spend another pool’s capacity.

### Sequencer trust

The first prototype uses one shared sequencer to serialize pool blocks. This avoids asking a committee to sign each ordinary reservation, but wallets must trust the sequencer’s ordering and availability. A sequencer signature proves which key signed a block; it does not prevent censorship or the sequencer from signing competing histories. The prototype has no replicated consensus.

## 4. Intended order flow

1. **Find a provider.** The customer or provider discovers the other party through a directory, Nostr, or another channel.
2. **Submit an order intent.** The wallet sends ABSL the pool identifier, requested sats, order identifier, and a commitment to the on-chain refund destination. The actual refund destination and invoice remain private.
3. **Queue and admit.** ABSL queues the request for that pool. The sequencer processes the pool batch on its 60-second cycle, checks the registered pool and current exposure, then commits an accepted or rejected result in a signed pool block. An order is accepted only if its full principal fits the rules; ABSL does not reserve half an order.
4. **Create the invoice.** Only after the accepted reservation is committed does the provider create a fresh Lightning invoice. The provider sends the invoice and its binding data to ABSL, not directly to the customer.
5. **Validate and deliver privately.** ABSL parses and checks the invoice against the accepted intent and terms, then sends the valid invoice package privately to the requesting wallet. The invoice is intended to have a 60-second payment window. Relay or publication of an order request alone does not make it insured.
6. **Pay and verify.** The wallet pays the invoice. The provider’s Lightning backend must report the invoice as settled before the order advances. The current prototype’s status interface is supplied by the provider; a real backend and the required trust/evidence contract are still missing. Pending or unknown status must not be treated as success or as a reason to release capacity.
7. **Deliver and resolve.** The provider delivers the agreed fiat or service. The order remains accounted for until a valid, recorded release or other terminal transition applies. Disputes and Bitcoin payouts require rules and transaction paths that are not yet implemented end to end.

The 60-second batch interval and 60-second invoice lifetime are separate clocks: the batch first decides whether capacity is reserved; only then is the invoice created and its payment window started.

## 5. Capacity and collateral accounting

Let:

- `C` = the verified collateral assigned to this pool;
- `F_res` = the protocol’s single fee reserve for required Bitcoin transactions;
- `NetBase = max(C - F_res, 0)`;
- `S_order_max = floor(25% × NetBase)`;
- `pool_cap = floor(50% × NetBase)`, or a lower provider-selected cap;
- `available = max(pool_cap - outstanding_exposure, 0)`.

The 25% per-order ceiling limits the size of one order. The 50% aggregate ceiling limits the total exposure across all outstanding orders in that pool. Each accepted order consumes its full sats principal from capacity. If the full order does not fit the per-order limit, aggregate available capacity, and unencumbered collateral check, it is rejected. When capacity is exhausted, new orders are rejected.

Ordinary disputes do not themselves move or freeze the Bitcoin collateral. Their amounts remain included in the pool’s exposure accounting while unresolved. Actual collateral payout occurs only through a valid Bitcoin spend path, which has not yet been implemented.

These formulas describe the current working model. The exact `F_res` value, confirmed collateral amount, and complete rules for how every order state consumes or releases capacity still require validation. In the prototype, collateral values are inputs: the code does not independently establish that the Bitcoin exists, is controlled by the provider, is unspent, or is spendable under the promised script.

## 6. Public ledger and private information

The public pool ledger is intended to expose enough information to audit pool ordering and capacity accounting, including the pool identifier, block ancestry, order identifier, sats amount, request commitment, accept/reject decision, and capacity totals.

It must not publish the Lightning invoice, payment hash, refund script or salt, payer identity, or private fiat-delivery evidence. ABSL must deliver invoice and refund information only to the parties that need it, with dispute evidence restricted to the parties and authorized reviewers.

Public visibility helps participants inspect the same committed pool history. It does not, by itself, prove collateral or stop a malicious sequencer from creating conflicting histories.

## 7. What the prototype currently implements

The Rust codebase contains:

- Shared protocol types for terms, reservations, pool registration, order intents, pool blocks, invoice bindings, exposure records, and wallet verification.
- Deterministic capacity arithmetic and declared order-state transitions.
- A provider-local SQLite reservation flow with atomic local commits.
- A separate SQLite-backed reference sequencer with per-pool queues, a 60-second batch worker, signed-block commits, ancestry replay, and read routes.
- A provider HTTP route that supports the earlier provider-local order path.
- Draft invoice-validation and wallet-verification components.

The new open-ledger flow is **not connected end to end**. The provider-local route still creates an invoice before local reservation. The sequencer uses operator-imported pool registration and does not independently verify Bitcoin collateral. The invoice-validation gate is not yet routed through the sequencer flow. There is no private invoice relay/storage path, complete release lifecycle, real Lightning connector, or integrated provider and wallet flow. A successful test of an individual component does not prove the full system is safe.

## 8. Main unresolved work

Before a controlled test can be considered, the project still needs to:

- Define and independently verify eligible collateral UTXOs, pool ownership, confirmations, and exclusive outpoint assignment.
- Connect the intent-first flow from wallet through sequencer, provider, invoice validation, and private delivery.
- Define the invoice-expiry and release record so an unpaid or unissued invoice cannot lock capacity indefinitely, without allowing a false release to harm a payer who has paid.
- Finalize the Lightning settlement verification contract and connect a real provider Lightning backend. A preimage matching an invoice hash is not, by itself, independent proof that the payment settled.
- Define authenticated APIs, TLS, rate limits, signer key custody, backups, operational recovery, and sequencer failure behavior.
- Implement and review Bitcoin collateral, refund, payout, and exit scripts and transactions, including fee handling and confirmation observation.
- Define order deadlines, dispute evidence, appeal windows, and capacity release timing consistently. A short post-payment hold must not silently erase a longer valid claim period.
- Publish canonical encodings and independent conformance vectors for pool registration, order intent, pool blocks, invoice binding, and related records.
- Test the end-to-end flow on local fixtures and a Bitcoin test network, then obtain independent security review.

## 9. Status and scope

This is a technical working overview, not the controlling rulebook or a final interoperability specification. It distinguishes the current implementation from the target architecture and records important gaps so a prototype capability is not mistaken for a production guarantee.

ABSL remains early-stage, unaudited, and unsuitable for real customer funds. The code and protocol must not be presented as proving collateral-backed protection until collateral verification, Lightning settlement handling, lifecycle enforcement, and the Bitcoin spend paths are connected and independently reviewed.
