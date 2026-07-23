---
layout:     post
title:      "Invisibook Protocol"
date:       2026-04-25
author:     "Invisibook Lab"
tags:       ["MPC", "zk", "privacy"]
categories: ["Tech"]
---

Privacy-Preserving Cross-Chain Order Book Protocol

*Technical White Paper — Protocol Flow Details*

Version 1.0  |  2026

# **1  Overview**

Invisibook is a **privacy-preserving cross-chain order book protocol** designed for decentralized trading scenarios. It addresses a series of problems arising from the transparency of trade amounts in current DeFi order book models, including front-running, liquidity manipulation, other MEV attacks, and trading strategy leakage.

The core design philosophy of Invisibook is: **prices are public, amounts are private**. The on-chain order book exposes only price information for the matching engine to perform pairing, while the specific order amounts are stored on-chain as Poseidon hash commitments (Poseidon is tentatively chosen; other hash functions may be considered later — the key selection criterion is which hash achieves the best combined performance in MPC circuits and ZK proofs). No third party can learn the true size of any order. When two orders are successfully matched, the buyer and seller complete settlement through an off-chain peer-to-peer privacy settlement protocol, during which both parties' order amounts remain confidential to outsiders at all times.

The protocol's security is built upon three major cryptographic primitives: **the Poseidon hash function** for generating efficient arithmetic-friendly commitments, **zero-knowledge proofs (ZK Proofs)** for verifying various constraints without revealing private information, and **SPDZ-based malicious-secure MPC combined with collaborative zero-knowledge proofs (co-ZK)** for performing secure numerical comparison and generating verifiable proofs between two parties without revealing private inputs.

The protocol is divided into four phases:

* **Phase One: Cross-Chain Deposit** — Users deposit assets from external chains (such as Ethereum, Solana, and other L1/L2 networks) via the corresponding cross-chain bridge. On the Invisibook chain, the deposited funds are recorded as private UTXO notes (Zcash-style), with note commitments appended to an on-chain Merkle tree, and a ZK Proof is submitted to prove fund ownership.

* **Phase Two: Private Order Submission** — Users create orders by consuming private UTXOs (via nullifiers) as collateral. Prices are publicly disclosed in plaintext, amounts are hidden via Poseidon commitments, and a ZK Proof proves that the consumed UTXOs fully cover the order value.

* **Phase Three: On-Chain Transparent Matching** — The matching engine executes a price-priority matching algorithm based on plaintext prices, pairing buyers and sellers.

* **Phase Four: Off-Chain Privacy Settlement** — Matched orders are immediately locked on-chain and enter the settlement process. The two parties run a SPDZ-based MPC protocol that jointly produces secret shares of both the comparison result and a collaborative ZK proof (co-ZK). Each party submits their shares on-chain, where the chain assembles the comparison result and the proof. The smaller-order party then sends the plaintext to the larger-order party to complete the difference calculation, UTXO creation, and state update.

![Protocol flow diagram][image1]

*Figure 1: Invisibook Protocol Overall Flow and System Architecture*

# **2  Notation and Basic Definitions**

To maintain consistency in subsequent descriptions, this section defines the core symbols used in the protocol.

| Symbol | Type | Description |
| :---- | :---- | :---- |
| addr\_i | Address | User i's account address on the Invisibook chain |
| note | Tuple | A private UTXO note of the form (addr, v, r), where v is the value and r is the randomness |
| cm | Commitment | Note commitment: cm = Poseidon(addr, v, r), stored in an append-only Merkle tree |
| nf | Nullifier | A unique tag derived from a note, published on-chain to "spend" the note without revealing which note is consumed |
| amt (or q) | uint64 | The actual plaintext order amount (known only to the user) |
| r | Random number | 256-bit random blinding factor |
| C (or cm\_q) | Commitment | Poseidon hash commitment of the order amount: C = Poseidon(amt, r) |
| price (or p) | uint64 | Order price (plaintext, publicly visible on-chain) |
| f | uint64 | Order fee (plaintext), paid by the order creator |
| π | Proof | Zero-knowledge proof (ZK Proof) |

The Poseidon commitment is computed as follows:

C \= Poseidon( amt  ,  r )

where , denotes concatenation. Poseidon is a hash function highly optimized for ZK circuits, with arithmetic operations over finite fields far more efficient than SHA-256 or Keccak-256, making it widely used in ZKP systems. The blinding factor r ensures that identical order amounts do not produce identical commitment values, thereby preventing on-chain observers from inferring order information through commitment collisions.

# **3  Phase One: Cross-Chain Deposit and Ownership Proof**

## **3.1  Flow Description**

The user first initiates a cross-chain transfer transaction on the external source chain (Source Chain), depositing native assets or tokens into the **cross-chain bridge** deployed by Invisibook on that chain. After the bridge locks the user's assets, the deposit information is synchronized to the Invisibook chain via cross-chain message passing mechanisms (such as state root anchoring, relayer networks, or light client verification). The source chain can be any supported L1 or L2 network, such as Ethereum, Solana, Arbitrum, etc.

On the Invisibook chain, user funds are stored as **private UTXO notes** in a Zcash-style model. Each note is a tuple (addr, v, r), where addr is the owner's address, v is the value, and r is a random blinding factor. The note commitment cm = Poseidon(addr, v, r) is appended to an on-chain append-only Merkle tree. This design ensures that even though the Merkle tree is public to all validators, no third party can determine the owner or value of any note.

After completing the deposit, the user must submit a **zero-knowledge proof π\_deposit** on the Invisibook chain, proving that they are indeed the initiator and legitimate holder of the deposit transaction on the source chain, and that the corresponding note commitment has been correctly computed and inserted into the Merkle tree.

## **3.2  Algebraic Description**

Suppose user Alice deposits amount d into the cross-chain bridge. A new private UTXO note note\_new = (addr\_Alice, d, r) is created, with commitment cm\_new = Poseidon(addr\_Alice, d, r). The following must hold:
```
π_deposit: {

  cm_new = Poseidon(addr_Alice, d, r)

  ∧  deposit_tx ∈ source_chain_tx_root

  ∧  sender(deposit_tx) = addr_Alice

}
```

The first constraint ensures the note commitment is correctly formed, the second ensures the deposit transaction was indeed confirmed by the source chain blockchain, and the third ensures the binding between the transaction initiator and the Invisibook account. After verification, cm\_new is appended to the on-chain Merkle tree.

# **4  Phase Two: Private Order Submission**

## **4.1  Flow Description**

When a user wishes to place an order on the Invisibook order book, they need to construct a **private order**. An order is a special on-chain object with the following tuple structure:

Order = (oid, τ, d, p, cm\_q, pk, f)

where oid is the unique order ID, τ ∈ {BUY, SELL} is the order direction, d is the trading pair identifier, p is the plaintext price, cm\_q = Poseidon(q, r\_q) is the commitment to the order quantity q, pk is the creator's public key, and f is the plaintext fee.

To create an order, the user must **consume one or more private UTXO notes** (via publishing their nullifiers on-chain) whose total value covers p · q + f. This mechanism ensures full collateralization: the consumed notes serve as locked collateral for the order.

The user must generate and submit a zero-knowledge proof π\_order that satisfies the following constraints:

* Each consumed note exists in the on-chain Merkle tree (Merkle membership proof)

* Each nullifier is correctly derived from the corresponding note

* The total value of consumed notes ≥ p · q + f (full collateralization)

* Commitment format correctness: cm\_q = Poseidon(q, r\_q)

* Amount positivity: q > 0

## **4.2  Algebraic Description**

Order = (oid, τ, d, p, cm\_q, pk, f)

where τ ∈ {BUY, SELL} indicates the order direction. The complete ZK Proof statement is as follows:
```
π_order: {

  ∃ q, r_q, {note_i, path_i, nf_i}  s.t.

  ∀i: MerkleVerify(root, cm_i, path_i) = true

  ∧  ∀i: nf_i = DeriveNullifier(note_i)

  ∧  Σ note_i.v ≥ p · q + f

  ∧  cm_q = Poseidon(q, r_q)

  ∧  q > 0

}
```
This proof ensures that users cannot place orders without sufficient collateral (the consumed UTXOs must fully cover the order value plus fee), nor submit illegal zero-amount or negative-amount orders. Since the proof is verified on-chain, any order that fails to satisfy the above constraints will be rejected. Upon acceptance, the nullifiers of the consumed notes are recorded on-chain, preventing double-spending.

## **4.3  Security Analysis**

By hiding amounts within Poseidon commitments, Invisibook achieves **order amount privacy**. The on-chain matching engine, validators, and other traders can only see the price and direction of an order, and cannot determine its specific quantity. This design effectively defends against the following attack vectors:

* **Front-running:** Attackers cannot front-run by observing large orders.

* **Liquidity Manipulation:** When attackers see large market orders, they may preemptively remove low-priced limit orders and re-place them at higher prices, forcing victims to execute at worse prices. Since amounts are invisible, attackers cannot judge order size, and thus cannot determine whether manipulation would be profitable.

* **Strategy Inference:** Market makers or competitors cannot infer users' trading strategies and position information from order quantities.

# **5  Phase Three: On-Chain Order Matching**

## **5.1  Matching Mechanism**

The matching engine on the Invisibook chain operates on the core principle of **Price Priority**. Since all order prices are publicly disclosed in plaintext, the matching engine can execute standard order matching logic normally.

Specifically, the matching engine maintains a standard two-sided order book:

* **Bid Book:** Sorted by price from high to low.

* **Ask Book:** Sorted by price from low to high.

When a new buy order price ≥ the current best ask price (or vice versa), the matching engine marks these two orders as matched and triggers the subsequent off-chain settlement process.

## **5.2  Matching Priority Rules**

When multiple orders have prices within the executable range, the matching engine sorts them according to the following **four-level priority** to determine the matching order:

* **First Priority — Best Price (Price Priority):** Among buy orders, the highest bid is matched first; among sell orders, the lowest ask is matched first. This is the fundamental principle of all order books, ensuring the market's price discovery function operates normally.

* **Second Priority — Earliest Block Height (Block Height Priority):** When multiple orders have the same price, the block height at which the order was included serves as the time-ordering criterion. Orders in lower-numbered blocks (i.e., earlier on-chain) enjoy higher matching priority. This rule ensures first-come-first-served fairness, preventing latecomers from displacing earlier participants at the same price.

* **Third Priority — Highest Fee (Fee Priority):** If multiple orders have the same price and are within the same block (i.e., same block height), they are sorted by the plaintext fee field f in descending order. Orders paying higher fees are matched first. This rule provides a market-based ordering mechanism within blocks — when users wish to gain higher execution priority at the same price level, they can express this desire by increasing their order fee.

* **Fourth Priority — Intra-Block Transaction Index (Index Priority):** If orders share the same price, block height, and fee, they are ordered by their transaction index within the block in ascending order. This provides a deterministic tiebreaker to ensure a fully defined ordering.

The above priority rules can be formally expressed as:

Priority(order) = ( price, block\_height↑, fee↓, tx\_index↑ )

That is: first sort by price (descending for buy orders, ascending for sell orders), then by block height in ascending order when prices are equal (earlier is prioritized), then by fee in descending order when block heights are also equal (higher fee is prioritized), and finally by intra-block transaction index in ascending order as a deterministic tiebreaker.

**Strictly pairwise matching:** Because order amounts are hidden, the matching engine cannot aggregate multiple orders. Each match pairs exactly one bid with one ask. After a match, both orders are **immediately locked on-chain**: they can no longer be canceled and are forced into the settlement process.

## **5.3  Important Design Trade-off**

Since order amounts are in ciphertext form, the matching engine **cannot know** during the matching phase whether the true quantities of two matched orders are equal. Therefore, Invisibook's matching result is only a **"price match"**, not a "full execution match" in the traditional sense. The actual quantity comparison and difference processing must be completed during the off-chain settlement in Phase Four.

This design represents the most fundamental difference between Invisibook and traditional order book matching engines: traditional engines can complete precise quantity matching and partial fill processing during matching, whereas Invisibook defers quantity-related computation to the off-chain privacy settlement phase. Additionally, matching is strictly pairwise (one bid, one ask) because the chain cannot aggregate multiple hidden-amount orders.

# **6  Phase Four: Off-Chain Privacy Settlement**

Phase Four is the most complex and critical step in the Invisibook protocol. After the matching engine pairs two orders, both orders are **immediately locked on-chain** and enter the settlement process. The order holders (hereafter referred to as Alice and Bob) must complete settlement through an off-chain peer-to-peer privacy protocol.

![][image2]

*Figure 2: Phase Four — Detailed Off-Chain Privacy Settlement Flow*

## **6.1  Secure Amount Comparison: Malicious-Secure Secret-Sharing MPC (SPDZ)**

The first step of settlement is to determine which party has the smaller order amount. However, since both parties' order amounts are private, neither party can directly send their plaintext amount to the other (doing so would leak privacy during the comparison phase).

Invisibook employs a malicious-secure MPC protocol based on **SPDZ** to complete commitment verification and numerical comparison without revealing either party's input.

**SPDZ ("Speedz")** is a malicious-secure multi-party computation framework based on additive secret sharing and information-theoretic MAC (IT-MAC). In Invisibook's settlement scenario, Alice and Bob each hold secret inputs (order amount amt and blinding factor r), and both parties run the SPDZ protocol to execute the following logic:

1. **Commitment verification**: Compute Poseidon(amt\_A, r\_A) under secret sharing and verify it equals the on-chain commitment C\_A; similarly verify Poseidon(amt\_B, r\_B) == C\_B.
2. **Numerical comparison**: Compute the comparison result b = (amt\_A ≤ amt\_B) under secret sharing.
3. **IT-MAC malicious security guarantee**: SPDZ's IT-MAC mechanism ensures that if either party attempts to tamper with their input or intermediate computation, it will be detected by the other party, and the protocol will abort.

**Integrated design with collaborative ZK proofs (co-ZK)**: Commitment verification and numerical comparison are executed within the MPC protocol as a single atomic operation. Neither party needs to publicly reveal their commitment opening. The protocol internally computes the Poseidon hash using secret shares and compares it against the on-chain commitment. Furthermore, the MPC protocol jointly generates a **collaborative zero-knowledge proof (co-ZK) π\_cmp** that attests to the correctness of the comparison — neither party alone can construct this proof, and neither party learns the other's private input during the process.

**Share output**: The MPC protocol produces two types of secret shares:

* **Comparison result shares**: Alice receives share\_b\_A, Bob receives share\_b\_B, satisfying share\_b\_A + share\_b\_B ≡ b (mod p)
* **Proof shares**: Alice receives share\_π\_A, Bob receives share\_π\_B, which can be assembled on-chain into the complete proof π\_cmp

Neither the comparison result nor the proof is directly revealed to either party. Each party submits both their comparison share and proof share to the chain, and the on-chain logic completes the assembly.

### **6.1.1  On-Chain Share Synthesis**

After MPC completion, both parties must submit their respective shares on-chain within the prescribed block deadline (10 block times). The on-chain logic executes the following operations:

* Assemble the comparison result: compute share\_b\_A + share\_b\_B mod p to obtain b
* Assemble the collaborative proof: combine share\_π\_A and share\_π\_B to reconstruct π\_cmp
* Verify π\_cmp on-chain to confirm the correctness of the comparison
* b = 1 indicates amt\_A ≤ amt\_B (Alice is the smaller-order party); b = 0 indicates amt\_A > amt\_B (Bob is the smaller-order party)
* After the comparison result and proof are publicly verified on-chain, the subsequent settlement process is triggered

On-chain synthesis ensures that the finality of the comparison result does not depend on any single party, but is cryptographically guaranteed through on-chain proof verification.

### **6.1.2  Timeout Freeze Mechanism**

The protocol distinguishes two failure scenarios:

**Case 1 — MPC Abort:** If the MPC protocol itself aborts (e.g., due to detected cheating via IT-MAC, network failure, or either party going offline during computation), **no penalty is imposed**. Both orders are unlocked and returned to the order book for re-matching. This is because an abort may result from legitimate network issues rather than intentional misbehavior.

**Case 2 — Withholding Shares:** If the MPC completes successfully but one party fails to upload their shares on-chain within the deadline (10 block times):

* The delaying party's order is frozen for 72 hours
* The party that already uploaded has their order released and can re-enter matching

The penalty is a **freeze, not a slash**, because failure to submit may also result from network faults rather than malicious intent. This mechanism ensures both parties are incentivized to promptly complete on-chain share submission, guaranteeing the liveness of the settlement process.

## **6.2  Smaller-Order Party Settlement (Alice as Example)**

Assume the comparison result is amt\_A <= amt\_B, meaning Alice holds the smaller order. Alice must then perform the following operations:

### **6.2.1  Peer-to-Peer Plaintext Transmission**

Alice sends her plaintext order amount amt\_A and the corresponding 256-bit random blinding factor r\_A to Bob through secure peer-to-peer communication. Bob can then locally verify:

Poseidon(amt\_A , r\_A) == C\_A

where C\_A is the order commitment Alice previously published on-chain. If verification passes, Bob can be confident that the plaintext data Alice sent is consistent with the on-chain commitment, with no deception.

### **6.2.2  On-Chain State Update**

Alice submits the following updates to the chain:

* Her order is **marked as destroyed** on the order book (fully filled)

* A new **private UTXO note** is created and paid to Bob (the counterparty), representing the settled asset transfer of amt\_A

* Alice provides a zero-knowledge proof π\_A proving the consistency of the settlement (the UTXO amount matches the committed order quantity)

This result is confirmed by the on-chain share synthesis and proof verification (see 6.1.1).

## **6.3  Larger-Order Party Settlement (Bob as Example)**

After Bob receives amt\_A and r\_A from Alice, he first locally verifies the commitment consistency. Upon successful verification, Bob locally computes the new remaining order quantity:

amt\_B' = amt\_B - amt\_A

Bob then selects a new random blinding factor r\_B' and computes the new order commitment:

C\_B' = Poseidon(amt\_B' , r\_B')

Bob submits the following updates to the chain:

* His original order is **marked as destroyed** on the order book

* A **new order o\_B'** is created with the updated commitment C\_B' (the residual order continues as an open order on the book)

* A new **private UTXO note** is created and paid to Alice (the counterparty), representing the settled asset transfer of amt\_A

* Attach a zero-knowledge proof π\_B that simultaneously proves the following three statements:

```
π_B: {

  (1)  amt_B' = amt_B - amt_A  ∧  amt_B' ≥ 0

  (2)  Poseidon(amt_A , r_A) = C_A

  (3)  The new UTXO amount is consistent with the fill quantity amt_A

}
```

Statement (1) proves that the residual order amount is the correct result of subtracting the smaller order amount from the old order amount, and that the residual is non-negative (ensuring Bob did not tamper with the calculation). Statement (2) proves that the amt\_A used by Bob is indeed consistent with Alice's previously committed order ciphertext C\_A on-chain (preventing Bob from fabricating the smaller order quantity to gain illegitimate benefits). Statement (3) ensures the new UTXO paid to the counterparty is consistent with the actual fill.

**Equal quantities case:** When amt\_A = amt\_B, both orders are fully filled and marked as destroyed. Two new private UTXO notes are created — one paid to each party — and no residual order remains.

## **6.4  Challenge**

After on-chain share synthesis is complete and the comparison result has been publicly confirmed, the smaller-order party must send their plaintext amount amt and blinding factor r to the larger-order party to complete settlement. If the smaller-order party (Alice as example) refuses to send, the following challenge process is triggered:

If Alice has not sent the plaintext after more than 5 block times, Bob can initiate a forced request on-chain, attaching his public key Pub\_B, to compel Alice to encrypt her amt\_A and r\_A with Pub\_B and submit Pub\_B(amt\_A, r\_A) on-chain. After Alice sees the forced request, she must send Pub\_B(amt\_A, r\_A) to the chain within 10 block times; otherwise, Alice's order will be immediately frozen for 72 hours, and Bob's order can re-enter matching with other encrypted orders.

After Bob sees Pub\_B(amt\_A, r\_A) appear on-chain, he downloads it locally, decrypts it, and proceeds with verification and settlement following the steps in 6.2/6.3.

**Adjudication:** If Alice posts plaintext data that is incorrect (i.e., does not match her on-chain commitment), Bob can submit Alice's claimed (amt\_A, r\_A) to the chain. The on-chain logic checks whether Poseidon(amt\_A, r\_A) = C\_A. If the check fails, Alice's order is frozen and Bob's order is released for re-matching.

# **7  Proof Obligations Summary by Phase**

The following table summarizes the zero-knowledge proofs involved in each phase of the protocol and the core statements they prove:

| Phase | Proof | Core Statement |
| :---- | :---- | :---- |
| Phase One | π\_deposit | Note commitment is correctly formed ∧ deposit transaction exists in source chain ∧ initiator identity binding |
| Phase Two | π\_order | Consumed notes exist in Merkle tree ∧ nullifiers correctly derived ∧ total value ≥ p · q + f ∧ cm\_q = Poseidon(q, r\_q) ∧ q > 0 |
| Phase Four (comparison) | π\_cmp | Collaborative ZK proof (co-ZK) assembled from MPC shares: verifies commitment openings and comparison correctness |
| Phase Four (smaller party) | π\_A | Settlement UTXO is consistent with committed order quantity |
| Phase Four (larger party) | π\_B | amt\_B' = amt\_B - amt\_A ∧ amt\_B' ≥ 0 ∧ Poseidon(amt\_A, r\_A) = C\_A ∧ UTXO amount consistent with fill |

# **8  Security Properties Summary**

The Invisibook protocol provides the following security guarantees at different levels:

| Security Property | Mechanism | Description |
| :---- | :---- | :---- |
| **Order Amount Privacy** | Poseidon commitment \+ ZKP | Only commitment values are stored on-chain; no third party can recover the true amount |
| **Account Balance Privacy** | UTXO note commitments \+ Merkle tree \+ nullifiers | User funds are stored as private UTXO notes; only commitments are visible on-chain, spent via unlinkable nullifiers |
| **Secure Comparison** | SPDZ malicious-secure MPC \+ collaborative ZK proof (co-ZK) \+ on-chain synthesis | Both parties can compare order sizes and produce a verifiable proof without revealing plaintext amounts to each other |
| **Settlement Correctness** | On-chain ZKP verification | Difference calculations and commitment consistency are all enforced through zero-knowledge proofs on-chain |
| **Settlement Liveness** | Timeout freeze mechanism | Delayed share submission results in a 72-hour freeze for the delaying party, ensuring both parties complete settlement promptly |
| **Anti Front-running** | Amount invisibility | Attackers cannot perform front-running or liquidity manipulation MEV attacks by observing order amounts |
| **Cross-Chain Security** | Source chain state root verification \+ deposit proof | Deposit operations are verified through source chain state roots, ensuring the correctness of cross-chain asset mapping |

[image1]: <data:image/png;base64,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>

[image2]: /img/invisibook_private_settlement_flow.png
