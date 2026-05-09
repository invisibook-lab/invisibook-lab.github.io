---
layout:     post
title:      "Secret Binary Carry Computation in Secret-Sharing MPC"
date:       2026-03-21
author:     "Invisibook Lab"
tags:       ["MPC", "Edabits", "privacy"]
categories: ["Tech"]
---

---

## Chapter 1: Problem Background and Core Challenges

### 1.1 What Is Secret Addition

In secure multi-party computation (MPC), two parties (Alice and Bob) each hold a secret share of a binary number and wish to compute the sum of the two numbers without revealing the plaintext of either operand. Here, "secret share" refers to Boolean secret sharing: a bit x is split into two shares x₁ and x₂ such that x = x₁ ⊕ x₂, where ⊕ denotes the XOR operation. Alice holds x₁, Bob holds x₂, and each party's share alone is a uniformly random bit from which the true value of x cannot be inferred.

The core difficulty of binary addition lies in the carry: the carry at bit i depends on the carry at bit i-1, forming a chain dependency. If computed naively bit by bit in serial, an n-bit addition requires O(n) rounds of communication, which is unacceptable in MPC. This article will explain in detail how prefix computation compresses this to O(log n) rounds.

### 1.2 Linearity and Nonlinearity in Boolean Secret Sharing

In Boolean secret sharing, XOR is a linear operation that is entirely free and requires no communication. AND is a nonlinear operation, and each evaluation consumes one Beaver triple and one round of communication. This is the foundation for understanding the entire protocol.

**Why XOR is free:** Suppose x = x₁ ⊕ x₂ and y = y₁ ⊕ y₂. To compute shares of z = x ⊕ y, each party simply XORs their two shares locally: Alice computes z₁ = x₁ ⊕ y₁, and Bob computes z₂ = x₂ ⊕ y₂. Verification: z₁ ⊕ z₂ = (x₁ ⊕ y₁) ⊕ (x₂ ⊕ y₂) = (x₁ ⊕ x₂) ⊕ (y₁ ⊕ y₂) = x ⊕ y. The associativity and commutativity of XOR allow each party to operate on their shares independently, and the result is automatically correct.

**Why AND is not free:** Expanding (x₁ ⊕ x₂) ∧ (y₁ ⊕ y₂) = (x₁∧y₁) ⊕ (x₁∧y₂) ⊕ (x₂∧y₁) ⊕ (x₂∧y₂). Among these, Alice can compute x₁∧y₁ locally and Bob can compute x₂∧y₂ locally, but x₁∧y₂ and x₂∧y₁ are "cross terms" where each party holds one factor, requiring communication to compute. This is where Beaver triples and OT come into play.

---

## Chapter 2: Beaver Triples and Secret AND Gates

### 2.1 Definition of Beaver Triples

A Beaver triple is a set of preprocessed random values (u, v, w) satisfying w = u AND v. These values are entirely independent of the actual computation inputs and are generated randomly in advance. The triple is secret-shared between the two parties: Alice holds (u₁, v₁, w₁), Bob holds (u₂, v₂, w₂), satisfying u₁ ⊕ u₂ = u, v₁ ⊕ v₂ = v, w₁ ⊕ w₂ = w.

### 2.2 Computing Secret AND Using Beaver Triples

Suppose Alice and Bob want to compute secret shares of [x] AND [y]. The protocol is as follows:

**Step 1: Compute and reveal masked differences.** Each party locally computes their share of d as d_i = x_i ⊕ u_i, and their share of e as e_i = y_i ⊕ v_i. Then both parties exchange their d and e shares, reconstructing the plaintext d = x ⊕ u and e = y ⊕ v. Here d and e are safe because u and v are completely random, acting as a one-time pad, so d and e reveal no information about x and y.

**Step 2: Locally compute the final shares.** Using the algebraic identity x ∧ y = (d ⊕ u) ∧ (e ⊕ v) = (d∧e) ⊕ (d∧v) ⊕ (u∧e) ⊕ (u∧v), where d∧e is a publicly known constant and u∧v = w is known. Therefore:

```
Alice computes: z₁ = (d AND e) ⊕ (d AND v₁) ⊕ (u₁ AND e) ⊕ w₁
Bob   computes: z₂ = (d AND v₂) ⊕ (u₂ AND e) ⊕ w₂

Note: The constant term d∧e is included by Alice only; otherwise it would cancel out via XOR.
```

Verification: z₁ ⊕ z₂ = (d∧e) ⊕ (d∧v) ⊕ (u∧e) ⊕ w = x ∧ y. The entire process requires only one round of communication (each party sends 2 bits) plus one Beaver triple.

### 2.3 Concrete Numerical Example

Take x = 1, y = 1 (so x AND y = 1) as an example. Suppose the secret sharing is: Alice holds x₁=1, y₁=0, and Bob holds x₂=0, y₂=1. The preprocessed triple is (u=1, v=0, w=0), with Alice holding (u₁=0, v₁=1, w₁=1) and Bob holding (u₂=1, v₂=1, w₂=1).

```
Alice: d₁ = x₁⊕u₁ = 1⊕0 = 1,  e₁ = y₁⊕v₁ = 0⊕1 = 1
Bob:   d₂ = x₂⊕u₂ = 0⊕1 = 1,  e₂ = y₂⊕v₂ = 1⊕1 = 0

Revealed: d = d₁⊕d₂ = 1⊕1 = 0,  e = e₁⊕e₂ = 1⊕0 = 1

Alice: z₁ = (0∧1) ⊕ (0∧1) ⊕ (0∧1) ⊕ 1 = 0⊕0⊕0⊕1 = 1
Bob:   z₂ = (0∧1) ⊕ (1∧1) ⊕ 1 = 0⊕1⊕1 = 0

Verification: z₁ ⊕ z₂ = 1 ⊕ 0 = 1 = x AND y ✓
```

---

## Chapter 3: Generating Beaver Triples with Oblivious Transfer (OT)

### 3.1 The Problem: What If There Is No Trusted Third Party

The previous chapter assumed the triples already exist, but in practice there is no trusted third party to generate them. The two parties need to generate them "from scratch" on their own. The key tool is 1-out-of-2 oblivious transfer (OT).

### 3.2 Definition of Oblivious Transfer

1-out-of-2 OT involves two roles: the sender prepares two messages m₀ and m₁, and the receiver has a choice bit c ∈ {0, 1}. After execution, the receiver can see only m_c (the message selected by c) and cannot see the other message m_{1-c}; the sender does not know which message the receiver chose. Think of it as a safe with two compartments: the sender places an envelope in each compartment, the receiver uses a key to open only one compartment, and the sender does not know which one was opened. OT can be concretely implemented using RSA, elliptic curves, or other public-key cryptography; here we treat it as a black box.

### 3.3 AND Is Essentially "Choose One of Two"

The key intuition for understanding how OT computes AND is this: the result of u₁ AND v₂ depends only on v₂. When v₂=0, regardless of what u₁ is, the result is always 0; when v₂=1, the result is u₁ itself. AND is inherently a "choose one of two" structure, and OT is precisely the cryptographic tool for "secure choose one of two" -- the two match perfectly.

### 3.4 Expanding the Cross Products of w

After each party randomly selects their shares of u and v, they need to collaboratively compute shares of w = u AND v:

```
w = (u₁⊕u₂) AND (v₁⊕v₂)
  = (u₁∧v₁) ⊕ (u₁∧v₂) ⊕ (u₂∧v₁) ⊕ (u₂∧v₂)
     ─────    ─────    ─────    ─────
     Alice    cross    cross    Bob
     local    term     term     local
```

Alice can compute u₁∧v₁ by herself, and Bob can compute u₂∧v₂ by himself. The difficulty lies in the two cross terms: u₁∧v₂ (Alice has u₁, Bob has v₂) and u₂∧v₁ (Bob has u₂, Alice has v₁). Each party holds one factor, and neither can send theirs to the other.

### 3.5 Using OT to Compute Shares of the Cross Term u₁ ∧ v₂

Alice has u₁, and Bob has v₂. The protocol is as follows:

1. Alice randomly selects a completely independent random bit r. This r has no relation to u₁ whatsoever -- it is purely the result of a coin flip. r will become Alice's output share.
2. Alice prepares two messages based on her u₁: m₀ = r (if v₂=0, the product=0, so Bob should get r, yielding r⊕r=0), m₁ = r ⊕ u₁ (if v₂=1, the product=u₁, so Bob should get r⊕u₁, yielding r⊕(r⊕u₁)=u₁).
3. Bob uses v₂ as the choice bit to execute OT, obtaining t = m_{v₂}. He cannot see the other message, and Alice does not know which one he chose.
4. Result: Alice's share = r, Bob's share = t. XORing the two together yields exactly u₁ AND v₂.

**Correctness verification:**

```
Case 1: v₂ = 0
  Bob gets t = m₀ = r
  Alice's share ⊕ Bob's share = r ⊕ r = 0 = u₁ AND 0 ✓

Case 2: v₂ = 1
  Bob gets t = m₁ = r ⊕ u₁
  Alice's share ⊕ Bob's share = r ⊕ (r ⊕ u₁) = u₁ = u₁ AND 1 ✓
```

**Why r must be independently random:** r plays the role of a one-time pad. If r has any correlation with u₁ (e.g., r = u₁, or r is always 0), Bob might be able to deduce u₁ after seeing m_{v₂}. For example: if r is always 0, then m₁ = u₁, and when v₂=1, Bob directly sees u₁ in plaintext. But when r is truly random, regardless of whether u₁ is 0 or 1, m₀ and m₁ each appear as uniformly random bits, and Bob cannot distinguish them.

### 3.6 Using OT to Compute the Other Cross Term u₂ ∧ v₁

The method is completely symmetric, with roles swapped: Bob acts as the OT sender (he has u₂), randomly selects s, and prepares m₀ = s, m₁ = s ⊕ u₂; Alice acts as the receiver (she has v₁) and uses v₁ as the choice bit. Alice obtains t' = m_{v₁}, and the shares are assigned as Alice holding β₁ = t', Bob holding β₂ = s.

### 3.7 Assembling the Final Shares of w

All four terms now have shares, and each party assembles them locally via XOR:

```
Alice: w₁ = (u₁ ∧ v₁) ⊕ α₁ ⊕ β₁
Bob:   w₂ = α₂ ⊕ β₂ ⊕ (u₂ ∧ v₂)

Verification: w₁ ⊕ w₂
= (u₁∧v₁) ⊕ (α₁⊕α₂) ⊕ (β₁⊕β₂) ⊕ (u₂∧v₂)
= (u₁∧v₁) ⊕ (u₁∧v₂) ⊕ (u₂∧v₁) ⊕ (u₂∧v₂)
= (u₁⊕u₂) ∧ (v₁⊕v₂) = u ∧ v = w  ✓
```

Generating each Beaver triple requires 2 OT invocations. In practice, OT Extension techniques can be used: only about 128 base public-key OTs are needed, after which millions of cheap OTs can be produced using purely symmetric encryption.

### 3.8 Complete Numerical Example

Suppose Alice has u₁=1, v₁=1, and Bob has u₂=0, v₂=0. The true values are u=1, v=1, w=1∧1=1.

```
═══ Local Terms ═══
Alice local: u₁∧v₁ = 1∧1 = 1
Bob   local: u₂∧v₂ = 0∧0 = 0

═══ OT Cross Term u₁∧v₂ ═══
Alice picks r=1, prepares m₀=1, m₁=1⊕1=0
Bob uses v₂=0 to choose → gets t=m₀=1
Shares: α₁=1, α₂=1, verify 1⊕1=0=u₁∧v₂ ✓

═══ OT Cross Term u₂∧v₁ ═══
Bob picks s=0, prepares m₀=0, m₁=0⊕0=0
Alice uses v₁=1 to choose → gets t'=m₁=0
Shares: β₁=0, β₂=0, verify 0⊕0=0=u₂∧v₁ ✓

═══ Assembling Shares of w ═══
Alice: w₁ = 1 ⊕ 1 ⊕ 0 = 0
Bob:   w₂ = 1 ⊕ 0 ⊕ 0 = 1

Verification: w₁ ⊕ w₂ = 0 ⊕ 1 = 1 = u ∧ v ✓
```

The complete Beaver triple shares held by each party are: Alice (u₁=1, v₁=1, w₁=0), Bob (u₂=0, v₂=0, w₂=1). No trusted third party was involved -- the triple was generated entirely by the two parties themselves.

---

## Chapter 4: Generate and Propagate Bits: Expressing Carries with g and p

### 4.1 Carry Formula for a Single-Bit Full Adder

For bit i, let the two addend bits be a_i and b_i, and let the incoming carry from the lower bit be c_i. The carry output is:

```
c_{i+1} = (a_i AND b_i) OR (c_i AND (a_i XOR b_i))
```

Definitions:

- **g_i = a_i AND b_i (generate bit):** When both bits are 1, this position necessarily produces a carry, regardless of what comes from below.
- **p_i = a_i XOR b_i (propagate bit):** When exactly one of the two bits is 1, this position passes the incoming carry through unchanged.
- When a_i = b_i = 0, g=0 and p=0: this position neither generates a carry nor propagates one -- it "absorbs" any incoming carry from below.

Substituting yields the concise recurrence: **c_{i+1} = g_i OR (p_i AND c_i)**.

In MPC, computing g_i requires one secret multiplication (AND, consuming one Beaver triple), while computing p_i is a purely local XOR (free). All bit positions can be processed in parallel, requiring only 1 round of communication.

### 4.2 Unrolling the Recurrence: Full Expression for the Carry

Starting from c₀ = 0, we expand layer by layer:

```
c₁ = g₀
c₂ = g₁ OR (p₁ ∧ g₀)
c₃ = g₂ OR (p₂ ∧ g₁) OR (p₂ ∧ p₁ ∧ g₀)
c₄ = g₃ OR (p₃ ∧ g₂) OR (p₃ ∧ p₂ ∧ g₁) OR (p₃ ∧ p₂ ∧ p₁ ∧ g₀)
```

General formula: c_{i+1} is the OR of multiple terms, each representing a "relay path" -- the carry is born at bit k via g_k, then propagated through p_{k+1}, p_{k+2}, ..., p_i all the way to the top. As long as any one of these paths is open, the carry exists.

The meaning of each term:

```
Term k = p_i · p_{i-1} · ... · p_{k+1} · g_k

Reading: "A carry is born at bit k, then passes consecutively through
          bits k+1, k+2, ..., i, and finally exits from the top of bit i"

Imagine a relay race:
  g_k is the starting runner (the carry originates here)
  p_{k+1}, p_{k+2}, ..., p_i are the relay runners (each passes the baton)
  All relay runners must be present (AND) for the carry to reach the finish
```

### 4.3 Mutual Exclusivity of g and p: OR Reduces to XOR

A critical algebraic property: for any single bit position, g_i = 1 and p_i = 1 cannot both hold simultaneously. Because g_i = 1 implies a_i = b_i = 1, and then p_i = a_i XOR b_i = 1 XOR 1 = 0. Intuitively, a bit position either generates a carry on its own or propagates someone else's carry -- it cannot do both at the same time.

This mutual exclusivity extends to intervals: if G_{i:j} = 1, it means some bit g_k = 1 within the interval, and at that bit p_k = 0, introducing a zero factor into the product chain of P_{i:j}, so the entire P must be 0. For any interval, G and P cannot both be 1.

**This means the OR in the merge formula is free!** In the merge formula G_merged = G_high OR (P_high AND G_low), if G_high = 1 then P_high = 0, so the other term must be 0; and vice versa. The two operands are mutually exclusive, meaning a AND b is always 0, so a OR b = a XOR b XOR 0 = a XOR b, which reduces to a pure XOR -- completely free. Each merge requires only 2 ANDs (instead of 3), saving 1/3 of the triple consumption.

---

## Chapter 5: Prefix Computation: From O(n) to O(log n)

### 5.1 The Meaning of an Interval

The (G, P) of an interval [j, i] answers two questions: G_{i:j} represents "assuming the carry entering at bit j is 0, what is the carry exiting at bit i?" P_{i:j} represents "if a carry of 1 enters at bit j, can it pass through all the bits and exit at bit i?"

A single bit's (g_i, p_i) is the smallest interval [i, i]. The goal is to compute the full interval from bit 0 to bit i, (G_{i:0}, P_{i:0}), at which point G_{i:0} directly equals the carry c_{i+1} (since the initial carry c₀=0).

### 5.2 The Merge Operator ◇

Define the combination operator ◇, which joins adjacent intervals into a longer interval:

```
(G_high, P_high) ◇ (G_low, P_low) = (G_high XOR (P_high AND G_low),  P_high AND P_low)
```

Here G_merged = G_high XOR (P_high AND G_low) means "either the high segment generated a carry on its own, or a carry generated by the low segment was propagated up through the high segment"; P_merged = P_high AND P_low means "for an external carry to pass through the entire segment, both the low and high segments must propagate it." Note that OR has already been replaced by XOR here (due to mutual exclusivity).

This operator is associative, meaning computations can be grouped and parallelized arbitrarily -- this directly gives rise to the prefix tree structure. Each merge requires 2 ANDs (secret multiplications), and multiple merges at the same level can be executed in parallel, requiring only 1 round of communication.

### 5.3 Concrete Prefix Tree Process (4-Bit Example)

Take A=1011 + B=0111 as an example (answer 18 = 10010).

**Initialization:** Compute g and p for all bit positions in parallel (1 round of communication):

```
Bit 0: a=1, b=1 → g₀=1, p₀=0  (generates, does not propagate)
Bit 1: a=1, b=1 → g₁=1, p₁=0  (generates, does not propagate)
Bit 2: a=0, b=1 → g₂=0, p₂=1  (does not generate, propagates)
Bit 3: a=1, b=0 → g₃=0, p₃=1  (does not generate, propagates)
```

**Level 1 (span 1, 1 round of communication):** Merge adjacent pairs of bits; both merges execute in parallel.

```
[1:0] = (g₁,p₁) ◇ (g₀,p₀) = (1 XOR (0∧1), 0∧0) = (1, 0)
  → Bits 0 and 1 together generate a carry on their own

[3:2] = (g₃,p₃) ◇ (g₂,p₂) = (0 XOR (1∧0), 1∧1) = (0, 1)
  → Bits 2 and 3 together do not generate a carry, but can propagate one
```

**Level 2 (span 2, 1 round of communication):** Merge the Level 1 results.

```
[3:0] = (G_{3:2},P_{3:2}) ◇ (G_{1:0},P_{1:0}) = (0,1) ◇ (1,0)
     = (0 XOR (1∧1), 1∧0) = (1, 0)
  → The carry generated by bits 1:0 is propagated up through bits 3:2; the entire segment generates a carry
```

### 5.4 Extracting the Final Carries and Computing the Sum

c₁ = G_{0:0} = g₀ = 1, c₂ = G_{1:0} = 1, c₃ = G_{2:0} = 1, c₄ = G_{3:0} = 1. Then the sum at each bit is s_i = p_i XOR c_i (purely local XOR, 0 communication):

```
s₀ = p₀ ⊕ c₀ = 0 ⊕ 0 = 0
s₁ = p₁ ⊕ c₁ = 0 ⊕ 1 = 1
s₂ = p₂ ⊕ c₂ = 1 ⊕ 1 = 0
s₃ = p₃ ⊕ c₃ = 1 ⊕ 1 = 0
Overflow carry c₄ = 1

Result: 10010 = 18 = 11 + 7 ✓
```

### 5.5 Communication Complexity Summary

| Step | Operation | Communication Rounds |
|------|-----------|---------------------|
| Compute g, p | n parallel ANDs + n local XORs | 1 round |
| Prefix tree merging | Several parallel ANDs per level | O(log n) rounds |
| Extract carries | Local read | 0 rounds |
| Compute sum | Local XOR | 0 rounds |
| **Total** | | **O(log n) rounds** |

---

## Chapter 6: Security Analysis: No Carry Is Ever Revealed

### 6.1 End-to-End Share Tracking

Throughout the entire computation, all intermediate bits exist in the form of secret shares. The initial a, b are shares; the computed g, p are shares; the G, P produced by each level of the prefix tree are shares; and the final carries c and sums s are still shares. At no step is it necessary to "first reconstruct the plaintext and then continue computing."

### 6.2 The Only Public Information: (d, e) in Beaver AND

Each Beaver AND reveals a pair (d, e), where d = x ⊕ u and e = y ⊕ v. Since u and v are completely random triple components used only once, the distribution of d and e is entirely independent of the true inputs -- this is the security of the one-time pad, at an information-theoretic level, relying on no computational complexity assumptions.

Taking 4-bit addition as an example, the information revealed throughout the process is: the initial g, p computation reveals 4x2 = 8 (d,e) pairs, the Level 1 merging reveals 4x2 = 8 (d,e) pairs, the Level 2 merging reveals 2x2 = 4 (d,e) pairs, for a total of 20 revealed bits. All 20 bits equal "real data XOR independent random triple," which amounts to 20 independent uniformly random bits -- zero information.

### 6.3 Security Conclusion

All the information Alice sees throughout the entire process includes: her own input shares, her own triple shares, the revealed (d,e) values, and the intermediate shares she computes. The joint distribution of this information is completely independent of Bob's true input. In other words: regardless of what Bob's input is, the distribution Alice sees is exactly the same. The carries c₁ through c₄ are never revealed to either party, and the final sum s is not revealed until it is deliberately "opened."

---

## Summary: The Complete Technical Pipeline

The entire technical pipeline presented in this article can be strung together in a single narrative:

1. In Boolean secret sharing, XOR is a free linear operation, while AND is a nonlinear operation requiring communication.
2. Each secret AND consumes one Beaver triple and one round of communication: the randomly masked (d, e) are revealed, and the result shares are assembled locally.
3. Beaver triples are generated by the two parties themselves using OT: the cross terms of AND are decomposed into a "choose one of two" problem, masked with independent random r, and securely executed via OT.
4. The carries in binary addition are decomposed into generate bits g and propagate bits p, whose mutual exclusivity causes the OR in the merge formula to reduce to a free XOR.
5. The prefix tree leverages the associativity of the merge operator to reorganize the chain-dependent carry computation into a tree-parallel structure, compressing from O(n) rounds to O(log n) rounds.
6. Throughout the entire process, all intermediate values exist as secret shares; the only revealed (d, e) values are perfectly hidden by one-time pads, achieving information-theoretic security.

From the low-level OT primitive, to Beaver triples, to secret AND gates, to g/p computation, to the parallel merging of the prefix tree -- each layer builds upon the one below, ultimately achieving the complete carry computation of binary addition within O(log n) rounds of communication, without ever revealing a single bit of plaintext.
