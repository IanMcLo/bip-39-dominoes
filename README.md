# 🁣 BIP39 Domino Draw Seed Generator

A single-file, offline, cryptographically auditable BIP-39 seed phrase generator driven by physical domino draws from a standard 28-tile double-six set.

Adapted from the [BIP39 Dice Roll Seed Generator](https://github.com/IanMcLo/bip-39-dice), generalizing the same exact rejection-sampling construction from a fair 6-sided die (base-6) to a fair, with-replacement domino draw (base-28). Built for security purists: this tool uses **exact rejection sampling** to reduce modulo bias to mathematically **zero**, features on-load Known Answer Tests (KATs) that fail-closed, and includes a live Modulo Bias Audit Terminal with statistical draw-fairness checks tailored to the domino model, so you can verify the math yourself.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![HTML 100%](https://img.shields.io/badge/HTML-100%25-orange)]()

## 🛡️ Security & Features

- **Zero Modulo Bias (Rejection Sampling):** The base-28 draw integer is mapped to the target $2^b$ space. If the integer falls into the remainder zone ($X \ge T$), the tool refuses to generate and asks you to clear and reshuffle.
- **Tap-to-Draw Grid Input:** Every domino draw is entered by tapping one of 28 on-screen tiles (smaller pip value always shown first) rather than typing digits. Every tap is a valid, complete draw by construction — there is no free-text field and no invalid-input state to get wrong.
- **Draw With Replacement, Always:** The security model requires each draw to be independent and uniformly random, exactly like a fair die roll. You draw one tile, tap the matching tile in the grid, **return it to the set, and reshuffle thoroughly** before the next draw. Drawing without replacement uses a different, incompatible probability model and is not supported.
- **Live Modulo Bias Audit Terminal:** A floating action button (🔬) opens a live audit sheet showing the exact BigInt math (N, R, r, T, X) and the live verdict (✅ ACCEPT or ⛔ REJECT) on every tap.
- **Live Domino-Fairness Checks (Advanced):** With the Audit Terminal's "Advanced" toggle enabled, it runs two statistical diagnostics once at least 20 draws are entered: a **pip-value marginal chi-squared test** (7 categories, using both halves of every tile) and a **doubles-vs-non-doubles test** (a domino-specific structural check with no dice equivalent, since ~25% of tiles are doubles under a fair draw), plus a lag-1 autocorrelation test for sequential patterns. A naive 28-category tile-level test was deliberately **not** implemented — it would require ≥140 draws to be statistically valid, more than any tier collects, making it permanently meaningless at these sample sizes. All three checks are diagnostics about the physical dominoes, not the entropy math — **informational only**, gated behind Advanced, and the accept/reject decision is unaffected either way.
- **On-Load Self-Tests (Fail-Closed):** Every page load verifies the SHA-256 implementation, the official BIP-39 wordlist hash, the draw-to-entropy packing, and 4 official BIP-39 test vectors. If any test fails, the Generate button is disabled.
- **Draw Requirements Per Tier:** Draw counts are chosen for a comfortable rejection-sampling safety margin (all tiers land between 1-in-200,000 and 1-in-1.4 million exact rejection probability):
  * **12 words:** 30 draws (~144 bits raw)
  * **15 words:** 37 draws (~178 bits raw)
  * **18 words:** 44 draws (~212 bits raw)
  * **21 words:** 50 draws (~240 bits raw)
  * **24 words:** 57 draws (~274 bits raw)
- **Undo & Correction:** An "Undo Last Draw" button removes the most recent tap without requiring a full reshuffle and restart, for correcting an accidental tap.
- **Memory Wipe:** A dedicated Hard Reset button explicitly nullifies closure-scoped variables and clears the DOM.
- **Mobile-First & Desktop-Ready:** A flex-wrap tap grid sized for touch, responsive bottom sheets, and collapsible status pills.

## 📦 Verification

To ensure the file you downloaded hasn't been tampered with, verify the SHA-256 checksum.

**Option 1: Sidecar file (Linux/macOS)** Download `index.html.sha256` into the same folder and run:

```
sha256sum -c index.html.sha256
# Expected output: index.html: OK
```

**Option 2: Manual Hash** Hash your local `index.html` file using any trusted SHA-256 tool and compare it against the published hash in `index.html.sha256`.

## 💻 Usage

1. **Air-gap your device:** Disconnect from the internet.
2. Open `index.html` in any modern browser.
3. Wait for the green "✅ Wordlist + self-tests verified" pill to appear.
4. Select your desired word count (12–24 words).
5. Draw a tile from a well-shuffled standard 28-tile double-six domino set, tap the matching tile in the grid (smaller pip value first), **return the tile to the set, and reshuffle thoroughly** before drawing again.
6. Tap the 🔬 button to watch the live rejection sampling math; enable "Advanced" to also see the live pip-bias, doubles-proportion, and autocorrelation checks once you've entered 20+ draws.
7. Once you hit the target draw count, tap **Generate Seed**.
8. Write down your phrase, tap **Clear Draws**, and power off the device.

> ⚠️ If you see a "REJECTION SAMPLING TRIGGERED" message, this is expected occasionally (roughly 1-in-200,000 to 1-in-1.4 million depending on word count) and is not an error. Tap "Clear Draws" and start the full sequence again — do not try to fix or partially edit a rejected sequence.

## 📄 License

MIT – use at your own risk. This is security‑critical software.
Review the code, verify the outputs against known test vectors, and **only use on air‑gapped devices**.

