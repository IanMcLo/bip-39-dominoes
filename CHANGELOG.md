## v1.0.0 — Initial Release

Domino-draw adaptation of the [BIP39 Dice Roll Seed Generator](https://github.com/IanMcLo/bip-39-dice), generalizing the same exact rejection-sampling construction from a fair 6-sided die (base-6) to a fair, with-replacement domino draw from a standard 28-tile double-six set (base-28).

### Added

- **Zero Modulo Bias (Rejection Sampling):** The base-28 draw integer is mapped to the target $2^b$ space via the same exact-rejection construction as the dice tool: if the integer falls into the remainder zone ($X \ge T$), generation halts and the operator is asked to clear and reshuffle. Accepted outputs are exactly uniform — not merely bias-bounded.
- **Tap-to-Draw Grid Input:** All 28 domino tiles are rendered as tappable buttons (smaller pip value shown first). Every tap is a complete, valid draw by construction, eliminating the free-text validation surface the dice tool needs for its digit-based input.
- **Draw counts per tier**, chosen for a comfortable exact rejection-sampling margin (all tiers between 1-in-200,000 and 1-in-1.4 million):
  * **12 words:** 30 draws (~144 bits raw)
  * **15 words:** 37 draws (~178 bits raw)
  * **18 words:** 44 draws (~212 bits raw)
  * **21 words:** 50 draws (~240 bits raw)
  * **24 words:** 57 draws (~274 bits raw)
- **Live Modulo Bias Audit Terminal:** Same live BigInt math display (N, r, T, X) and ✅/⛔ verdict as the dice tool, gated behind an "Advanced" toggle off by default.
- **Domino-Specific Fairness Diagnostics (Advanced):** A direct translation of the dice tool's 6-face chi-squared test would require 28 categories, needing ≥140 draws for statistical validity — more than any tier collects, which would make it permanently invalid. Two properly-powered tests were built instead: a **pip-value marginal chi-squared test** (7 categories, using both halves of every tile, valid from n≥18) and a **doubles-vs-non-doubles test** (a domino-specific structural check with no dice equivalent, valid from n≥20), alongside a lag-1 autocorrelation test for sequential patterns. All three are informational only and never affect the accept/reject decision.
- **On-Load Self-Tests (Fail-Closed):** SHA-256 KATs, official BIP-39 wordlist hash, draw-to-entropy packing KAT, and 4 official BIP-39 test vectors are all verified on load; the Generate button is disabled if any check fails.
- **Undo Last Draw:** Removes the most recent tap without requiring a full reshuffle and restart.
- **Memory Wipe & Non-Persistence Hardening:** Hard Reset button, burn-after-reading timer, panic-hotkey clipboard clearing, and beforeunload cleanup — carried over from the dice tool's hardening model.

### Verification

- Rejection-sampling threshold, KAT vector, and full entropy→checksum→mnemonic pipeline independently re-derived and verified in Python before being used as the shipped KAT.
- Full logic re-executed end-to-end against the actual extracted script content (not a reimplementation) via a minimal DOM harness: grid rendering, tap/lock behavior, a complete 24-word generation cycle, Clear Draws reset, Undo, tier-switch draw-count preservation, and a genuinely-constructed rejection case (double-six tapped 30 times) all confirmed working correctly, including the critical path where a rejected sequence produces no mnemonic and shows no results.
- Both new statistical diagnostics validated against simulated fair and deliberately-biased draw sequences to confirm they discriminate correctly at their stated thresholds.

*Release File Hash: `c5f5947ff41c0689b252677f1dc37038c6d33b1ee57c74ea1f4ad4c8d0460415`*

