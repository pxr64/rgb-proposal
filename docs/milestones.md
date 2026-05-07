# Milestones

Scope and timeline for RGB RFQ Network.

**Funded scope: Milestone 1 + Milestone 2 + Milestone 3** (~24–30 weeks, 6–7 months). Multi-maker routing, Lightning RGB settlement, and additional asset schemas are documented under [Future Work](#future-work) and are intentionally out of the funded scope. Budget is shared with prospective funders on request.

Each milestone lists goals, deliverables, and quantified success criteria. Timelines are estimates from milestone start. M2 and M3 work can be partially parallelized once the M1 protocol surface stabilizes.

## Milestone 1 — RFQ Infrastructure

Foundational protocol and broker. The output of this milestone is a runnable RFQ network with mock settlement, sufficient to integration-test against without RGB on-chain dependencies.

### Goals

- Define the RFQ protocol surface (request, quote, accept, reject, expire)
- Implement the broker as a coordinator with no persistent state (ephemeral routing state only)
- Provide a client SDK for wallets and applications
- Provide a maker node skeleton with pluggable quoting and inventory

### Deliverables

- RFQ protocol specification (versioned message schemas)
- Broker reference implementation (Rust)
- Client SDK (Rust crate)
- Maker node skeleton with mock inventory backend
- Conformance test suite covering protocol round-trips, quote expiry, and reservation lifecycle

### Success Criteria

- Conformance suite covers ≥95% of protocol message paths and lifecycle transitions
- Broker sustains ≥100 RFQs/sec on a single core under synthetic load with bounded memory
- Reservation lifecycle correctness verified by property tests covering quote, expire, reject, and accept paths
- Mock-settlement RFQ round-trip completes in <500ms p50 on local loopback

### Estimated Timeline

8–10 weeks.

## Milestone 2 — RGB Settlement Integration

Replace the mock settlement path with real RGB20 settlement on top of rgb-lib v0.11. The output is end-to-end transfers from taker wallet to maker, anchored on Bitcoin and validated client-side on signet.

### Goals

- Integrate rgb-lib v0.11 stash and state-transition APIs into the maker node
- Implement PSBT construction over real UTXO inventory
- Produce and deliver consignments to takers
- Implement reservation-aware inventory backed by a real Bitcoin wallet
- Validate end-to-end transfers on signet

### Deliverables

- RGB integration crate (state-transition assembly, consignment build) bound to rgb-lib v0.11
- PSBT construction module bound to maker UTXO inventory
- Consignment delivery transport (initial: direct HTTP; pluggable interface for future transports)
- Signet end-to-end test harness
- Operator documentation for running a maker node

### Success Criteria

- End-to-end RGB20 signet transfer completes in <30s p50, <90s p99 from quote acceptance to consignment validation
- ≥10 consecutive transfers without state corruption, double-reservation, or stuck consignments
- Consignment delivery retry succeeds across at least three simulated transport-failure scenarios (timeout, disconnect, partial-write)
- Maker node passes rgb-lib stash-validation invariants after every settlement
- Consignment delivery completes before PSBT broadcast on the happy path; abort-before-broadcast path verified by negative test

### Estimated Timeline

10–12 weeks.

## Milestone 3 — Wallet/WASM Support

Make the client SDK embeddable in browser wallets and other constrained environments. The output is a WASM build of the client and a documented integration surface for wallet vendors.

### Goals

- Compile the client SDK to WASM with no native-only dependencies
- Define a wallet integration boundary (key access, invoice generation, consignment ingestion)
- Provide TypeScript bindings and example browser integration
- Engage with at least one third-party RGB wallet team for compatibility validation

### Deliverables

- WASM build of the client SDK
- TypeScript bindings package (npm-publishable)
- Wallet integration specification
- Reference browser integration (minimal demo wallet)
- Compatibility report from third-party wallet engagement

### Success Criteria

- WASM build is <2MB gzipped
- Protocol round-trip latency overhead vs native client ≤50ms (excluding settlement)
- TypeScript bindings cover the full public client API and ship with runnable browser examples
- Compatibility validated end-to-end on signet with at least one third-party RGB wallet
- No native-only dependencies in the WASM build (verified by dependency audit)

### Estimated Timeline

6–8 weeks.

## Future Work

The following items are out of scope for the funded grant. They are documented here to make the long-term shape of the project explicit and to clarify what is intentionally deferred.

### Advanced Routing

Multi-maker fanout, reservation-aware quoting across makers, and partial fills. This becomes meaningful once a critical mass of maker nodes operates on the network and is best designed with operator feedback from M1–M3 in hand.

Anticipated scope:

- Multi-maker fanout with deterministic quote aggregation
- Reservation-aware quoting that accounts for partial fills
- Partial-fill settlement protocol (multi-PSBT, multi-consignment)
- Routing strategy interface for broker operators (best-price, lowest-latency, split-fill)
- Broker↔maker reservation hint protocol for multi-maker fanout (broker-held fanout state only; no cross-maker reservation sync)

### Lightning RGB Settlement

Maker-side adapter for [rgb-lightning-node](https://github.com/RGB-Tools/rgb-lightning-node), allowing makers to settle RGB transfers over Lightning channels in addition to on-chain. Requires further maturation of rgb-lightning-node and Lightning RGB channel semantics. The maker node's settlement engine is intentionally a trait boundary so that a Lightning adapter is an addition, not a rewrite.

### Additional Asset Schemas

Initial scope is RGB20 (fungible). RGB21 (collectibles) and RGB25 (CFA) support can be added once the RGB20 settlement path is hardened.
