---
layout:     post
title:      "SPDZ MAC: Malicious Model Principles"
date:       2026-01-11
author:     "Invisibook Lab"
tags:       ["MPC", "SPDZ", "MAC", "cryptography"]
categories: ["Tech"]
---

Secure Multi-Party Computation (MPC) allows multiple participants to jointly compute a function without revealing their private inputs to each other. SPDZ is a widely used MPC protocol that not only protects privacy but also withstands malicious adversaries -- even if some participants attempt to cheat or tamper with data, the protocol can detect such behavior and guarantee the correctness of the result. One of its key techniques is the **information-theoretically secure Message Authentication Code (MAC) mechanism**.

---

### What Role Does the Information-Theoretic MAC Play in SPDZ?

In the SPDZ protocol, each secret-shared value is not merely a "share" -- it also comes with an **authentication tag (MAC tag)**. This tag is generated from a combination of the value and a **global key** that is shared among all participants but unknown to any single participant. In other words:

* Every secret-shared value (v) has a corresponding MAC tag.
* The MAC tag is the result of "signing" (v) with a global key, but this signature is stored in secret-shared form.
* Each participant holds a portion of both the shared value and the corresponding MAC share.

Because this global key is hidden from all participants (it is produced via secret sharing and never exists in its entirety in any single party's hands), no individual party can forge a valid MAC tag on its own. This MAC does not rely on computational assumptions (such as the hardness of integer factorization) -- even if an adversary has unlimited computational resources, as long as they do not possess the complete key, they cannot forge a valid tag. This is why it is called **information-theoretically secure**.

---

### How Does the MAC Prevent Tampering?

#### 1. Every Value Has an Authentication Tag

In SPDZ, every shared value involved in the computation has:

* A secret-shared value;
* A corresponding MAC authentication tag.

This tag certifies that the value was legitimately produced by the protocol, rather than arbitrarily fabricated by a participant.

#### 2. Operations During the Online Phase Preserve MAC Consistency

During the online computation phase of the protocol, operations such as addition and multiplication are performed not only on the shared values but also on their corresponding MAC tags. The protocol design guarantees that:

* If a participant attempts to tamper with an intermediate secret value, they **must simultaneously tamper with the corresponding MAC tag**;
* However, since the MAC tag is generated based on the global key -- of which each participant holds only a partial share and which cannot be reconstructed -- no participant can construct a valid new MAC tag on their own.

In other words, throughout the computation, MAC tags "follow" the secret-shared values as they propagate. Any unauthorized modification will cause an inconsistency in the tag relationship.

#### 3. MAC Check When Revealing the Final Value

When the protocol needs to reconstruct a secret value (e.g., outputting the final result), participants publicly share their value shares and MAC tag shares. The protocol verifies consistency through the following two steps:

* **Combine all value shares and MAC tag shares** to obtain the complete value (v), the complete MAC tag (m), and the complete global key shares (k);
* **Verify whether** (m = text{MAC}(k, v)) holds (i.e., the tag correctly corresponds to the value and the key).

If any part has been tampered with (whether the value or the tag), the combined verification will fail. Therefore, the protocol can detect even a single participant's attempt to output an incorrect result.

---

### Why Is This Mechanism Powerful Enough?

This information-theoretic MAC mechanism has two core advantages:

* **Independent of the adversary's computational power**: Even with unlimited computational resources, an attacker cannot construct a matching MAC without the complete key shares. This provides stronger security than approaches based on hard problems like integer factorization.
* **Tamper protection covers the entire computation**: Not just the final output -- every operation is accompanied by tag maintenance and checking, so cheating is detected before the output is revealed.

Thanks to these properties, SPDZ's information-theoretic MAC is one of the cornerstones for achieving **malicious security**. This means the protocol not only prevents participants from peeking at data but also ensures that participants cannot manipulate the computation result by intentionally deviating from the prescribed operations.

---

## Example

There are two people:

* A
* B

Goal:
**In a setting where no one is trusted, guarantee that no one can tamper with intermediate results.**

---

## Phase 1: Generate a "Rule Number alpha That No One Knows"

### Step 1: Each Person Picks a Secret Number

* A picks: alpha_a = 3
* B picks: alpha_b = 4

### Step 2: Logically Define

> **The true alpha = 3 + 4 = 7**

Key facts:

* A does not know 4
* B does not know 3
* **No one knows alpha = 7**

But it **does exist**.

---

## Phase 2: Secret-Share an Ordinary Number x

Suppose:

* x = 3

### Split into two shares:

* A gets: x_a = 1
* B gets: x_b = 2

1 + 2 = 3 ✔

---

## Phase 3: Generate the "Shadow"

### Key Principle (remember this well):

> **The shadow is not "computed by someone";
> it is "a consistency constraint defined by the system."**

The true rule is:

> **Shadow = alpha x x**

Logically:

* alpha = 7
* x = 3
* Therefore, shadow = 21

⚠️ Note:

* **No one actually "computed" 7 x 3**
* 21 is merely "a fact that must hold"

---

### So How Is the Shadow Split Between A and B?

The system does only one thing:

> **Split it arbitrarily, as long as the two shares add up to 21**

For example:

* A gets a shadow share: 3 x 3 = 9
* B gets a shadow share: 3 x 4 = 12

9 + 12 = 21 ✔

---

## Now, What Does Each Person Hold?

### A holds:

* A share of the number: 1
* A share of the shadow: 9

### B holds:

* A share of the number: 2
* A share of the shadow: 12

At this point, a global fact holds:

> **(A's shadow + B's shadow)
> = alpha x (A's number + B's number)**

Moreover:

* A does not know alpha
* B does not know alpha

---

## Phase 4: Begin Computation (e.g., Subtraction, Comparison)

Suppose we want to compute:

> z = x - y

The rule is:

* However the numbers change,
* the shadows **must change in exactly the same way**

For example:

* A:

  * z_a = x_a - y_a
  * shadow_a = shadow_x_a - shadow_y_a
* B does the same

This step **does not require knowing alpha**.

---

## Phase 5: A Attempts to Cheat (The Critical Part)

A thinks:

> "I'll secretly inflate my z_a a little."

So A modifies:

* Number share: ✔ A can change it
* Shadow share: ❌ A does not know what to change it to

Because:

> **The shadow must still satisfy:
> total shadow = alpha x total number**

But alpha is unknown to A.

---

## Phase 6: The Final Check (MAC Check)

What the system does can be summarized in one sentence:

> **Add up everyone's number shares,
> add up everyone's shadow shares,
> and check whether
> 'shadow = alpha x number' still holds**

If A has cheated:

* This equation will certainly not hold
* There is no strategy to "guess correctly" (the probability is negligibly small)

---



## Summary

In the SPDZ protocol, the information-theoretically secure MAC is a protection mechanism for computational integrity. By attaching an authentication tag to every shared value, it:

* Prevents any malicious participant from forging a valid tag on their own;
* Maintains consistency between tags and values throughout every operation;
* Checks whether tags and values match when results are revealed;

thereby effectively preventing any unauthorized tampering and guaranteeing correctness and security from within the protocol itself.
