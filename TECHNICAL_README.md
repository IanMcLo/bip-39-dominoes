# Technical & Cryptographic Specification

This document describes the information-theoretic foundations, entropy bounds, framing pipeline, and security model of the `bip-39-dominoes` physical seed generator. It is intended as a precise, implementable reference for auditors and advanced users.

---

## 1. Information-Theoretic Foundations

### 1.1 Physical Entropy Source and Uniform Tile Mapping

The physical entropy source is repeated independent draws from a standard double-six domino set (28 unique tiles) with replacement and reshuffling. Each tile is uniquely identified by its canonical pip pair `(i, j)` where `0 ≤ i ≤ j ≤ 6`, yielding exactly 28 distinct outcomes per draw.

Shannon entropy per draw:

$$H(X) = -\sum_{x \in \{0,\dots,27\}} P(X=x) \log_2 P(X=x) = -28 \times \frac{1}{28} \log_2\left(\frac{1}{28}\right) = \log_2 28 \approx 4.8074 \text{ bits/draw}$$

Min-entropy per draw (worst-case single-trial predictability):

$$H_\infty(X) = -\log_2\left(\max_x P(X=x)\right) = -\log_2\left(\frac{1}{28}\right) = \log_2 28 \approx 4.8074 \text{ bits/draw}$$

Because Shannon entropy and min-entropy are equal under a uniform distribution, each draw contributes approximately 4.807 bits of entropy in both average and worst-case senses prior to software processing.

### 1.2 Canonical Tile Ordering

To ensure each physical tile maps to exactly one digital symbol, tiles are encoded with the **smaller pip first**: the tile showing pips 3 and 5 is recorded as `3|5`, not `5|3`. Doubles (e.g., `2|2`) are unaffected by this rule. This normalization preserves the full log2(28) bits per draw without introducing orientation-dependent non-uniformity.

The 28 tiles are indexed 0–27 in row-major order: for tile `(i, j)` with `i ≤ j`, the index is:

$$\text{index}(i, j) = 7i - \frac{i(i-1)}{2} + (j - i)$$

This yields: `(0|0)=0, (0|1)=1, ..., (0|6)=6, (1|1)=7, (1|2)=8, ..., (6|6)=27`.

---

## 2. Framing, Checksum and Word Slicing

### 2.1 Raw Entropy Lengths

BIP-39 requires raw entropy lengths E that are multiples of 32 bits. Standard choices and corresponding checksum lengths are:

| Mnemonic Length | Raw Entropy $E$ | Checksum bits $CS = E / 32$ | Total bits ($E + CS$) |
|---|---|---|---|
| 12 words | 128 | 4 | 132 |
| 15 words | 160 | 5 | 165 |
| 18 words | 192 | 6 | 198 |
| 21 words | 224 | 7 | 231 |
| 24 words | 256 | 8 | 264 |

### 2.2 Checksum Derivation (SHA-256)

Compute the SHA-256 digest over the raw entropy byte array:

$$\text{Digest} = \text{SHA-256}(\mathrm{RawEntropy}_{\mathrm{bytes}})$$

Take the first $CS$ bits of the Digest as the checksum bits:

$$\text{Checksum} = \text{Digest}[0 : CS - 1]$$

Concatenate the raw entropy bitstring and the checksum bits to form the full bitstream $S$ of length $L = E + CS$.

### 2.3 11-bit Word Indices

Partition $S$ into contiguous 11-bit chunks and interpret each chunk in MSB-first order to produce the BIP-39 word indices:

$$W_k = \sum_{i=0}^{10} S[11k + i] \times 2^{10-i}$$

where $W_k \in [0, 2047]$ indexes the BIP-39 English wordlist.

---

## 3. Empirical vs. Theoretical Min-Entropy

Short sample sizes and the chosen measurement resolution can create apparent reductions in empirical min-entropy that do not reflect a true loss of cryptographic strength.

### 3.1 Single-Bit Variance Example (160 bits)

A uniformly random 160-bit string has expected number of ones $\mu = 80$ and standard deviation $\sigma \approx 6.32$. Observing 82 ones corresponds to $z \approx 0.316$, well within typical statistical fluctuation.

### 3.2 Byte-Level Sampling Artifacts

When viewing $N = 20$ bytes (160 bits) as 8-bit symbols, the maximum empirical frequency for a distinct byte value is $1/20 = 0.05$. A naive calculation using that per-byte frequency can under-estimate entropy for short samples. Use bitwise statistics or aggregate many samples before inferring entropy degradation.

---

## 4. Downstream Key-Derivation Architecture

After generating and verifying the mnemonic, standard wallet software expands the mnemonic into a seed using PBKDF2-HMAC-SHA512 as specified by BIP-39:

$$\text{Seed} = \text{PBKDF2-HMAC-SHA512}(\text{Mnemonic},\ \text{"mnemonic"} \parallel \text{Passphrase},\ 2048,\ 512)$$

This KDF both stretches and mixes the mnemonic (and optional passphrase), producing 512 bits of master seed material. The extracted seed's security cannot exceed the entropy in the original mnemonic; thus the dominant security parameter is the raw entropy length $E$.

---

## 5. Bounded Rejection Sampling: Eliminating Modulo Bias Exactly

Modulo bias occurs when a total number of outcomes ($N$) is mapped into a target range ($R$) via a modulo operation, and $N$ is not perfectly divisible by $R$. **This implementation eliminates bias entirely** by rejecting any draw sequence that would land in the biased zone.

### a. Defining the Spaces

Let $k$ be the number of draws. The total number of unique base-28 outcomes is:

$$N = 28^k$$

Let $b$ be the target number of bits. The target range is:

$$R = 2^b$$

Let $q = \lfloor N/R \rfloor$ and $r = N \bmod R$. Under naive modulo reduction, $r$ buckets would receive $q+1$ values and $R-r$ buckets would receive $q$ values — this is the source of the bias.

### b. The Rejection Boundary

Define the **uniformity boundary**:

$$T = N - r = N - (N \bmod R)$$

Let $X$ be the integer formed by treating the draws as base-28 digits (first draw = most significant digit, digit = tile index 0–27).

- **If $X \ge T$**: **reject**. The tool refuses to generate and asks the operator to clear and re-draw.
- **If $X < T$**: **accept**. Compute the output as $X \bmod R$.

Because every accepted output is drawn from an integer range of exactly $T$, which is an exact multiple of $R$, **every one of the $R$ output buckets receives exactly $T/R$ accepted inputs — mapping is perfectly uniform**. There is no residual bias; it is removed by construction.

### c. Rejection Probability Per Tier

The rejection probability $P(\text{reject}) = r/N$ depends only on the draw count $k$ and target bits $b$, and is exactly computable:

| Words | Draws ($k$) | Target ($b$) | Raw bits $H = k\log_2 28$ | Buffer ($H-b$) | Exact $P(\text{reject}) = r/N$ |
|---|---|---|---|---|---|
| 12 | 30 | 128 | 144.221 | 16.221 | 0.000334% (1 in 299,074) |
| 15 | 37 | 160 | 177.872 | 17.872 | 0.000302% (1 in 330,783) |
| 18 | 44 | 192 | 211.524 | 19.524 | 0.0000705% (1 in 1,418,990) |
| 21 | 50 | 224 | 240.368 | 16.368 | 0.000497% (1 in 201,384) |
| 24 | 57 | 256 | 274.019 | 18.019 | 0.000252% (1 in 397,180) |

---

## 6. Physical Drawing Protocol

The operator follows this protocol for each draw:

1. **Place** all 28 tiles face-down in a pile or spread
2. **Shuffle** thoroughly (mix tiles to break positional correlation)
3. **Draw** one tile at random without looking at the face
4. **Record** the tile using canonical ordering (smaller pip first): e.g., a tile showing 5 and 3 is recorded as `3|5`
5. **Replace** the tile back into the set
6. **Reshuffle** before the next draw
7. **Repeat** until the required draw count is reached

This protocol ensures each draw is independent and uniformly distributed over the 28 tiles.

---

## 7. Live Statistical Diagnostics: Chi-Squared and Autocorrelation Checks

Sections 5–6 establish that the entropy math is exactly uniform provided each draw is truly independent and fair. The checks in this section address whether that precondition holds for a given set of dominoes — they are diagnostics **about the operator's physical tiles**, not about the entropy conversion, and they have no influence on the accept/reject decision in Section 5b.

Both checks live in the Bias Audit Terminal, behind the "Show advanced values" toggle, and update live once the draw count reaches the minimum sample size.

### a. Chi-Squared Goodness-of-Fit Test (Pip Values)

A tile-level chi-squared test (df=27) would require expected counts ≥ 5 per bin → minimum 140 draws, which exceeds every tier. Instead, this tool tests the **individual pip values** (0–6) across both ends of drawn tiles.

For $n$ draws, each tile contributes 2 pip observations, yielding $2n$ total observations across 7 pip values. Let $O_p$ be the observed count of pip value $p \in \{0,\dots,6\}$ and $E = 2n/7$ the expected count under uniform tiles. The test statistic is:

$$\chi^2 = \sum_{p=0}^{6} \frac{(O_p - E)^2}{E}$$

This is compared against the critical value for $\text{df} = 6$ at $\alpha = 0.05$: $\chi^2_{\text{crit}} = 12.592$. $\chi^2 > \chi^2_{\text{crit}}$ is shown as ⚠️; otherwise ✔️.

**Minimum sample size:** the chi-squared approximation requires $E \ge 5$, i.e. $2n/7 \ge 5 \Rightarrow n \ge 18$. Below 18 draws the terminal reports insufficient data.

### b. Lag-1 Autocorrelation Test

The chi-squared test checks only the marginal distribution of pip values. To detect sequential correlation (e.g., a tile that systematically follows another), the terminal computes the lag-1 autocorrelation coefficient on the tile index sequence.

For draws $v_1, \dots, v_n$ with mean $\bar{v}$:

$$r = \frac{\sum_{i=1}^{n-1} (v_i - \bar{v})(v_{i+1} - \bar{v})}{\sum_{i=1}^{n} (v_i - \bar{v})^2}$$

Significance is assessed via the Fisher $z$-transform:

$$z = \tanh^{-1}(r) \cdot \sqrt{(n-1) - 3}$$

$|z| > 1.96$ (two-tailed $\alpha = 0.05$) is shown as ⚠️; otherwise ✔️. This shares the same $n \ge 18$ minimum.

### c. Scope and Limitations

- Both checks are advisory only and are gated behind the opt-in toggle since they are computed from the operator's actual draws.
- The lag-1 test detects correlation between adjacent draws only. Patterns at lag 2 or higher would not be caught.
- Neither check can distinguish biased tiles from an operator who is not drawing independently.

---

## 8. Implementation & Interoperability Notes

- **Byte/bit ordering:** The first recorded draw is the most significant base-28 digit (MSB-first). The accumulated base-28 integer is reduced via bounded rejection sampling and converted to big-endian hex, zero-padded to the target byte length.

- **Rejection Sampling, Not Truncation:** The implementation performs the accept/reject test before reduction. On rejection, the UI raises `⛔ REJECTION SAMPLING TRIGGERED` and disables generation; the operator must clear and re-draw.

- **Elevated Draw Counts:** Minimum draw counts (30 for 12 words up to 57 for 24 words) provide the raw-entropy buffer that keeps rejection rare across all tiers (worst case roughly 1 in 201,384).

- **Canonical Tile Ordering:** Tiles are recorded with the smaller pip first. The UI enforces this: if the operator enters `5|3`, it is flagged as invalid with the message "enter smaller pip first." This prevents orientation-dependent encoding errors.

- **Input Methods:** The primary input is a tap-grid of 28 buttons (one per tile). A secondary text input accepts two-digit codes (e.g., `35` for the 3|5 tile) for paste or rapid entry. Both paths feed the same base-28 accumulator.

- **Verification:** When verifying outputs with third-party tools, paste the displayed raw hex entropy. Re-entering raw draws into tools that assume a different tile-to-byte ordering, a different rejection strategy, or lower draw counts will produce a different entropy value.

- **Cryptographic Primitives:** Mnemonic key derivation uses PBKDF2-HMAC-SHA512 per BIP-39. SHA-256 for wordlist verification is provided natively by `window.crypto.subtle` with a pure-JS fallback for offline `file://` access.

---

## 9. Threat Model & Operational Security

This section summarises the physical and operational assumptions and gives guidance for safe use.

- **Assumptions:** The generator assumes tiles are drawn by an honest operator in a physically private environment and that the recording medium is under the operator's control during generation.

- **Threats considered:** Shoulder-surfing, biased or tampered tiles, accidental leakage via networked devices, operator error when re-entering data, and residual secret material left in the clipboard or Audit Terminal.

- **Operational recommendations:**

  * Draw and record in private; remove cameras, disable microphones, and avoid network-connected devices in the immediate area.
  * Use a standard, undamaged double-six domino set from a reliable source. The tool automates pip-value chi-squared and lag-1 autocorrelation checks once 18+ draws are entered — treat a flagged result as a prompt to inspect your tiles or improve your shuffling technique.
  * Shuffle thoroughly between draws. Inadequate shuffling introduces sequential correlation, which the autocorrelation test will detect.
  * Prefer an air-gapped computer for converting draws to entropy. If a device is used, verify the artifact's checksums prior to use.
  * Do not paste raw entropy or the mnemonic into online web pages. When verification is required, transfer only the raw entropy hex using an offline method.
  * Treat raw entropy hex and the mnemonic as highly sensitive. The tool attempts to clear the clipboard automatically ~45 seconds after a Copy action, and immediately on "Clear" or "Hard Reset" — but this is a best-effort clear, not a guaranteed secure erasure.
  * The Bias Audit Terminal hides sequence-derived values behind an explicit "Show advanced values" toggle, off by default, since those values are computed from the operator's actual draws and leak partial information if left visible on screen or captured in a screenshot.

- **Out of scope:** Supply-chain compromises of cryptographic libraries, OS-level compromise of the recording device, and coercion attacks against the operator.

---

## 10. Common Pitfalls and How to Avoid Them

- **Re-typing draws into other tools:** Always verify by using the displayed raw entropy hex rather than re-typing draw sequences.

- **Byte/bit ordering mismatch:** This project treats the first recorded draw as the most significant base-28 digit and outputs big-endian hex. Other tools may use different conventions. Use the displayed hex when cross-checking.

- **Tile orientation confusion:** Always record the smaller pip first. The UI enforces this and flags reversed pairs. If you find yourself unsure which way you held a tile, the convention resolves it: `3|5`, never `5|3`.

- **Insufficient shuffling:** Poor shuffling introduces sequential correlation between draws. If the autocorrelation test flags your sequence, improve your shuffling technique (e.g., spread tiles on a table and mix them physically before each draw).

- **Insufficient sample size:** The UI enforces the current minimums: 30 draws for 12 words, up to 57 draws for 24 words — the tool will not allow generation below the required count.

- **Exposing the mnemonic:** Never paste the mnemonic into networked web pages. Use local, audited tools for any additional verification.

---

## 11. Trusted Code Base & Audit Checklist

For auditors and advanced users, verify the following before using this tool in a threat-sensitive workflow:

- Identify the implementation files for entropy collection and conversion (draw parsing, base-28 → BigInt, rejection-sampling gate, integer → byte array, checksum, and word slicing). Confirm these are the only code paths used during generation.

- Confirm the sources and versions of cryptographic primitives (Web Crypto API, and the name/version of any bundled pure-JS SHA-256 implementation).

- Confirm the on-load self-test suite runs and fails closed: the wordlist SHA-256 hash, the four official BIP-39 known-answer vectors, and the draws→entropy known-answer test should all pass before the Generate button is enabled.

- Reproducible builds: publish artifact checksums and provide build instructions so auditors can reproduce release artifacts and verify integrity.

Suggested audit checklist:

- Confirm the implementation files for draws → entropy → hex → mnemonic and list their paths.
- Confirm SHA-256 and PBKDF2 implementations and their origins/versions.
- Confirm the rejection-sampling boundary ($T = N - (N \bmod R)$) is computed and enforced before any modulo reduction, and that rejection surfaces a visible error.
- Confirm the Section 7 chi-squared and autocorrelation diagnostics are read-only: verify neither their computed statistics nor their pass/fail state feeds back into the accept/reject decision.
- Reproduce the worked example in Section 12 in an air-gapped environment.
- Verify there are no network calls, telemetry, or remote loading in the artifact.
- Verify release artifact checksums/signatures.

---

## 12. Worked Example — 12 words (30 draws)

This worked example demonstrates how a 30-draw sequence maps to raw entropy hex and a BIP-39 mnemonic under the current bounded-rejection-sampling implementation. These figures are the tool's own verified Known Answer Test (`DRAWS_KAT`) and reproduce exactly on every page load.

### Example Parameters

- **Target Mnemonic:** 12 words ($b = 128$ bits, 16 bytes)
- **Draw Sequence:** 30 draws (first draw = `1|1`, remaining 29 draws = `0|0`)

### Step-by-Step Conversion

1. **Map Draws to Base-28 Digits:**

   - First draw (`1|1`): index 7 (from the canonical ordering formula)
   - Remaining 29 draws (`0|0`): index 0 each

2. **Calculate the Base-28 BigInt ($X$) and the Rejection Boundary:**

   $$N = 28^{30} = 25{,}986{,}090{,}120{,}790{,}645{,}892{,}257{,}018{,}950{,}637{,}850{,}957{,}185{,}024$$
   $$R = 2^{128} = 340{,}282{,}366{,}920{,}938{,}463{,}463{,}374{,}607{,}431{,}768{,}211{,}456$$
   $$r = N \bmod R = 86{,}888{,}506{,}259{,}191{,}412{,}953{,}679{,}503{,}439{,}721{,}136{,}128$$
   $$T = N - r = 25{,}986{,}003{,}232{,}284{,}386{,}700{,}844{,}065{,}271{,}134{,}411{,}236{,}048{,}896$$
   $$X = 7 \times 28^{29} = 6{,}496{,}522{,}530{,}197{,}661{,}473{,}064{,}254{,}737{,}659{,}462{,}739{,}296{,}256$$

3. **Check the Rejection Condition:**

   $$X < T \implies \textbf{ACCEPT}$$

4. **Reduce via Modulo:**

   $$X \bmod R = 191{,}863{,}310{,}025{,}267{,}084{,}970{,}107{,}179{,}575{,}814{,}389{,}760$$

   **Raw Entropy Hex:** `90578786cdbfe20f4400000000000000`

5. **Checksum & Word Index Generation:**

   - Compute $\text{SHA-256}(\text{Raw Entropy})$ to extract the 4-bit checksum: `0100`
   - Append checksum to raw entropy to form the 132-bit bitstream $S$.
   - Split $S$ into 11-bit MSB-first chunks to map to BIP-39 wordlist indices.

6. **Verification Output:**

   - **Raw Entropy Hex:** `90578786cdbfe20f4400000000000000`
   - **Generated Mnemonic:** `motion rotate ticket opinion wreck amateur avoid abandon abandon abandon abandon above`

This exact draw sequence, hex output, and mnemonic are checked automatically as part of the on-load self-test suite — if you reproduce a different result, the implementation you are auditing has diverged from this specification.

---

## Acknowledgements & References

- BIP-39: <https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki>
- Ian Coleman BIP39 tool: <https://iancoleman.io/bip39/>
- SHA-256, PBKDF2 specifications (NIST and RFC references)
