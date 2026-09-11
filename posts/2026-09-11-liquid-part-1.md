---
layout: default
title: Every Optimization Is a Claim About Equivalence
---

### What the Liquid drain was actually about — Part 1: the mechanism

---

I found this story the way most people did: as an argument on X that I could not follow.

Two people were disagreeing about whether the Liquid drain said anything about privacy. One said the cryptography was fine and a cache broke. The other said the cache only existed *because* privacy is expensive, and that hidden amounts are what let the theft go unnoticed. Both sounded right. I did not have enough background to tell which was.

So I went backwards, to the primitives.

I came into smart contract security from cryptography rather than from writing applications, and that shapes what I notice first. When I read that four thousand coins had been created out of nothing on a chain that hides amounts, my first thought was not "exploit." It was: *this is the wrap-around problem again.* I have now met that same problem in three different places wearing three different costumes, and it is worth understanding properly, because it does not stop showing up.

This post is about how the theft worked. There is a second story — who knew what, and when, and how it was classified — and that one is still being argued about in public by the people involved, so I am leaving it until both sides have published. Part 2 will cover it.

---

## What happened

On 6 September 2026, at 15:53:10 UTC, Liquid block 4,050,336 contained a transaction whose range proof was invalid. The network accepted it anyway.

Roughly 4,000 unbacked L-BTC were created and pegged out for real bitcoin. About 3,996 BTC left the federation wallet in twenty-three minutes — around 95% of every bitcoin ever pegged into the sidechain. Reserves went from 4,205 BTC to 202.

No private key was stolen. No signer was phished. No hardware security module was tricked. The 11-of-15 multisig produced eleven valid signatures, because from where it sat the withdrawal looked ordinary.

That last sentence is the whole story, and the rest of this post is about why the strongest part of the security model was irrelevant.

---

## Why confidential transactions need range proofs

Liquid hides transaction amounts. To understand what broke, you need to understand what that costs.

An amount is hidden inside a **Pedersen commitment**:

```
C = v·G + r·H
```

Here `v` is the amount, `r` is a random blinding factor, and `G` and `H` are two curve points with no known relationship between them. The commitment **hides** — `r` is random, so `C` reveals nothing about `v`. And it **binds** — finding a different pair `(v', r')` that produces the same `C` means solving a discrete log.

The elegant part is that commitments add:

```
C₁ + C₂ = (v₁+v₂)·G + (r₁+r₂)·H
```

The sum of the commitments *is* a commitment to the sum. So a node can verify

```
Σ(inputs) − Σ(outputs) = a commitment to zero
```

and confirm that value was conserved without ever seeing a single amount. Blind bookkeeping that actually works.

### The hole

Amounts live modulo the curve order. And inside a modular group, **there is no such thing as negative**.

Here it is with a fake modulus of 100, so the shape is visible:

```
input:      10

output A:   95      ← this "is" −5
output B:   15

sum:        95 + 15 = 110 ≡ 10  (mod 100)   ✓
```

The conservation check passes perfectly. Output B holds 15, from an input of 10. Five units have been created from nothing.

On a real curve, the "−5" is a 78-digit number. It is a perfectly valid field element. The verifier has no way to recognise it as negative, because negativity is not a property the structure has.

This is why every confidential output carries a **range proof**: a proof that the hidden value sits inside `[0, 2ᵏ)` without revealing what it is.

> Homomorphic addition guarantees equality. It does not guarantee non-negativity. The range proof is the only thing that makes "amount" mean amount.

And range proofs are expensive. Remember that, because it is where this goes.

---

## The cache

A transaction is not validated once. It is validated on mempool acceptance, again on block validation, again during a reorg, again if a block is reconsidered. Bitcoin Core keeps a signature cache for exactly this reason. Elements, a fork of Core, added the equivalent for range proofs: `CachingRangeProofChecker`, in `sigcache.cpp`.

A cache is a table from keys to answers, and it carries one silent promise:

> **The key uniquely identifies the question.**

If two different questions produce the same key, the answer to one becomes a free answer to the other. The cache did not lie. It answered a question you never asked.

### The question was bigger than the key

A range proof does not stand alone. It proves something about a specific commitment, in a specific context. On Elements, which supports multiple assets, that context includes the proof itself, the value commitment, the asset commitment, and the script.

The real question is the whole tuple. The root cause, per CertiK's analysis, was an **ambiguous cache-key encoding that allowed two different validation inputs to produce the same cached entry**. Independent analysis of the disputed block found that the malicious output's range proof was cryptographically invalid *under its own (asset, script) context*.

The key covered the proof. It did not fully cover the context.

### Priming

The attacker did not submit an invalid proof and hope. The sequence was better than that:

```
STEP 1 — PRIME
  Send setup transactions. Entirely valid.
  The node verifies them properly, spends the time,
  and writes:  key K → VALID
  Nothing looks wrong, because nothing is wrong.

STEP 2 — COLLECT
  Send the inflation transaction.
  Different context. Same key K.
  Node: "I know this one." → VALID
  The range proof never runs.
```

Teach the cache a true answer, then ask a different question with the same key.

With no range proof enforced, the mod-100 trick above is available in full. Four thousand L-BTC out of nothing.

The rehearsal was careful. On 4 September, two small peg-ins totalling 2.15 BTC. On the morning of 6 September, three dry-run peg-outs of 0.95, 1.71 and 0.55 BTC between 11:30 and 13:16 UTC. Every one completed: the peg-out mechanism worked, the federation signed, real bitcoin arrived. Then the real one.

---

## The part that should worry you more

A cache is per-node memory. It is not consensus state. It does not synchronise.

So a node's verdict on a block depended on what happened to be in its local memory. A node with a primed cache: valid. A node without: invalid.

That is exactly what occurred. Part of the network accepted the block and part rejected it, and the chain split. The functionaries followed the accepting side, so the peg-outs were paid.

> **Once a verdict depends on local memory, consensus stops being deterministic.**

An optimization living in per-node state had been placed inside the consensus decision path. That is a design property, not an accident, and it was true for years before anyone exploited it.

---

## One layer over, the same problem was taken seriously

This is the part I did not expect to find.

Simplicity activated on Liquid mainnet on 1 August 2025, after eight years of development. It is a smart contract language built from nine combinators, with no unbounded loops and no global state, whose semantics are formally specified in Coq. As Adam Back put it, it is verification-only — it does not have to run the program.

Nine combinators is beautiful mathematics and computationally hopeless. A SHA-256 written in raw combinators is millions of tiny steps. So Simplicity has **jets**: the node recognises a subexpression, and instead of evaluating it step by step, calls a native implementation. The Elements integration has 471 of them, fixed at the time of integration.

A jet carries exactly one correctness requirement: it must produce the same result as the expression it replaces, for every input, always.

Why that requirement is absolute is worth stating plainly. You write a contract in Simplicity. You formally prove it correct. The proof is about the combinators. But what executes is the jets. If they diverge on even one input, you have proven a property of something that never runs.

Now put the two side by side:

```
JET     fast path:   native C
        slow path:   millions of combinators
        equivalence: same output for every input
        enforced by: formal specification in Coq

CACHE   fast path:   "I know this one, it's valid"
        slow path:   verify the range proof
        equivalence: the key must determine the question
        enforced by: a hand-written key encoding
```

Same problem. Same company. Same system. Two completely different standards of rigour.

This is not hypocrisy and I do not want to write it as though it were. It is something more useful: the discipline existed at one layer and not at the layer beneath it. Formal verification of the program layer does not save you when the validation infrastructure underneath is hand-optimised C++.

Which gives the general form, and it is the reason I wanted to write this at all:

> **Every optimization is a claim about equivalence: that the fast path returns what the slow path would.**
> **The claim is either proven or assumed. The bug always lives in "assumed."**

---

## So was it about privacy?

Back to the argument I could not follow.

The cryptography did not break. Pedersen commitments did what they do, the homomorphic check did what it does, and Bulletproofs were never defeated. In that narrow sense, "this is not an argument against privacy features" is correct.

But the cache did not appear from nowhere. It exists because range proof verification is expensive and repeated, and range proofs are mandatory *because* amounts are hidden. Remove confidentiality and the whole structure — expensive proof, hot path, memoisation, key encoding — has no reason to exist.

So privacy did not fail. It **presented a bill, and the bill was paid in a different currency**: an optimization that nobody proved equivalent to the thing it replaced.

That reframing also points at the fix. Verification cost is not a constant of nature.

---

## What could have made the cache unnecessary

I am still learning proof systems, so I am raising this as a possibility rather than a proposal.

The cache exists because verification is expensive and repeated. That suggests a different question than "how do we fix the cache." It suggests: **why does the cache need to exist at all?**

In PLONKish proof systems, range checks are handled by lookup arguments — proving a value belongs to a precomputed table rather than decomposing it into bits and constraining each one. That inverts the cost profile completely: range checks go from among the most expensive operations to among the cheapest. If verification were cheap enough, nobody would write a cache, and this class of bug would have nowhere to live.

Two objections come up immediately, and one of them does not survive contact with the actual system.

**"That would need new opcodes."** True. Pairing-based verification is not available in Elements today. But Liquid is not Bitcoin L1 — it ships consensus changes on a regular cadence, coordinated with a known federation. This is a network that hard-forks as a matter of routine.

**"That would introduce a trusted setup."** This is the objection I do not think survives, and the reason is structural rather than technical.

Liquid is a federated sidechain. Its peg already depends on eleven of fifteen named functionaries behaving correctly — continuously, every day, for as long as the network exists. A KZG ceremony requires that **one** participant was honest **once**, after which it is settled permanently and nobody has to keep being trustworthy.

Those are not the same kind of assumption, and the second is the weaker one. A system that has already accepted ongoing eleven-of-fifteen honesty is not in a position to reject a one-time at-least-one-honest-participant assumption on purity grounds. This is an L1 argument applied to something that is not an L1. On Bitcoin mainnet, refusing a ceremony is coherent. Here it is borrowed reasoning.

Which leaves the actual trade-off, and it is a real one. A proof system verifier inside consensus is a new and subtle surface. Four hundred and seventy-one jets are four hundred and seventy-one trust assumptions, even with formal specification behind them. Rigour does not remove surface; it changes what kind of claim you are making about it.

But that is a comparison between assumptions of different shapes, which is the only comparison actually on the table. Security is a spectrum, not a binary, and a federated sidechain that pretends otherwise is arguing with a version of itself that does not exist.

---

## Closing
Four systems, one shape.

Parity had a wallet library that did exactly what its code said. Ronin had a five-of-nine multisig that was never broken. Mango had an oracle reporting the truth about a market. Liquid had eleven hardware security modules producing eleven valid signatures. In every one, the thing people point at when they say "this is secure" was functioning correctly while the money walked out — because the failure was somewhere nobody had labelled as security.

So the question I try to ask first is no longer whether something is secure. It is: when this fails, how will I find out, and will it be before or after?

Liquid's answer was after. The discrepancy became publicly computable only once the peg-outs hit Bitcoin's transparent chain — by which point each L-BTC was backed by roughly 4.7% of a real bitcoin, and there was nothing left to protect.

Twenty-three minutes after.

---

*Part 2 will cover the disclosure timeline: when the bug was known, how it was classified, and why a fix can exist for weeks without reaching the people running the money. Both Blockstream and the Bitcoin Red Team have said they will publish their accounts. I would rather read those first than guess.*
