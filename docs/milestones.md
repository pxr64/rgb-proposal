# Milestones

Scope and timeline for RGB RFQ Network. Timelines are estimates from the start of each milestone and assume sequential execution; some work in later milestones can be parallelized once Milestone 1 stabilizes.

## Milestone 1 — RFQ Infrastructure

Foundational protocol and broker. The output of this milestone is a runnable RFQ network with mock settlement, sufficient to integration-test against without RGB on-chain dependencies.

### Goals

- Define the RFQ protocol surface (request, quote, accept, reject, expire)
- Implement the broker as a stateless coordinator
- Provide a client SDK for wallets and applications
- Provide a maker node skeleton with pluggable quoting and inventory

### Deliverables

- RFQ protocol specification (versioned message schemas)
- Broker reference implementation (Rust)
- Client SDK (Rust crate)
- Maker node skeleton with mock inventory backend
- Conformance test suite covering protocol round-trips, quote expiry, and reservation lifecycle

### Estimated Timeline

8–10 weeks.

## Milestone 2 — RGB Settlement Integration

Replace the mock settlement path with real RGB20 settlement. The output is end-to-end transfers from taker wallet to maker, anchored on Bitcoin and validated client-side.

### Goals

- Integrate RGB v0.11 stash and state transition APIs into the maker node
- Implement PSBT construction over real UTXO inventory
- Produce and deliver consignments to takers
- Implement reservation-aware inventory backed by a real Bitcoin wallet
- Validate end-to-end transfers on signet

### Deliverables

- RGB integration crate (state transition assembly, consignment build)
- PSBT construction module bound to maker UTXO inventory
- Consignment delivery transport (initial: direct HTTP; future: pluggable)
- Signet end-to-end test harness
- Operator documentation for running a maker node

### Estimated Timeline

10–12 weeks.

## Milestone 3 — Wallet/WASM Support

Make the client SDK embeddable in browser wallets and other constrained environments. The output is a WASM build of the client and a documented integration surface for wallet vendors.

### Goals

- Compile the client SDK to WASM with no native-only dependencies
- Define a wallet integration boundary (key access, invoice generation, consignment ingestion)
- Provide TypeScript bindings and example browser integration
- Validate against at least one third-party RGB wallet

### Deliverables

- WASM build of the client SDK
- TypeScript bindings package
- Wallet integration specification
- Reference browser integration (minimal demo wallet)
- Compatibility report against a third-party wallet

### Estimated Timeline

6–8 weeks.

## Milestone 4 — Advanced Routing

Extend the broker and maker node to support multi-maker fanout, reservation-aware quoting across makers, and partial fills. The output is a routing layer that can split a single RFQ across multiple liquidity providers.

### Goals

- Multi-maker fanout with deterministic quote aggregation
- Reservation-aware quoting that accounts for partial fills
- Partial-fill settlement protocol (multi-PSBT, multi-consignment)
- Routing strategy interface for broker operators

### Deliverables

- Multi-maker routing module in the broker
- Partial-fill protocol extension to the RFQ specification
- Reservation coordination protocol between broker and maker nodes
- Routing strategy reference implementations (best-price, lowest-latency, split-fill)
- Test suite covering split fills, partial expiries, and routing failures

### Estimated Timeline

10–12 weeks.
