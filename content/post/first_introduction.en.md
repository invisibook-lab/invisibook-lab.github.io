---
layout:     post
title:      "Invisibook: A Privacy-Preserving Decentralized Order Book"
date:       2025-12-25
author:     "Invisibook Lab"
tags:       ["blockchain", "decentralization", "privacy", "order book", "DeFi"]
categories: ["Tech"]
---

### Introduction: Why We Need a Decentralized Privacy Order Book

Order book trading is the most mature and scalable form of trading in modern financial systems, and it serves as the core mechanism for price discovery and liquidity formation. However, in an era of digitization, on-chain execution, and high-frequency trading, order books reveal a long-overlooked structural contradiction: **a trading system cannot simultaneously achieve censorship resistance, privacy protection, and verifiable execution.**

### Decentralized Order Books: Censorship-Resistant and Verifiable, but Lacking Privacy

Decentralized trading systems deploy order submission, sequencing, and matching rules on open protocols, eliminating dependence on centralized intermediaries and endowing trade execution with inherent censorship resistance. As long as a user can interact with the network, their orders cannot be arbitrarily rejected, frozen, or selectively delayed by any single entity.

More importantly, on-chain trading is not only censorship-resistant but also **verifiable**. Matching rules, execution sequences, and settlement outcomes are transparent to all participants, and anyone can independently verify after the fact whether trades were executed according to the established rules. This verifiability is unachievable in traditional exchanges and stands as one of the most fundamental institutional advantages of decentralized trading.

However, existing decentralized order books almost without exception come at the cost of **fully transparent order data**. Price, quantity, and timing information of orders are broadcast in real time to all observers, turning trading activity into a signal source that can be continuously parsed and exploited. This transparency is far from neutral -- at the market microstructure level, it triggers a series of systemic risks.

### Systemic Risks Arising from Transparent Order Books

In a fully transparent order book, order quantities directly expose a trader's capital scale, execution urgency, and remaining trading intent. This places genuine trading demand at a structural disadvantage and gives rise to the following typical behaviors:

**Premature Amplification of Price Impact**

Once the market identifies a potentially large or persistent order, liquidity providers adjust their quoting structure before execution occurs, raising ask prices or lowering bid prices to front-run the anticipated impact. As a result, slippage is amplified before execution even takes place, and genuine traders bear systematically higher execution costs.

**Toxic Flow Hunting and Adverse Selection**

Informationally advantaged participants can identify passive or informationally disadvantaged order flow through changes in order book depth, treating such flow as "toxic orders." Through front-running, counter-positioning, or short-term hedging, risk is transferred to genuine demand participants while profits are structurally absorbed. This behavior is rational in a transparent environment and will, over the long term, erode the willingness of genuine liquidity to participate.

**Liquidity Withdrawal and Depth Failure**

Once trading intent is identified, other liquidity collectively withdraws within an extremely short time frame, causing apparent depth to fail at critical moments. Liquidity that appears sufficient during calm market conditions rapidly evaporates when genuine execution demand arises, creating endogenous instability.

In on-chain environments, these risks are further amplified. The globally permissionless, real-time observability, block ordering rights, and MEV mechanisms make the above behaviors not only feasible but capable of being institutionalized and automated. **Without privacy protection, decentralization may actually be more detrimental to genuine traders than traditional exchanges.**

### Centralized Privacy Trading: Privacy Built on Trust and Censorship

Centralized exchanges provide privacy protection for certain trades through internal matching and privatized execution, but this privacy is **asymmetric and revocable**. Orders remain fully transparent to the exchange itself, its infrastructure operators, and potentially affiliated institutions.

This means:

- Exchanges constitute natural censorship points, capable of freezing, rejecting, or selectively servicing orders;
- Private trading relies on trust in a single entity and cannot be independently verified by external parties;
- Privacy is not a right guaranteed by the protocol, but a privilege granted by the platform.

Under this structure, traders must compromise between execution security and institutional credibility.

### Decentralized Privacy Order Book: Resolving the Structural Contradiction

This project is proposed precisely to resolve the long-standing structural contradiction described above. We believe that a truly sound trading system must simultaneously satisfy:

- **Decentralized execution, to eliminate censorship points;**
- **Native privacy protection, to prevent trading intent from being systematically exploited;**
- **Verifiable matching and settlement rules, to ensure market fairness.**

A decentralized privacy order book is not a simple improvement upon existing trading models, but a fundamental rebalancing at the institutional level of the relationship among transparency, privacy, and verifiability. Its goal is not to hide the market itself, but to prevent transparency from being weaponized against genuine traders.

## Invisibook -- A Privacy-Preserving Decentralized Order Book

### Overall Workflow

1. Users hold a certain amount of cryptocurrency on the chain, with balances recorded in ciphertext form.
2. Users submit encrypted orders on-chain along with a ZK proof:
    - Only the order quantity is encrypted; the quoted price is in plaintext.
    - The ZK proof demonstrates that the user has sufficient balance and fees to place the order.
3. On-chain matching is performed based on the quoted price according to the following criteria (since order quantities are in ciphertext, matching may be imprecise):
    - Block height is used as the basis for time ordering; price matching is prioritized among orders in lower-numbered blocks.
    - If orders are in the same block with the same quoted price, they are matched in descending order of gas fee.
    - Matching is based on unit price between limit order <-> limit order and market order <-> limit order pairs.
4. After a successful match, the orders are locked on-chain, and both parties are required to sign the matching result. Matched orders cannot be withdrawn and must await settlement.
5. Users view the on-chain matching results and perform peer-to-peer settlement off-chain with the matched counterparty using MPC, generating a ZK proof that attests to the following:
   - The buyer's credited amount / the seller's debited amount = the order settlement amount.
   - The remaining order quantity = the original order quantity - the settled order quantity.
6. The ZK proof is uploaded to the chain for verification. Upon successful verification, the corresponding order and balance states are updated (all in ciphertext form).

### MPC Settlement

MPC under a malicious-security model for peer-to-peer order settlement.

Secret-sharing-based MPC:
https://github.com/renegade-fi/ark-mpc
https://github.com/data61/MP-SPDZ


Garbled-circuit-based MPC references:
https://eprint.iacr.org/2017/030.pdf
https://github.com/emp-toolkit/emp-ag2pc

### Collaborative ZK Proofs

Papers:
1. https://eprint.iacr.org/2025/1388
2. https://eprint.iacr.org/2024/143

### Anti-MEV Protocol

When limit orders are submitted on-chain, the quoted price is in plaintext, which creates a risk of MEV attacks. Therefore, we adopt the MEVless protocol for MEV resistance:

- EIP-8099: [https://github.com/ethereum/EIPs/pull/10855](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8099.md)
- https://github.com/ethereum/RIPs/pull/75

## Consensus Protocol

### Proof of Buying

In the first phase, we will operate as a Layer 2 AppChain on CKB, so our consensus protocol needs to be grounded in CKB's security:

1. **Admission Consensus**: All CKB holders are eligible to participate in mining.
2. **Block Production Consensus**: Miners must pay a certain amount of CKB to a designated address on CKB Layer 1 and locally compute a random value using a VDF algorithm, which is then broadcast to the network. Each node computes `VDF_output * CKB_payment_amount`, and the node with the highest value is selected as the block producer. The VDF input is the blockHash of the previous block.
3. **Finality Consensus**: Among all forks, the chain with the highest cumulative sum of `(VDF_output * CKB_payment_amount)` is selected as the canonical chain.
4. **Exit Consensus**: A participant can exit simply by ceasing payment or stopping VDF computation.
