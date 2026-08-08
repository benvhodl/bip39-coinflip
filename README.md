# BIP39 Coin-Flip Entropy Generator

A single, self-contained HTML page that turns physical coin flips into the entropy words of a BIP39 mnemonic — entirely offline, entirely in your browser. It deliberately stops one word short of a complete mnemonic; see [Why it stops one word short](#why-it-stops-one-word-short).

**[Try it live](https://benvhodl.github.io/bip39-coinflip/)**, or just open [`index.html`](index.html) directly in any browser.

## What it does

- Lets you choose a **12-word** or **24-word** seed length and an entropy mode up front. Once you press **Begin**, those choices lock for the rest of the session — the page then walks you through generating the first **11** or **23** words respectively — the entropy words — and stops.
- Offers two entropy-generation modes:
  - **Standard** ("simpler / faster") — one coin flip = one bit, 11 flips per word (Heads = 1, Tails = 0).
  - **Bias-resistant** ("recommended for high-value wallets") — a [von Neumann extractor](https://en.wikipedia.org/wiki/Randomness_extractor#Von_Neumann_extractor): flip the coin twice per output bit. Heads→Tails = 1, Tails→Heads = 0, and Heads→Heads / Tails→Tails are discarded and re-flipped. Continues until 11 bits have been extracted for the word.
- Combines each word's bits most-significant-bit first into a number from 0–2047 and looks that number up in the full 2048-word BIP39 English list embedded in the page. That embedded list was diffed, word-for-word, against the canonical BIP39 English wordlist during development, and the entropy→word mapping was independently cross-checked against the official BIP39 test vectors. The page doesn't re-verify itself against either source at runtime, on purpose: it makes no network requests, so it has no way to fetch a canonical copy to compare against, and a self-hash baked into the same file a malicious copy could just as easily forge wouldn't prove much anyway. The real defense is verifying this file itself — reading the code or comparing it against the canonical source — before you trust it with real entropy.
- In Standard mode, alternates which face you're asked to start each flip on (and rotates through your coins, one per word) as a mitigation for the real, measured [same-side bias](https://arxiv.org/abs/2310.04153) of physical coin flips, and asks you to land the coin on a soft surface rather than catching it. This is described honestly as a mitigation, not a fix — because each word uses an odd number of flips (11), the alternation balances out across pairs of words, not within any single word, and it doesn't make bits mathematically independent or identically distributed. The starting-face pattern itself is fully **deterministic** (alternating by word number and flip number) rather than drawn from a random number generator — this page calls no `crypto.getRandomValues`, `Math.random`, or any other software RNG anywhere, for anything. See [Threat model](#threat-model) for the tradeoff that implies.
- If a coin lands on its edge, is caught, leaves the surface, or the result is otherwise unclear, the page's guidance is: don't guess, reset that word's flips and start it over. In Bias-resistant mode an ambiguous flip must never be quietly treated as a same-same discard.
- Shows only one word at a time — just the word itself, not its binary value or list index, so the page holds as little entropy-derived state as possible at any moment. After you write it down and confirm, the page clears that word's flip data and the word itself from both memory and the DOM before moving to the next one — there is never a growing list of words on screen, and no array anywhere in the code holds the complete sequence. To be precise about what "clears" means: the page drops its references and clears the DOM; JavaScript can't guarantee physical RAM erasure.
- Never uses localStorage, sessionStorage, cookies, IndexedDB, or a service worker, and makes no network requests.
- Ships a self-test suite runnable from the page itself via the "Diagnostics" panel, using synthetic test vectors only — it never touches real flip data. It checks the word list's internal consistency, an exhaustive bit→word mapping across all 2048 possible values (not just a few sample points), the von Neumann extractor's logic including exact single-flip undo, input validation on every pure function, and state invariants (bit counts never exceed 11, word counts never exceed the target, and so on).

## Why it stops one word short

The last word of a BIP39 mnemonic isn't simply a checksum of the earlier words — it's a mix of leftover entropy bits and checksum bits, computed from a SHA-256 hash of *all* the preceding entropy. For a 12-word mnemonic, the first 11 words carry 121 of the 128 entropy bits, and the 12th word packs in the remaining 7 entropy bits plus a 4-bit checksum. For a 24-word mnemonic, the first 23 words carry 253 of the 256 entropy bits, and the 24th word packs in the remaining 3 entropy bits plus an 8-bit checksum. This page contains no checksum code at all: not a disabled feature, just logic that was never written. There is no button, mode, or hidden state anywhere in this file that computes a final word, reconstructs full entropy, derives a seed, or builds an extended key.

```
physical coin
     |
this computer  -->  entropy words only  -->  you write them down
     |
computer clears its reference to each word once you confirm it
     |
final word (remaining entropy + checksum) generated elsewhere
     |
complete mnemonic assembled only outside this computer
```

The point is to minimize how much wallet-secret material a single computer ever holds. Even a fully trustworthy machine today could be compromised later; a machine that never held the complete seed can't leak it. You get the final word from a trusted hardware device, or another explicitly trusted offline method — see the two workflows below.

## Threat model

- **Trusted:** the user's physical coin flips, as the entropy source. The application assumes the user obtains entropy from physical coin flips; it does not attempt to prove that this physical entropy source is unbiased or that successive flips are independent. Standard mode reduces two specific, measured biases — same-side-start bias and catch-based bias — via the alternating-start-face convention and the land-don't-catch instruction, but that starting-face display is a toss procedure, not itself an entropy bit; the only entropy input is the physical landing face you report. Bias-resistant mode instead removes any fixed-but-unknown per-flip bias mathematically, via the von Neumann extractor, under the assumption that consecutive flips are independent trials; it does not guarantee immunity to arbitrary correlations between flips (e.g. if how you catch or re-place the coin depends on the previous outcome).
- **Trusted, narrowly: the browser.** The browser is trusted only to faithfully transform the bits you report into BIP39 output — combining them into an index and looking that index up in the embedded word list — without altering the values you enter. This page uses **no software randomness whatsoever**: the deterministic start-face pattern in Standard mode is computed from the word and flip number alone, not from any RNG. The tradeoff is that this pattern is predictable to anyone who reads the code — but that predictability can't inject bias into a bit's value, since every bit always comes from whatever the physical coin actually showed you. It just means Standard mode's starting-face mitigation falls back to the baseline physical bias if you consider that pattern compromised, rather than staying hidden behind a random seed.
- **Not trusted, not needed: the network.** Once loaded, the page makes no network requests and stores nothing (no localStorage, cookies, or history). The one exception is the hosted GitHub Pages copy itself, which must be trusted once, at load time, to deliver unmodified code — see [Usage](#usage) below for why running an offline copy avoids that.
- **Out of scope:** whether any specific physical coin is fair, whether the person flipping it introduces bias, and anything about the entropy quality of individual coin flips beyond the mitigations described above. Also out of scope, by design: the final word, and anything downstream of it (seed derivation, key derivation, wallet creation).

## Generating the final word with a hardware wallet

This page only ever produces the entropy words — 11 of 12, or 23 of 24 — and deliberately can't produce a complete, checksummed mnemonic on its own (see [Why it stops one word short](#why-it-stops-one-word-short)). One way to turn its output into a complete, valid seed phrase is to pair it with a hardware wallet that supports entering your own manually-generated entropy, such as the [Blockstream Jade](https://blockstream.com/jade/):

1. Pick 12-word or 24-word mode and Standard or Bias-resistant mode on this page, press Begin, and work through the flips it asks for, writing each entropy word down on paper as you go — the page shows one word at a time and never a growing list. It tells you when it's done — 11 of 11 or 23 of 23 — and stops there on purpose.
2. On Jade, use its [recovery-phrase-from-your-own-entropy flow](https://help.blockstream.com/blockstream-jade/add-more-security-functionality/create-a-recovery-phrase-using-dice) (Blockstream's guide uses dice as the example, but any true-random source — including coin flips — works identically) and enter your 11 or 23 words.
3. Jade can't hand you back a single specific final word, because part of it is still genuine unpicked entropy; instead it restricts the keyboard to only the words that produce a valid checksum, and either lets you choose or offers to "randomly select a final word for you." Take the random option (or otherwise pick without deliberating) — consciously choosing a word from that list reintroduces the exact human bias this whole page is designed to avoid.
4. You now have a complete, correctly-checksummed BIP39 mnemonic, generated entirely from coin flips end to end. The complete seed exists on Jade, and briefly in your handwriting — never on the computer that ran this page.

From there, Jade can use that phrase directly, or — on a **Jade Plus** — [export it as a SeedQR](https://help.blockstream.com/blockstream-jade/use-jade-air-gapped/create-a-seedqr-from-your-recovery-phrase), a way of encoding the mnemonic as a QR code. That SeedQR can be scanned into Jade's **Temporary Signer** mode: a stateless session that Jade forgets the moment it's powered off, letting you sign with that seed without ever adding it as the device's persistent wallet. The original (non-Plus) Jade supports Temporary Signer too, but without the camera-based SeedQR scanning — you'd enter the recovery phrase manually instead.

## Alternative: computing the final word without a hardware wallet

If you don't have a Jade or another hardware wallet that supports entering your own entropy, you can compute the checksummed final word yourself using [iancoleman's BIP39 tool](https://github.com/iancoleman/bip39) — open source (MIT licensed) and, like this page, capable of running fully offline with no network access.

The same air-gap rule applies here, even more so: don't type real coin-flip words into the live [iancoleman.io/bip39](https://iancoleman.io/bip39/) page. Instead, download the standalone build (`bip39-standalone.html`) from its [GitHub releases](https://github.com/iancoleman/bip39/releases), move it to your offline machine alongside this page, and open it there.

Unlike Jade, this tool doesn't offer a "type your words, pick from a restricted list of valid endings" flow — instead it computes a complete mnemonic, including the checksummed last word, from a raw entropy bit string. Earlier versions of this page displayed each word's binary value and list index to make reconstructing that bit string easy; the current version deliberately doesn't, so that it holds as little entropy-derived state as possible at any moment. This workflow still works, it's just manual now:

1. Work through this page as usual for 12-word or 24-word mode, writing down each entropy word.
2. For each word, look up its position (0-indexed) in the canonical BIP39 English wordlist and convert that position to an 11-bit binary string, most-significant-bit first (e.g. `abandon` is index 0 → `00000000000`; `zoo` is index 2047 → `11111111111`). Any calculator's "convert to binary" function works, or do it by hand. Concatenate those bit strings in word order — that's the first 121 (12-word) or 253 (24-word) bits of your raw entropy, 7 or 3 bits short of the full 128 or 256 needed. Flip a coin 7 more times (12-word) or 3 more times (24-word), heads = 1 / tails = 0 as usual, and append those bits too.
3. On the offline `bip39-standalone.html`, paste that full binary string into the **Entropy** field and set its type to **Binary**. The tool derives the complete, correctly checksummed mnemonic from it — the first 11 (or 23) words it shows should exactly match the words you already picked, which is a good sanity check that you transcribed everything correctly — and the final word is the one you needed.

This gets you a valid mnemonic without special hardware, but it's a lower-assurance path than Jade in two ways: it's ordinary browser JavaScript with no secure element, so its trustworthiness rests on the code itself rather than dedicated hardware; and the manual word→binary lookup in step 2 is an easy place to introduce a transcription error, which is exactly what the sanity check in step 3 is for. Also make sure every one of the leftover 7 (or 3) bits in step 2 genuinely came from a coin flip rather than the tool's own "generate random mnemonic" button — using that would reintroduce a software RNG into what's otherwise fully physical entropy.

## Usage

`index.html` is a single, self-contained file with no build step, no dependencies, and no server — it never makes a network request once loaded.

**For anything beyond casual testing, run it on an offline computer.** Download `index.html` (e.g. via GitHub's "Download raw file" button, or `git clone`), move it to a machine that's disconnected from the internet — or better, one that's never been connected — and open it there:

```bash
open index.html
```

or just double-click the file. This matters because the words you generate are meant to be secret; typing or clicking through them on a machine that's online, or on the hosted GitHub Pages version, means trusting that machine and its network connection with that secret. An air-gapped machine removes that risk entirely, which is the same reasoning behind using dice or coin flips for entropy in the first place: minimizing what you have to trust.

The hosted version above is meant for trying the tool out and reading the code that will run on your machine — not for generating words you intend to keep secret.
