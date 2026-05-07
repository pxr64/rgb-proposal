# Threat Model

Threat model for RGB RFQ Network. Scope: v1, on-chain RGB20 settlement. This document covers protocol-level threats; deployment and key-management threats for individual maker operators are out of scope and the responsibility of the operator.

## Trust Assumptions

- **Broker is untrusted with respect to assets and keys.** It never holds funds, RGB stash, or signing material. A malicious or compromised broker can degrade service or selectively relay messages, but cannot move assets or forge state.
- **Maker is trusted only by the takers who choose to trade with it.** Maker selection is the taker's counterparty and privacy decision. The protocol does not require makers to trust each other.
- **Bitcoin layer security and RGB client-side validation are assumed sound.** Threats against the underlying chain or against rgb-lib's validation logic are out of scope here.
- **The taker's wallet is responsible for invoice generation and consignment validation.** The protocol does not protect against a taker wallet that mishandles its own seals.

## Threats and Mitigations

### Quote-stuffing / RFQ flooding

A taker (or a maker masquerading as a taker) floods the broker with RFQs to exhaust maker reservation capacity, spam quote computation, or amplify load on downstream makers.

*Mitigation:* per-session rate limits at the broker; maker-configurable RFQ admission policy; per-pseudonymous-session reservation budgets. RFQs are cheap to drop early.

### Malicious broker leaking RFQs

A broker logs RFQ payloads or routing patterns and sells or leaks them to third parties.

*Mitigation:* the broker never sees taker UTXOs, unblinded seals, or consignment payloads, so leakage is bounded to "asset, side, amount, session pseudonym." The protocol does not assume a single broker; takers can choose among independent brokers, and wallets can rotate brokers per RFQ.

### Maker griefing (quote-then-refuse)

A maker quotes aggressively to attract takers, then refuses to settle on acceptance, wasting the taker's round-trip and potentially blocking competing quotes.

*Mitigation v1:* short quote expiries; broker tracks acceptance/settle ratio per maker; takers (and brokers) can prefer makers with healthy ratios. Future: optional quote bonds posted by the maker, slashable on griefing.

### Reservation-table denial of service

An adversary requests quotes repeatedly and lets reservations expire, keeping maker UTXOs locked and starving honest takers.

*Mitigation:* short reservation TTLs; per-session reservation budgets enforced by the maker; maker-controlled quote admission policy. Reservation budgets are local to each maker, so no global synchronization is required.

### Consignment delivery failure or interception

Consignment fails to reach the taker, or is intercepted in transit.

*Mitigation:* consignment delivery precedes PSBT broadcast on the happy path; if delivery fails, the maker aborts before broadcasting. Delivery transport is retriable. Confidentiality of the consignment payload is not a load-bearing assumption — the on-chain commitment anchors the state, and possession of the consignment alone does not move funds.

### Replay or stale-quote acceptance

A taker accepts an expired or replayed quote, or an attacker replays a captured acceptance.

*Mitigation:* quotes are signed with maker identity, scoped to a single quote ID, and re-checked against the maker's local reservation table at acceptance time. Expired quote IDs are rejected at the maker before any settlement work begins.

### Cross-quote double-spend by a single maker

A maker issues two quotes against the same UTXOs and tries to settle both.

*Mitigation:* the reservation table is the single source of truth for maker inventory commitments. Two quotes cannot reserve the same UTXO; settlement consumes the reservation, after which the UTXO cannot back another quote until reservation release.

### Timing-correlation linkage

A broker, or an external observer with broker-side access, correlates the timestamp of an acceptance message with the appearance of a new Bitcoin transaction, and links the maker's known on-chain address patterns back to specific RFQs.

*Mitigation:* the broker never sees the PSBT or the maker's source UTXOs in any protocol message, so this is a probabilistic, not deterministic, leak. Makers can reduce it further by introducing settlement jitter, batching settlements, rotating broadcast paths, or operating their own broker. The protocol does not eliminate this leak — takers and makers should treat any broker as a passive observer of timing.

### Broker selectively dropping or reordering messages

A broker withholds quotes from one maker, reorders quote arrivals, or selectively delivers acceptances to favor a colluding maker.

*Mitigation:* takers see all returned quotes and select among them; broker reordering does not change which quotes are valid. Multi-broker support means a taker can route the same RFQ through more than one broker and compare. This is a degraded-service threat, not a fund-loss threat.

## Out of Scope

- Post-quantum cryptographic threats against Bitcoin or RGB.
- Operator-level threats against an individual maker's keys, hardware, or hosting.
- Network-level traffic analysis beyond the protocol's stated privacy properties (see [architecture.md](architecture.md)).
- Threats specific to multi-maker routing and partial fills (those will be addressed when that work is funded; see [milestones.md](milestones.md) Future Work).
