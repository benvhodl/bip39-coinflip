# BIP39 Coin-Flip Entropy Generator

A small, self-contained, offline tool for converting physical coin-flip observations into individual BIP39 words — entirely in your browser, no dependencies, no network.

**[Try it live](https://benvhodl.github.io/bip39-coinflip/)**, or just open [`index.html`](index.html) directly in any browser.

> The computer generates individual BIP39 entropy words, but never constructs the complete mnemonic.

For a 12-word BIP39 mnemonic, this tool generates the first **11** words. For a 24-word BIP39 mnemonic, it generates the first **23** words. The final word is intentionally generated elsewhere, preferably using trusted hardware that supports entering externally generated entropy. **This is not a complete BIP39 mnemonic generator** — that omission is the central design decision behind the project; see [Why the final word is not generated here](#why-the-final-word-is-not-generated-here).

## Security model

The purpose of this project is to minimize what the computer ever needs to know. For each word:

```
physical coin flips
        ↓
current word's entropy bits
        ↓
one BIP39 word
        ↓
user writes the word down
        ↓
current word is cleared from application state
        ↓
next word
```

The application never maintains a collection like:

```
word 1
word 2
word 3
...
word 11/23
```

It does not construct the complete entropy. It does not construct the complete mnemonic. It does not calculate the final word. It does not calculate the BIP39 checksum. It does not derive a BIP32 seed, xprv, xpub, or wallet.

Don't take that further than it actually is, though. The computer temporarily sees the current entropy word and its underlying 11 bits — it never constructs or stores the complete BIP39 mnemonic or complete entropy, but it does necessarily see each word as it's generated, or there'd be nothing to write down. When a word is completed and you confirm you've written it down, the application removes its references to that word and clears its DOM representation. JavaScript cannot guarantee physical RAM erasure; a cleared object is merely eligible for garbage collection, not securely wiped. That distinction matters when deciding how much to trust this over dedicated hardware.

## Entropy modes

### Standard

One physical coin flip produces one raw bit: Heads maps to 1, Tails maps to 0. 11 physical flips produce one BIP39 word.

The page alternates which face you're asked to start each flip on, and which face starts each new word — a mitigation for the real, measured [same-side bias](https://arxiv.org/abs/2310.04153) of physical coin flips. Because each word contains 11 flips — an odd number — this alternating procedure balances the known same-side effect across pairs of words rather than within every individual word. Standard mode is the simplest and fastest mode, but remains subject to physical coin bias and temporal dependence: it does not produce perfectly unbiased bits, and it is not an extractor.

### Von Neumann

(Labeled "Bias-resistant" in the app.) Flips are processed in pairs:

```
HT → 1
TH → 0
HH → discard
TT → discard
```

HH and TT are discarded completely; the next two physical flips form a new pair. A pair produces an output bit only when the two flips differ — with a perfectly unbiased coin, the expected cost is about 4 physical flips per extracted bit, not 2.

Von Neumann extraction removes fixed marginal bias when consecutive observations can reasonably be modeled as independent trials with a stable probability of Heads. It does not eliminate arbitrary temporal correlation, and it cannot fix a compromised physical process — for example, if how you catch or re-place the coin depends on the previous outcome.

| Mode | Physical process | Speed | Bias handling |
| --- | --- | --- | --- |
| Standard | 1 flip → 1 bit | Fast | Reduces some starting-side effects across words, but does not extract bias |
| Von Neumann | 2-flip pairs | Slower | Removes fixed Heads/Tails bias under its assumptions |

For a high-value wallet, Von Neumann mode is the more conservative choice, because it explicitly removes a well-defined class of physical coin bias. That doesn't make it equivalent to a hardware wallet — it's still asking a human to flip a real coin and honestly report the result.

## Physical flipping procedure

1. Use a real physical coin.
2. Use a level, soft landing surface.
3. Flip the coin cleanly and let it land naturally.
4. Do not catch it.
5. Record the actual landing face.
6. If the result is ambiguous, do not guess.

If the coin lands on its edge, leaves the landing surface, is caught, is obscured, or you cannot confidently identify the face, restart the current word rather than guessing. Don't silently treat an ambiguous physical event as a random bit — in Von Neumann mode especially, an ambiguous flip must never be quietly folded into a same-same discard.

## Multiple coins

The application can rotate between multiple physical coins, one per word:

```
Word 1 → Coin 1
Word 2 → Coin 2
Word 3 → Coin 3
Word 4 → Coin 1
...
```

The purpose is source diversity, not additional entropy. Using multiple coins does not multiply the entropy of each flip and does not guarantee independence — it simply avoids making every word depend on the same physical coin. One coin should remain in use for the entire current word.

## How the words are constructed

BIP39 uses 11-bit word indices. There are exactly 2048 words in the canonical English word list, so:

```
11 bits → integer 0–2047 → BIP39 word
```

The implementation uses MSB-first bit ordering (the first flip is the most significant bit). The embedded word list is the canonical BIP39 English list; see [Testing](#testing) for how that was verified.

## Why the final word is not generated here

For a 12-word mnemonic:

```
128 total entropy bits
121 bits  → first 11 words
7 bits    → remaining entropy in the final word
4 bits    → checksum in the final word
```

For a 24-word mnemonic:

```
256 total entropy bits
253 bits  → first 23 words
3 bits    → remaining entropy in the final word
8 bits    → checksum in the final word
```

The final word is not simply a checksum of the previous words — it contains both the remaining entropy bits and the BIP39 checksum, computed from a SHA-256 hash of the complete entropy. This application deliberately stops before that word: the main design goal is to ensure that no single ordinary computer ever needs to hold the complete seed. The first 11 or 23 words can be generated individually from physical entropy on this computer; the remaining entropy bits and checksum are handled separately, by a trusted hardware/device workflow.

## Recommended workflow

**12-word seed:**

```
1. Choose 12-word mode.
2. Choose Standard or Von Neumann.
3. Generate words 1–11 using physical coin flips.
4. Write each word down before continuing.
5. The application stops.
6. Generate the 12th word using trusted hardware / an appropriate offline workflow.
```

**24-word seed:**

```
1. Choose 24-word mode.
2. Choose Standard or Von Neumann.
3. Generate words 1–23 using physical coin flips.
4. Write each word down before continuing.
5. The application stops.
6. Generate the 24th word using trusted hardware / an appropriate offline workflow.
```

The final word must be generated using a trusted device or workflow that explicitly supports importing/entering externally generated entropy — follow the documentation for the specific hardware wallet you're using. As one documented example, the [Blockstream Jade](https://blockstream.com/jade/) supports a [recovery-phrase-from-your-own-entropy flow](https://help.blockstream.com/blockstream-jade/add-more-security-functionality/create-a-recovery-phrase-using-dice); check whichever device you actually have for its equivalent, rather than assuming any given wallet supports this. Do not reconstruct the full entropy on this computer to work around not having such a device, and do not enter the words this page generates into any online service — including the live iancoleman.io/bip39 page — to compute the final word.

## Running offline

The HTML is intentionally self-contained: no build step, no dependencies, no server.

1. Obtain the HTML from a trusted source.
2. Verify its integrity before taking the computer offline (see [Verify the HTML](#verify-the-html)).
3. Disconnect the computer from the network.
4. Close unnecessary applications.
5. Run the HTML locally.
6. Generate the required words.
7. Write them down manually.
8. Close the page/browser when finished.

Incognito/private browsing is not an air gap and does not protect against malware, browser extensions, keyloggers, screen capture, or a compromised operating system. It mainly reduces browser history/cache persistence, which this page doesn't create in the first place — it makes no network requests and uses no storage.

## Verify the HTML

A hash embedded inside the same HTML file cannot prove the HTML itself hasn't been maliciously modified — a modified copy could ship a modified hash right alongside it. For high-value use, verify the downloaded HTML against a SHA-256 hash or signed release obtained through an independent trusted channel before taking the machine offline. No such hash is currently published for this repository.

## What this application does NOT do

- generate the final BIP39 word
- calculate the BIP39 checksum
- reconstruct the complete entropy
- reconstruct the complete mnemonic
- derive a seed
- derive xprv/xpub
- generate wallet addresses
- store generated words
- use browser randomness
- use persistent browser storage
- use the clipboard
- make network requests

This is an important part of the threat model, not an incomplete feature list. The application does not use `Math.random()`, `crypto.getRandomValues()`, or any other browser RNG — the intended entropy source is the physical coin, and the software performs only deterministic recording and conversion of the observed physical outcomes. None of this makes the overall system automatically secure; see [Limitations](#limitations).

## Limitations

**Physical coin bias.** A physical coin may not be perfectly fair. Standard mode does not mathematically eliminate physical bias. Von Neumann mode removes fixed marginal bias under its assumptions but cannot eliminate arbitrary physical or temporal correlations.

**Human procedure.** The user must correctly observe and record each physical result.

**Computer security.** The tool is still running on a computer/browser. An infected or compromised computer could potentially display misleading information, alter the software, capture screen contents, modify the word list, or observe the current word. The HTML should be independently verified and preferably run offline.

**JavaScript memory.** The application clears references to completed words, but JavaScript cannot guarantee physical RAM erasure.

**Physical backups.** The security of the resulting wallet also depends on how the handwritten seed backups are stored afterward.

## Testing

The implementation includes tests for:

- the canonical 2048-word list
- all 2048 possible 11-bit mappings
- MSB-first conversion
- the Standard collector
- Von Neumann extraction
- Von Neumann undo behavior
- reset behavior
- word-completion boundaries
- state clearing
- configuration locking

These tests verify the deterministic software transformation and state handling. They do not prove that a particular physical coin or flipping procedure is random. Run them from the page itself via the "Diagnostics" panel.

## Design philosophy

The project intentionally favors:

- small codebase
- no dependencies
- no network
- no browser RNG
- no persistent storage
- no clipboard
- no complete mnemonic
- one word at a time
- explicit physical entropy
- deterministic transformation
- independently testable BIP39 mapping

The goal is not to build a wallet. The goal is to make the physical-entropy-to-word transformation as small and auditable as practical.

## License

No license file is currently included in this repository.
