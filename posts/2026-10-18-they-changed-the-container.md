---
layout: default
title: They changed the container, not the secret
---

A key is only as unpredictable as the pool it was chosen from. That sentence sounds obvious, and almost every serious wallet failure is a variation on ignoring it.

Rainbow tables are the classic example, and they are worth a second because they are so often misdescribed. They do not break hashing. The hash function is fine. What they break is the fact that *people choose* — passwords come from a small, predictable corner of the possible space, so an attacker precomputes that corner once and walks in. The weakness was never in the algorithm. It was in the word "chosen."

Keys have the same disease, and it shows up the instant something is *selected* instead of *drawn at random*: a prefix a trader wants, a passphrase a person can remember, a seed a device quietly shortcuts. What follows are two incidents that look unrelated — a market maker losing $160 million on Ethereum in September 2022, and around 500 Bitcoin wallets swept in twenty-five minutes in July 2026 from a device sold on the opposite promise, air-gapped and offline. They are the same story told about keys. And the people who owned the money reacted to both in the same wrong way.

## The bug that is not a cryptography bug

Start with what did *not* break. In neither case did anyone break secp256k1, forge a signature, or find a flaw in ECDSA. The elliptic-curve math worked exactly as specified. The failure is one step upstream of the cryptography, in a place auditors are trained to walk past: the size of the input that the key is built from.

**Profanity (the Wintermute case).** Profanity was a popular tool for generating Ethereum *vanity addresses* — addresses with a chosen prefix, like a run of leading zeros. An Ethereum address is the last 20 bytes of `keccak256(pubkey)`, so you cannot choose it; you can only try keys until one hashes to something you like. To do that fast, Profanity seeded its generator with a single 32-bit value and fed it into a deterministic PRNG (`mt19937`) to expand into a 256-bit "private key."

```
What it looked like:   a 256-bit key      2^256  ≈ 10^77
What it actually was:  a 32-bit seed      2^32   ≈ 4.3 × 10^9
```

The key *looked* 256 bits wide. Every key the tool could ever produce lived inside a set of 4.3 billion — a space you can enumerate on a laptop. 1inch disclosed this on 15 September 2022.

**Coldcard (the July 2026 case).** Coldcard is an air-gapped Bitcoin hardware wallet. A firmware regression shipped in v4.0.0 (March 2021) meant that seed generation silently bypassed the STM32's hardware RNG and fell back to a software PRNG seeded from the device serial number and timer registers. The cause was a preprocessor guard, `#ifndef MICROPY_HW_ENABLE_RNG`, that checked whether the macro was *defined* rather than whether it was *enabled* — and the vendor had defined it as `0`. The build compiled clean. No `#error` ever fired.

```
Intended entropy (24 words):   2^256
Mk2 / Mk3 (fw 4.0.1–5.0.3):    ~2^40
```

Same category as Profanity: a secret that advertises one width and is actually built from another. On 30 July 2026, roughly 500 single-sig wallets — about 594 BTC — were swept in four blocks.

The two arrive at the same place by opposite routes, and the contrast is the point. In Profanity the truncation is visible in one line of source: the seed is 32 bits, right there. In Coldcard the *correct* hardware RNG was present in the compiled binary the whole time. It was simply never the one that reached the seed path. Reviews confirmed the good generator existed. None of them checked which of two identically-named functions the seed actually came from.

> Present is not the same as used.

## Why none of the defenses were pointed at the problem

This is the part that matters for anyone choosing a wallet on a feature list.

**Air-gap was irrelevant.** The Coldcard's whole pitch is that the secret never touches a networked machine. But the attacker never touched a Coldcard. He enumerated the seed space offline and derived the keys — 500 wallets in twenty-five minutes. An air-gap protects *where a secret is stored*. It can do nothing for a secret that is *computed*, because the attacker reconstructs it on his own hardware and never comes near yours.

**The strongest available defense was optional — and only worked if you used it well.** Coldcard did offer a way out: roll your own entropy with dice, or set a BIP-39 passphrase. But "set a passphrase" is not the same as "be safe," and this is where it gets sharp. The passphrase does not add to the attacker's search space; it *multiplies* it. Once the mnemonic had collapsed to ~2^40, the passphrase stopped being a bonus and became the wallet's entire security budget. A passphrase with real entropy — three or more genuinely random words — put the keys out of reach even for a thousand-GPU farm. A passphrase like `alice`, `qwerty`, or a single dictionary word fell to a wordlist in months, because it was *chosen* from the same small, predictable corner that dooms a reused password.

The dice are the cleaner escape, and they make the real point. Fifty dice rolls are the simplest, most basic source of entropy there is — exactly the kind of brute, low-tech input that is hard to get subtly wrong, precisely because there is nothing to choose. The vendor shipped both options. But they were opt-in, the default quietly failed, and one of the two options only helped if you understood *why* it helped.

> An optional protection is a courtesy, not a protection. And a protection you apply without its entropy is a password named `alice`.

**And the vanity requirement itself was the wrong trade.** It is tempting to say "the more specific you make something, the weaker it gets." As a general law that is false — a vanity key drawn from a *properly* seeded search is exactly as strong as any other key, because the chosen prefix costs search time, not entropy. The real failure is narrower and worth stating precisely: the special requirement (saving gas on a zero-prefixed address) pushed Wintermute off the standard key-generation path that many people have scrutinized, and onto a niche tool that almost no one had. The weakness was not the vanity. It was leaving the well-trodden path to get it.

## The reaction both parties got wrong

Here is the pairing I have not seen written down, and it is the reason to put these two incidents in one frame.

Both victims understood, correctly, that something was wrong. Both acted. Both acted on the wrong noun.

**Wintermute.** Around the time of the Profanity disclosure, Wintermute drained all the ETH out of its vanity admin address — a clear sign they had understood it might be compromised. But that address was also an *admin* of their DeFi vault, and they did not remove it. The attacker recomputed the private key, signed as admin, and moved the vault. The CEO later called it an internal human error. The contracts worked exactly as written: the vault lets admins transfer, and the compromised key was still an admin.

**Coldcard users.** The guidance for a leaked-entropy seed is brutal and non-obvious: a seed born on an affected device stays weak *even after you restore it onto a different brand*. Moving to a new, pristine device does nothing, because the weakness was fixed at the moment of generation. Someone who swapped hardware but kept their seed changed nothing that mattered.

Line them up:

| | what they changed | what they kept |
|---|---|---|
| **Wintermute** | emptied the ETH from the address | the key, still admin of the vault |
| **Coldcard users** | moved to a new device | the same seed |

Both moved the balance and kept the authority. Both treated the danger as living in *where the money sits* — the address, the device — when the danger lived in *the secret itself*, wherever it went. You cannot evacuate a key by emptying the wallet in front of it. If the secret is derivable, the only fix is a new secret: rotate the key, revoke the old one everywhere it has power, generate a fresh seed from entropy you control.

## What actually holds

What actually held, in both stories, was never the expensive part. It was the cheap part — when someone used it, and used it with real entropy. Fifty dice rolls produce full-strength randomness; there is nothing primitive about them except the price. A passphrase of three random words does the same. The wallets that survived the Coldcard sweep survived on one of those, not on the air-gap and not on the secure element — and the ones that *thought* they were covered by a passphrase like `michael` learned that a secret you choose badly is not a secret at all.

Because the whole edifice rests on a secret, and a secret only has to do one thing: stay secret. The moment you start making assumptions about how strong it is — 256 bits because the output is 256 bits wide, safe because the device is offline — you have stopped protecting it and started hoping. Strength you assumed is not strength you have.

There is a way to think about this that has nothing to do with cryptography. If you actually want to disappear, you do not buy the Ferrari and you do not retreat to an isolated tower on a hill. Both announce exactly where to look. You walk into Mumbai and you are gone — not because the city is a fortress, but because nothing about you stands out and nothing about you can be derived. A vanity address is a Ferrari. An air-gapped tower is a tower. The key that was never worth attacking is the one drawn, plainly and fully, from entropy no one else can reconstruct.
