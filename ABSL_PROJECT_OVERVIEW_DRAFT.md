# African Bitcoin Settlement Layer (ABSL)

**Project overview — technical working draft**  
**Project status:** Early-stage design and software prototype. Not audited. Not ready to protect real customer funds.

## 1. Purpose

The African Bitcoin Settlement Layer (ABSL) is a proposed protocol and software system for trades in which a customer pays a provider in Bitcoin over Lightning and the provider delivers local currency or an agreed service.

ABSL is intended to make each order explicit before payment: what the provider promises, how much Bitcoin exposure the order creates, what evidence records the order, and what process applies if the provider fails to meet the agreement. Bitcoin collateral is intended to back eligible customer claims under rules that must ultimately be enforced by Bitcoin transactions.

ABSL does not itself provide a marketplace, a Lightning network, or a Bitcoin custody service. It defines settlement rules and coordinates software components that interact with those systems.

## 2. Why settlement belongs beside discovery

A discovery protocol helps participants locate providers and publish service information. It can improve access and reduce the effort needed to find a trading partner. Discovery data alone, however, does not establish that a particular order fits within a provider’s available collateral, bind the customer and provider to the same order terms, prove that a Lightning payment settled, or enforce a refund or default payout.

ABSL is designed to complement discovery by handling order-level commitments and settlement. Discovery can help a wallet or provider find a counterparty. ABSL should then provide a direct order flow that checks capacity, binds exact terms, records a reservation, and exposes evidence that a wallet can validate.

Nostr may be used as an optional transport for provider listings or discovery messages. A Nostr relay is not the settlement authority: relays transport events but do not verify Bitcoin collateral, determine global capacity, or enforce a payout. A provider’s signed status message is still a provider assertion unless it is backed by independently verifiable evidence.

## 3. System components and boundaries

### Customer wallet

The wallet finds a provider, requests an order, checks the returned terms and provider signature, pays the Lightning invoice, and retains the order record and any evidence needed later. The wallet is a verifier of the records it receives; it cannot independently know that a provider has not shown a conflicting history to another customer.

### Provider node

The provider node coordinates order admission. It obtains or creates the invoice through a Lightning backend, constructs the order terms, calculates current exposure, signs a reservation record, and stores the reservation. The invoice should be returned to the wallet only after the reservation has been committed successfully.

### Reservation ledger

The ledger records the provider’s order exposure over time. In the prototype, reservation entries are signed and linked by sequence number and a previous-record hash. A local SQLite transaction provides atomic append behavior for processes that share the same database: two local requests should not both commit as the same next revision.

That local property is not a global consensus mechanism. It does not stop the provider from running another database or signing a competing branch. Cross-database equivocation detection and recovery remain open implementation work.

### Lightning backend

The Lightning backend creates invoices and, in a complete system, reports whether each invoice actually settled. Invoice creation and payment verification are separate operations. An invoice’s preimage is mathematically related to its payment hash, but a provider that created the invoice may already know the preimage. Therefore, a matching preimage supplied by a wallet is not, by itself, independent proof that the provider received a settled payment.

The current prototype has an interface boundary for invoice creation and cancellation, but no production Lightning connector and no settlement query wired into the order lifecycle.

### Bitcoin layer

Bitcoin is intended to hold provider collateral and enforce the allowed spend paths for refund, default payout, and provider exit. A record in the provider’s database does not lock Bitcoin. The protection exists only if the collateral output, script conditions, signatures, timeouts, and transaction paths are correctly constructed and available when needed.

The current prototype does not verify collateral UTXOs, create or broadcast the collateral script transactions, monitor confirmations, estimate real transaction fees, or execute refunds and payouts.

### Discovery layer

Discovery is an optional front door. It can help a customer or provider find a counterparty and retrieve a listing. It should not be treated as the source of truth for a specific order’s capacity or settlement state. The final order data should be obtained from the provider order API and checked by the wallet.

## 4. Intended order lifecycle

1. **Discovery:** A customer or provider locates a counterparty through a discovery service or another channel.
2. **Order request:** The wallet asks the provider node for a specific amount and supplies the required refund information.
3. **Private invoice creation:** The provider node asks its Lightning backend to create an invoice but does not yet reveal it to the wallet.
4. **Terms construction:** The node builds a canonical terms record binding the invoice details to the order, amount, refund destination commitment, provider identity, and relevant policy identifiers.
5. **Capacity admission:** The engine calculates whether the order fits the provider’s per-order and aggregate limits and whether existing pending, open, and disputed exposure leaves sufficient capacity.
6. **Signed reservation:** The provider signs a reservation record that commits to the order terms and links to the preceding ledger record.
7. **Atomic commit:** The reservation, event, revision, and signed receipt are committed together to the local ledger. If admission or persistence fails, the invoice is not returned; cancellation is attempted, with invoice expiry as a fallback.
8. **Wallet verification and payment:** The wallet checks the terms digest and reservation signature before paying the invoice. A future implementation must separately verify actual Lightning settlement against the provider’s node or another defined evidence source.
9. **Delivery and resolution:** The provider delivers the agreed local currency or service. The order moves through explicit lifecycle states and eventually settles, expires, is refunded, or enters a dispute process under rules that are still being completed.

The current HTTP admission route implements only the early order-creation portion of this flow. It does not observe customer payment, verify settlement, record delivery, or resolve disputes.

## 5. Exposure and capacity accounting

ABSL treats the customer’s protected principal as a Bitcoin amount. If a customer pays a stated number of satoshis, the intended principal claim is denominated in those same satoshis; it is not recalculated to preserve a fiat exchange value.

The current prototype documentation uses these working calculations:

```text
NetBase = max(collateral amount C - fee reserve F_res, 0)
per-order ceiling = floor(25% × NetBase)
aggregate ceiling = floor(50% × NetBase)
available capacity = selected aggregate cap - outstanding exposure
```

Outstanding exposure includes the order states that the engine treats as consuming capacity, including pending, open, and disputed orders. A provider may configure a lower cap, but not a higher cap than the protocol ceiling. The 25% and 50% values are working parameters in the current design; they should not be described as independently validated collateral protection.

At present, collateral amount and status are inputs to the prototype. The code does not independently establish that the Bitcoin exists, belongs to the provider, is unspent, or can be spent along the promised paths. Capacity arithmetic therefore demonstrates accounting behavior, not proof of secured funds.

## 6. Protocol records and wallet checks

The protocol crate contains draft encodings for order terms and signed reservation entries. The terms record binds the invoice and order parameters into a canonical byte sequence and digest. The reservation record identifies the provider, order, amount, terms digest, sequence, and previous-record hash, and carries a provider signature.

The wallet library can check the provider signature, the terms digest, and whether a successor record links to the prior record known to the wallet. This provides integrity for the specific records received. It does not prove that the provider has only one ledger history or that the collateral assumptions are true.

The wire formats are draft proposals. The repository currently has one implementation and in-repository vectors; independent encoders and conformance review are still required before the formats can be treated as interoperable standards.

## 7. Trust assumptions and failure cases

ABSL is not trustless merely because records are signed or carried over an open network. A signature shows which key signed data; it does not prove the data is true. A provider can sign an inaccurate collateral status. A single database can coordinate local requests but cannot prevent a separate database from producing a conflicting branch. A discovery relay can deliver a listing without verifying the provider’s financial state.

A production design must define how wallets verify collateral, payment settlement, reservation history freshness, and dispute evidence. It must also specify what happens during network failures, stale records, provider shutdown, chain congestion, fee spikes, and conflicting reservation histories. Until those controls and Bitcoin transaction paths are implemented and audited, the prototype must not be used as assurance for real funds.

## 8. Current implementation

The repository currently includes:

- A Rust workspace with shared protocol, engine, adapter, provider-service, and wallet-verification components.
- Draft terms and reservation record formats with signatures and validation.
- Deterministic capacity arithmetic and declared order-state transitions.
- A SQLite-backed local ledger with atomic reservation commits and revision checks.
- A provider HTTP admission route that returns an invoice, terms, and signed reservation only after local commit.
- A wallet verification library for provider signatures and linked reservation records.
- Automated tests for the implemented encoding, arithmetic, state, persistence, concurrency, and verification behavior.

This is an early software prototype, not an operational service. The provider executable does not yet start a configured production server or load a real backend. Test success applies to the code that exists; it does not establish end-to-end safety or enforceable collateral protection.

## 9. Work still required

Before a real-funds pilot, the project needs at least:

- A production-quality Lightning integration and a precise settlement-verification contract.
- Independent verification of collateral UTXOs and confirmation state.
- Reviewed Bitcoin script and transaction templates for collateral, refund, default payout, and exit paths.
- A complete order-state, expiry, dispute, adjudication, and payout implementation.
- A defined response to provider ledger forks, including detection, evidence preservation, and recovery rules.
- Canonical cross-implementation vectors and independent implementations of the protocol encodings.
- Operational security for keys, API credentials, node deployment, backups, monitoring, and recovery.
- End-to-end testing on a Bitcoin test network and an independent security audit.

## 10. Scope and status

This document is a technical project overview, not a final specification. It describes the current direction and prototype boundaries. Specific values, record formats, and procedures may change as the protocol is reviewed.

ABSL is being developed to explore settlement support for Bitcoin-denominated trades alongside discovery systems. The software and rules are incomplete, unaudited, and not ready to protect real customer funds.
