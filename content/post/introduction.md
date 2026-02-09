---
layout:     post
title:      "Invisibook: Decentralized  Privacy  order-book on Ethereum"
date:       2026-02-09
author:     "Invisibook Lab"
tags:       ["区块链", "去中心化", "隐私", "订单簿", "DeFi"]
categories: ["Tech"]
---


## **Introduction**

Invisibook is a decentralized privacy order book project. It is built on Ethereum L1 and achieves on-chain matching, off-chain settlement, and on-chain verification through a combination of MPC (Multi-Party Computation) + ZK (Zero-Knowledge proofs).

In Invisibook, the trading instruments and quoted prices are public, but the order quantities on-chain are in ciphertext form. The public and counterparties can only execute trades — they cannot discover the other party’s position size (even using AI or quantitative trading algorithms). This protects traders’ privacy and prevents reverse engineering of positions, front-running (rat trading / insider trading), and various forms of MEV that erode large profits.

## **Flow**

1. The user holds a certain amount of cryptocurrency in the Invisibook contract on Ethereum, recorded in ciphertext form.
2. The user submits a ciphertext/plaintext order on-chain along with a ZK proof:
    - if ciphertext, only the order quantity is encrypted (the quantity is a commitment); the price is in plaintext.
    - The ZK proof demonstrates that the user has sufficient balance and fee allowance to place this order.
3. On-chain matching is performed based on the order prices according to the following rules:
    - Block height serves as the time priority; matching starts from orders in lower block numbers.
    - If orders are in the same block and have the same price, they are matched from highest to lowest gas fee.
    - Matching occurs between limit ↔ limit orders and market ↔ limit orders based on quoted unit prices.
4. After successful matching, the involved orders are locked on-chain. Both trading parties must sign the matching result. Once signed, the orders cannot be canceled and must wait for settlement.
5. （1）If order is plaintext to plaintext, just onchain settle.
（2） If plaintext to cipertext, the cipertext endpoint should update the orders’ states and offer a zk-proof that proves the state-transition is legal.
（3）The client checks the on-chain matching result, then performs peer-to-peer off-chain settlement with the matched counterparty using MPC, and generates a ZK proof.

The MPC settlement process works as follows:

- Uses maliciously-secure MPC for off-chain point-to-point trading; during the process, neither party can learn the other’s exact data.
- Provides a ZK proof showing: the plaintext value of the order quantity commitment = the input value computed by MPC.

The ZK proof certifies:

- Buyer credited / seller debited amount = settlement amount of the order
- Remaining order quantity = old order quantity − settled order quantity
1. Upload the ZK proof to the chain for verification. If verification passes, the corresponding order and balance states are updated (all in ciphertext form).

# **Why Invisibook**

## **Limitations of Traditional Exchanges**

Traditional centralized exchanges (stock exchanges or crypto platforms) rely on a public order book mechanism where all details of buy/sell orders — price, quantity, and direction — are visible to all market participants. This transparency is intended to promote fair competition and price discovery, but in practice it has become a breeding ground for attackers and manipulators, causing significant losses to genuine traders.

First, large orders often suffer severe **price impact**. When an institutional investor submits a large buy or sell order, the public visibility immediately triggers market reactions. For example, a large buy order drives the price up, resulting in worse average execution prices than intended and increasing transaction costs. Conversely, large sells can trigger panic selling and depress prices. Studies show that even medium-sized orders in highly liquid markets can cause 0.5%–2% price deviation — for institutions managing large capital, this translates to millions of dollars in slippage losses.

Second, order transparency easily enables **front-running / reverse positioning**. High-frequency traders (HFT) or insiders can monitor the order book and act ahead. When a large order appears, attackers can frontrun by buying early and selling after the price moves, capturing profit. This includes classic “rat trading” (front-running by insiders) where traders exploit non-public information for personal gain. In crypto markets this problem is especially severe because blockchain transparency amplifies order book visibility, allowing genuine traders to be repeatedly “hunted.”

Additionally, visible order quantities facilitate **spoofing / layering**. Manipulators place large fake orders to create illusions and lure the market in a desired direction (e.g., flooding fake sell orders to push price down, then buying real assets at the depressed level). Regulators such as the U.S. SEC have repeatedly documented such behavior, but the inherent openness of order books makes it extremely difficult to eradicate. Overall, these limitations cause genuine traders to lose substantial profits, reduce market efficiency, and exacerbate inequality — retail investors struggle to compete with institutions.

## **Problems with Existing Decentralized Order Books**

1. **Censorship risk**

    Most decentralized order books still use off-chain matching engines, which creates enormous room for MEV extraction and various forms of censorship. Moreover, because order information is even more fully public and transparent than on traditional exchanges, it provides abundant opportunities for market makers and attackers to engage in front-running, insider trading, etc.

2. **Fake privacy**

    Many projects that claim to offer privacy decentralized order books do not provide real privacy. They rely on trust assumptions — typical examples include centralized matchers or TEEs (Trusted Execution Environments). Users must either trust Intel hardware or trust that the project’s centralized servers will not misbehave or go offline. In contrast, Invisibook solves the problem using pure cryptography with **zero trust assumptions**.


## **What  Invisibook Can Offer Ethereum**

1. **Gradually free the Ethereum ecosystem from dependence on Binance-led centralized exchanges and enable a native, truly private order book exchange on Ethereum**

Although the Ethereum ecosystem currently dominates in TVL, stablecoin settlement, RWA tokenization, etc., the vast majority of high-frequency, professional, and institutional-grade trading volume still heavily relies on centralized exchanges such as Binance, OKX, and Bybit. These platforms effectively control price discovery, liquidity aggregation, and leverage tools, and indirectly control the “real” liquidity windows and slippage benchmarks for a large portion of Ethereum ecosystem assets.

This dependency creates several long-term structural risks:

- **MEV & front-running monetized by centralized platforms** — institutional/whale large orders often go through CEX first, then sweep AMMs like Uniswap on-chain, forcing Ethereum users to accept the worst execution prices.
- **Complete lack of privacy** — on-chain orders/positions are fully transparent, making professional trading strategies easy to copy, snipe, or reverse-engineer.
- **Regulatory & black-swan contagion** — if a major CEX freezes funds, rugs, enforces KYC, or faces regional bans, a large portion of Ethereum’s “apparent liquidity” can vanish instantly (as seen multiple times in 2022–2023).
- **Innovation ceiling locked** — as long as professional trading mainly happens off-chain, Ethereum struggles to birth true on-chain native “Wall Street-grade” financial primitives.

A fully cryptographic privacy order book (using zk-SNARKs, FHE, homomorphic encryption, MPC, or hybrid schemes to enable encrypted order matching, dark-pool-style execution, and partial/zero-knowledge revelation only upon trade) fundamentally changes this landscape:

- Ethereum gains its first truly native privacy venue capable of carrying institutional- and professional-grade trading — eliminating the need to “go to Binance first, then settle back on-chain.”
- Core CEX features such as high-frequency limit orders, iceberg orders, conditional orders, and hidden-intent orders can finally be implemented natively on Ethereum L1 or efficient L2s without leaking intent, being front-run, or sandwiched.
- Price discovery power gradually flows back from CEX to Ethereum on-chain — when enough professional capital decides that “on-chain privacy order book execution quality + privacy > CEX convenience + tail risks,” a structural liquidity migration will occur.
- This is not merely “a more private Uniswap”; it is building Ethereum’s own hybrid “dark pool + public order book” infrastructure, upgrading Ethereum from a “settlement layer + spot AMM playground” to a full-fledged financial primitive layer with complete market microstructure (price formation).
- In the long run, this significantly reduces the “traffic tax” and “trust tax” that CEXs impose on the Bitcoin/Ethereum ecosystem, giving Ethereum genuine trading sovereignty and control over its own destiny.
1. **Increase ETH value capture**

If a privacy order book becomes sufficiently usable, it can significantly strengthen ETH’s position as the core value-capture asset across multiple dimensions:

- **Trading activity directly drives ETH gas consumption**

    Even on L2 or validium, zk-proof generation for the matching engine, state updates, data availability, forced withdrawals/challenges, etc., still consume ETH. Privacy computation is far more gas-intensive than regular AMM swaps — a single complex order may consume 5–30× more gas than a normal transaction. As trading volume scales, ETH burn + staking demand will increase markedly.

- **ETH becomes the benchmark margin & settlement currency for privacy finance**

    Experience from dYdX, Hyperliquid, etc. shows that when professional trading venues use ETH (or stETH) as the primary collateral/settlement currency, a strong positive flywheel forms: more institutions/professionals hold ETH → more ETH locked in privacy order book contracts → further rise in ETH liquidity & demand.

- **Create a new “privacy premium” capture mechanism**

    Native tokenomics can be designed so ETH holders capture a portion of protocol fees generated by the privacy order book through staking, providing privacy compute resources (similar to prover networks), governance participation, etc. This effectively adds a new “privacy finance usage tax” dimension to ETH — something almost nonexistent on Ethereum today.

- **Become a moat-level application that attracts institutional capital**

    Institutions value auditable privacy + composable settlement most. When a privacy order book becomes the preferred execution layer for RWA, institutional DeFi, and crypto hedge funds, large amounts of traditional financial capital will enter via ETH as the bridge (buy ETH → swap to stETH/wETH → provide liquidity/trade in the privacy order book). This creates a new steady source of ETH demand beyond cyclical speculation.

- **Network effects & ecosystem lock-in**

    Once developers, market makers, and quant teams migrate their strategies, risk models, and oracle integrations to this privacy order book infrastructure, switching costs become extremely high — creating a lock-in effect similar to Uniswap’s dominance over AMMs. This lock-in ultimately flows back to ETH, as nearly all privacy DeFi gas, settlement, and collateral must route through ETH.


A privacy order book is not just another application on Ethereum — it is the key infrastructure that upgrades ETH from “fuel” to **the core capital and settlement anchor of the privacy finance era**.

