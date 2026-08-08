# BIP39 Coin-Flip Entropy Generator

A single, self-contained HTML page that turns physical coin flips into the entropy words of a BIP39 mnemonic — entirely offline, entirely in your browser. It deliberately stops one word short of a complete mnemonic; see [Why it stops one word short](#why-it-stops-one-word-short).

**[Try it live](https://benvhodl.github.io/bip39-coinflip/)**, or just open [`index.html`](index.html) directly in any browser.

## What it does

- Lets you choose a **12-word** or **24-word** seed length up front. The page then walks you through generating the first **11** or **23** words respectively — the entropy words — and stops.
- Offers two entropy-generation modes:
  - **Standard** — one coin flip = one bit, 11 flips per word (Heads = 1, Tails = 0).
  - **Bias-resistant** — a [von Neumann extractor](https://en.wikipedia.org/wiki/Randomness_extractor#Von_Neumann_extractor): flip the coin twice per output bit. Heads→Tails = 1, Tails→Heads = 0, and Heads→Heads / Tails→Tails are discarded and re-flipped. Continues until 11 bits have been extracted for the word.
- Combines each word's bits most-significant-bit first into a number from 0–2047 and looks that number up in the full 2048-word BIP39 English list embedded in the page, which was checksum-verified (SHA-256) against the canonical list when this page was built. It doesn't re-check itself against a hardcoded hash at runtime on purpose: a copy of this file with an altered word list could just as easily ship an altered "expected hash" alongside it, so that kind of self-check mostly guards against accidental corruption rather than a deliberately malicious copy. The real defense is verifying this file itself — reading the code or comparing it against the canonical source.
- In Standard mode, alternates which face you're asked to start each flip on (and rotates through your coins, one per word) as a mitigation for the real, measured [same-side bias](https://arxiv.org/abs/2310.04153) of physical coin flips, and asks you to land the coin on a soft surface rather than catching it. Unlike some coin-flip tools, this starting-face pattern is fully **deterministic** (alternating by word number and flip number) rather than drawn from a random number generator — this page calls no `crypto.getRandomValues`, `Math.random`, or any other software RNG anywhere, for anything. See [Threat model](#threat-model) for the tradeoff that implies.
- Shows only one word at a time. After you write it down and confirm, the page clears that word's bits, its computed index, and the word itself from both memory and the DOM before moving to the next one — there is never a growing list of words on screen, and no array anywhere in the code holds the complete sequence.
- Never uses localStorage, sessionStorage, cookies, IndexedDB, or a service worker, and makes no network requests.
- Ships a small self-test suite (word-list integrity, bit→word mapping, von Neumann extractor logic, state-clearing) runnable from the page itself via the "Diagnostics" panel, using synthetic test vectors only — it never touches real flip data.

## Why it stops one word short

The last word of a BIP39 mnemonic isn't fully free — it encodes a checksum (plus a few leftover entropy bits) computed from a SHA-256 hash of *all* the preceding entropy, words included. This page contains no checksum code at all: not a disabled feature, just logic that was never written. There is no button, mode, or hidden state anywhere in this file that computes a final word, reconstructs full entropy, derives a seed, or builds an extended key.

```
physical coin
     |
this computer  -->  entropy words only  -->  you write them down
     |
computer forgets them
     |
final checksum word generated elsewhere
     |
complete seed exists only outside this computer
```

The point is to minimize how much wallet-secret material a single computer ever holds. Even a fully trustworthy machine today could be compromised later; a machine that never held the complete seed can't leak it. You get the final word from a trusted hardware device, or another explicitly trusted offline method — see the two workflows below.

## Threat model

- **Trusted:** the user's physical coin flips, as the entropy source. The application assumes the user obtains entropy from physical coin flips; it does not attempt to prove that this physical entropy source is unbiased or that successive flips are independent. Standard mode reduces two specific, measured biases — same-side-start bias and catch-based bias — via the alternating-start-face convention and the land-don't-catch instruction. Bias-resistant mode instead removes any fixed-but-unknown per-flip bias mathematically, via the von Neumann extractor, under the assumption that consecutive flips are independent trials; it does not guarantee immunity to arbitrary correlations between flips (e.g. if how you catch or re-place the coin depends on the previous outcome).
- **Trusted, narrowly: the browser.** The browser is trusted only to faithfully transform the bits you report into BIP39 output — combining them into an index and looking that index up in the embedded word list — without altering the values you enter. This version of the page uses **no software randomness whatsoever**: the deterministic start-face pattern in Standard mode is computed from the word and flip number alone, not from any RNG. The tradeoff is that this pattern is predictable to anyone who reads the code — but that predictability can't inject bias into a bit's value, since every bit always comes from whatever the physical coin actually showed you. It just means Standard mode's starting-face mitigation falls back to the baseline physical bias if you consider that pattern compromised, rather than staying hidden behind a random seed.
- **Not trusted, not needed: the network.** Once loaded, the page makes no network requests and stores nothing (no localStorage, cookies, or history). The one exception is the hosted GitHub Pages copy itself, which must be trusted once, at load time, to deliver unmodified code — see [Usage](#usage) below for why running an offline copy avoids that.
- **Out of scope:** whether any specific physical coin is fair, whether the person flipping it introduces bias, and anything about the entropy quality of individual coin flips beyond the mitigations described above. Also out of scope, by design: the final checksum word, and anything downstream of it (seed derivation, key derivation, wallet creation).

## Generating the final word with a hardware wallet

This page only ever produces the entropy words — 11 of 12, or 23 of 24 — and deliberately can't produce a complete, checksummed mnemonic on its own (see [Why it stops one word short](#why-it-stops-one-word-short)). One way to turn its output into a complete, valid seed phrase is to pair it with a hardware wallet that supports entering your own manually-generated entropy, such as the [Blockstream Jade](https://blockstream.com/jade/):

1. Pick 12-word or 24-word mode on this page and work through the flips it asks for, writing each entropy word down on paper as you go. The page tells you when it's done — 11 of 11 or 23 of 23 — and stops there on purpose.
2. On Jade, use its [recovery-phrase-from-your-own-entropy flow](https://help.blockstream.com/blockstream-jade/add-more-security-functionality/create-a-recovery-phrase-using-dice) (Blockstream's guide uses dice as the example, but any true-random source — including coin flips — works identically) and enter your 11 or 23 words.
3. Jade can't hand you back a single specific final word, because part of it is still genuine unpicked entropy; instead it restricts the keyboard to only the words that produce a valid checksum, and either lets you choose or offers to "randomly select a final word for you." Take the random option (or otherwise pick without deliberating) — consciously choosing a word from that list reintroduces the exact human bias this whole page is designed to avoid.
4. You now have a complete, correctly-checksummed BIP39 mnemonic, generated entirely from coin flips end to end. The complete seed exists on Jade, and briefly in your handwriting — never on the computer that ran this page.

From there, Jade can use that phrase directly, or — on a **Jade Plus** — [export it as a SeedQR](https://help.blockstream.com/blockstream-jade/use-jade-air-gapped/create-a-seedqr-from-your-recovery-phrase), a way of encoding the mnemonic as a QR code. That SeedQR can be scanned into Jade's **Temporary Signer** mode: a stateless session that Jade forgets the moment it's powered off, letting you sign with that seed without ever adding it as the device's persistent wallet. The original (non-Plus) Jade supports Temporary Signer too, but without the camera-based SeedQR scanning — you'd enter the recovery phrase manually instead.

## Alternative: computing the final word without a hardware wallet

If you don't have a Jade or another hardware wallet that supports entering your own entropy, you can compute the checksummed final word yourself using [iancoleman's BIP39 tool](https://github.com/iancoleman/bip39) — open source (MIT licensed) and, like this page, capable of running fully offline with no network access.

The same air-gap rule applies here, even more so: don't type real coin-flip words into the live [iancoleman.io/bip39](https://iancoleman.io/bip39/) page. Instead, download the standalone build (`bip39-standalone.html`) from its [GitHub releases](https://github.com/iancoleman/bip39/releases), move it to your offline machine alongside this page, and open it there.

Unlike Jade, this tool doesn't offer a "type your words, pick from a restricted list of valid endings" flow — instead it computes a complete mnemonic, including the checksummed last word, from a raw entropy bit string. This page makes reconstructing those bits straightforward:

1. Work through this page as usual for 12-word or 24-word mode, and this time also write down each word's `Binary` value (shown next to the result on the "clear and continue" screen) instead of just the word itself.
2. Concatenate those bit strings in order — that's the first 121 (or 253) bits of your raw entropy, 7 (or 3) bits short of the full 128 (or 256) needed. Flip a coin 7 more times (12-word) or 3 more times (24-word), heads = 1 / tails = 0 as usual, and append those bits too.
3. On the offline `bip39-standalone.html`, paste that full binary string into the **Entropy** field and set its type to **Binary**. The tool derives the complete, correctly checksummed mnemonic from it — the first 11 (or 23) words it shows should exactly match the words you already picked, which is a good sanity check that you transcribed everything correctly — and the final word is the one you needed.

This gets you a valid mnemonic without special hardware, but it's a lower-assurance path than Jade: it's ordinary browser JavaScript with no secure element, so its trustworthiness rests on the code itself rather than dedicated hardware. Also make sure every one of the leftover 7 (or 3) bits in step 2 genuinely came from a coin flip rather than the tool's own "generate random mnemonic" button — using that would reintroduce a software RNG into what's otherwise fully physical entropy.

## Usage

`index.html` is a single, self-contained file with no build step, no dependencies, and no server — it never makes a network request once loaded.

**For anything beyond casual testing, run it on an offline computer.** Download `index.html` (e.g. via GitHub's "Download raw file" button, or `git clone`), move it to a machine that's disconnected from the internet — or better, one that's never been connected — and open it there:

```bash
open index.html
```

or just double-click the file. This matters because the words you generate are meant to be secret; typing or clicking through them on a machine that's online, or on the hosted GitHub Pages version, means trusting that machine and its network connection with that secret. An air-gapped machine removes that risk entirely, which is the same reasoning behind using dice or coin flips for entropy in the first place: minimizing what you have to trust.

The hosted version above is meant for trying the tool out and reading the code that will run on your machine — not for generating words you intend to keep secret.
