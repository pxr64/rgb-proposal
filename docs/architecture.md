# Architecture

This document describes the architecture of RGB RFQ Network: the RGB-native constraints it is designed around, the maker node internals, and the rationale for choosing an RFQ model over an orderbook at this stage.

## RGB-Native Constraints

RGB is a client-side validation protocol. Asset state is not held on-chain or in a global ledger; it lives in per-user stashes and is validated by recipients against the consignment they receive. Any settlement layer for RGB must respect this model rather than work around it.

### UTXO-Bound Liquidity

RGB20 balances are bound to specific Bitcoin UTXOs. A maker holding `N` units of an asset does not hold a fungible pool — it holds a set of UTXOs, each carrying an allocation. Quoting and settlement must reason at the UTXO level:

- a quote implicitly commits a specific UTXO (or set of UTXOs) to a potential trade
- two concurrent quotes against overlapping UTXOs cannot both settle
- inventory state is the set of `(outpoint, asset_id, allocation)` tuples, not a scalar balance

This forces the maker node to track reservations per UTXO and to release them on quote expiry or rejection.

### Blinded UTXO Invoices

The receiver generates an RGB invoice that contains a blinded seal — a commitment to a UTXO that the sender cannot see. The sender constructs a PSBT and an RGB state transition that pays into that blinded seal without learning which UTXO it refers to.

The broker never sees an unblinded UTXO. The taker's privacy boundary is preserved end-to-end: the maker only learns the blinded seal at settlement time, and the broker only relays opaque invoice payloads.

### Consignments and Client-Side Validation

A successful RGB transfer produces two artifacts:

- a Bitcoin transaction (the PSBT, once signed and broadcast) that anchors the state transition
- a consignment, an off-chain bundle containing the state transition history the receiver needs to validate the new allocation

The receiver validates the consignment against its own RGB stash. If validation fails, the receiver does not credit the asset, regardless of what happened on-chain. This means consignment delivery is part of settlement, not a post-settlement step. The maker node is responsible for producing the consignment and ensuring it reaches the taker.

## Maker Node Architecture

A maker node is a self-contained service operated by a liquidity provider. It owns:

- a Bitcoin wallet (UTXOs, signing keys)
- an RGB stash (state transitions, allocations)
- a reservation table (UTXOs held for in-flight quotes)
- a quoting policy (pricing, expiry, size limits)
- a settlement engine (PSBT construction, consignment generation)

```mermaid
flowchart LR
    Broker[Broker]
    subgraph MakerNode[Maker Node]
        direction TB
        Quoter[Quoting Engine]
        Reservations[Reservation Table]
        Inventory[UTXO Inventory + RGB Stash]
        Settlement[Settlement Engine]
    end
    Bitcoin[Bitcoin Network]
    Wallet[Taker Wallet]

    Broker -->|RFQ| Quoter
    Quoter --> Reservations
    Reservations --> Inventory
    Quoter -->|Quote| Broker
    Broker -->|Accept + Invoice| Settlement
    Settlement --> Inventory
    Settlement -->|PSBT| Bitcoin
    Settlement -->|Consignment| Wallet
```

### Reservation-Aware Inventory

When the quoting engine produces a quote, it places a soft reservation on the UTXOs that would be consumed if the quote is accepted. Reservations:

- are scoped to a single quote ID
- expire when the quote expires
- are released on explicit rejection or timeout
- block other quotes from committing to the same UTXOs

This prevents the common failure mode where a maker hands out two quotes against the same inventory and can only honor one. Reservations are local state — they are not synchronized across maker nodes, since each maker controls its own UTXO set.

### Settlement Engine

On quote acceptance, the settlement engine:

1. consumes the reservation
2. parses the RGB invoice (blinded seal, asset ID, amount)
3. constructs a PSBT spending the reserved UTXOs
4. builds the RGB state transition assigning the allocation to the blinded seal
5. produces the consignment
6. delivers the consignment to the taker
7. signs and broadcasts the PSBT

Consignment delivery precedes broadcast. If delivery fails, the maker can abort before committing the Bitcoin transaction, avoiding the case where the chain advances but the receiver cannot validate.

## Why RFQ Fits RGB Better Than Orderbooks Initially

A central limit orderbook assumes resting liquidity, fungible inventory, and a matching engine that can pair any bid with any ask. None of these hold cleanly for RGB:

- **No resting liquidity surface.** An order on a public book would need to commit a UTXO publicly, leaking which output is being offered and burning the maker's privacy. RFQ keeps quoting private and pairwise.
- **Inventory is not fungible.** Two UTXOs holding the same asset are not interchangeable from the maker's perspective — they have different sizes, different histories, and different reservation states. Quoting per-RFQ lets the maker pick the right UTXO for the request.
- **Settlement is interactive.** RGB settlement requires invoice exchange, PSBT construction, and consignment delivery between specific counterparties. An orderbook abstracts counterparties away; RFQ keeps the pair explicit, which matches the protocol shape.
- **No partial-fill semantics by default.** Splitting an RGB allocation across multiple makers requires coordinated state transitions or sequential fills. RFQ makes this an explicit routing decision rather than an implicit matching outcome.

An orderbook may become viable later — for example, once routing across multiple makers and partial fills are standardized (Milestone 4). RFQ is the lower-risk starting point that respects RGB's constraints today.
