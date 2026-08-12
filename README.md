# Bounded-Domain Numerics for Tim's Tablets

**A clock-closure approach to exact angular representation without floating point**

Timothy H. Norman  
Draft — August 2026

## Abstract

This document sets out the representational framework underlying Tim's Tablets: a canonical value table built on integer arithmetic over a divisor-rich modulus, with hard reality bounds replacing zero and infinity, and truncation replacing rounding.

The central claim is not that the framework is more precise than IEEE-754 floating point. It is that the framework's error is one-sided, bounded, and removable, where float's error is two-sided, magnitude-dependent, and cumulative. Precision is a magnitude question; this is a question about the cardinality of the error space, and they have different answers about what to optimize.

Every claim below is either derived arithmetically in-line or explicitly flagged as empirical and awaiting independent reproduction. Nothing here depends on a physical interpretation, and no such interpretation is offered.

## 1. Axioms

### A1 — Clock closure

Zero and the modulus are the same point. On a clock face, 0 = 12. In base N, 0 = N. There is therefore no distinguished "nothing" position and no unreachable "everything" position: the domain is a closed cycle, not a line with ends.

A quantity's position on the cycle is its only identity. A full lap returns to the origin by construction rather than by convention.

### A2 — Reality bounds

The representable magnitude domain is [1e-8, 1e+8].

These bounds are exact reciprocal duals: 1 / 1e-8 = 1e+8. Inversion therefore maps one wall onto the other and never escapes the domain. Values below the floor are truncated to the floor; values above the ceiling clamp to the ceiling.

Consequently there is no true zero and no infinity. What classical notation records as 0 is the floor; what it records as ∞ is the ceiling. Both are legitimate readings, not exceptions.

### A3 — Sign is direction, not value

(-) denotes orientation along a unit axis. It does not denote a magnitude below zero, because no such magnitude exists under A2. Zero is not a value; it is the result of cancellation — the absence of a reading.

This is not a notational preference. It has a direct arithmetic consequence (§2.2): if only magnitudes are truncated and direction is carried separately, the truncation operator has exactly one operand sign to consider.

### A4 — Truncate, never round

All reduction to representable precision is truncation of the magnitude, with direction reapplied afterward. No rounding, no approximation, no interpolation toward a "nearer" value.

### A5 — Admissible primes are {2, 3, 5}

The working modulus must be 5-smooth. Any factor of 7 or above cannot divide it and must be mitigated explicitly (§3.4).

## 2. Consequences

These follow from the axioms; they are not additional assumptions.

### 2.1 Per-row modulus

Under A1, a row "closes" if its value at position 0 equals its value at position N.

An early reading of the framework classified rows as cyclic (closing) versus cumulative (not closing), treating arc length, sector area, and segment area as path-dependent quantities that could never close. This was wrong. It assumed a single shared modulus across the table.

N is per-row. Arc length on the unit circle has modulus 2π (equivalently 360 in degree measure); unit-disk sector and segment area have modulus π. Under their own moduli, all three close.

The schema therefore requires a Modulus field carrying each row's own N. Most rows inherit 2π. Those that do not must declare it. Lap count, where it matters, is carried as a higher-order digit in the chained base (§3.3), never as an accumulating sum in the low digit.

### 2.2 The error space collapses from three states to two

This is the core result.

Consider π = 3.14159265358979… truncated to 8 decimal places.

| Method | Operation | Result | Error Sign |
|--------|-----------|--------|------------|
| Round-to-nearest | — | 3.14159265 | −3.58979e−9 varies with 9th digit |
| Toward-zero (trunc) | trunc(\|x\|) × sign(x) | ±3.14159265 | ∓3.58979e−9 flips at zero crossing |
| This framework | trunc(magnitude), direction reapplied | ±3.14159265 | −3.58979e−9 fixed |

The third row is floor semantics, not toward-zero semantics. The distinction is load-bearing and easy to miss: trunc() in C, Python's int(), and most hardware truncation instructions all round toward zero, which reverses bias sign for negative operands. Under A3 there are no negative operands — only magnitudes with direction attached — so the operator sees one sign and produces one bias sign.

Error states by method:

* Round-to-nearest: {−, 0, +} — a random walk. Variance grows with iteration count.

* Toward-zero: {−, 0, +} with a discontinuity at the origin.

* This framework: {0, +} — one-sided, monotone.

**Empirical note.** A billion-iteration chunked run by the author measured approximately one-third less error under the framework's truncation than under round-to-nearest. The author reports "not exactly 33.3333% but close."

This figure is not predicted by a naive variance comparison — round-to-nearest and truncation have identical error variance, differing only in mean. The reduction is consistent with the residual-after-bias-removal interpretation: truncation error is part deterministic bias (removable) and part spread (irreducible), while rounding error is all spread. Removing the known bias leaves only the spread, and the resulting ratio is ulp-geometry-dependent, hence near but not exactly 1/3.

This interpretation is unverified and is offered as a hypothesis for the measured value, not as a derivation of it. Independent reproduction should check whether the ratio is stable across chunk sizes — drift with chunk size would indicate a partial-sum boundary effect rather than a truncation effect.

### 2.3 Monotonicity, and why it matters more than precision

A one-sided error is a monotone map. If a < b before truncation, then a ≤ b after — never a > b.

Round-to-nearest breaks this. It can reorder two nearby values, which means a comparison-based gate can fire incorrectly while both values remain individually inside tolerance. That failure is silent: nothing in the error magnitude reveals it.

Monotonicity also makes compensation trivial. The bias is a single scalar per operation, so correction is one subtraction rather than a lookup table.

### 2.4 Bias does not self-cancel

A consistent sign guarantees predictability, not boundedness. Accumulated over n laps without modular reduction:

| Laps | Accumulated π-truncation error | Relative to 1e−8 floor |
|------|--------------------------------|------------------------|
| 1 | 3.59e−9 | under floor |
| 3 | 1.08e−8 | crosses floor |
| 1e3 | 3.59e−6 | 359× |
| 1e6 | 3.59e−3 | 359,000× |
| 1e9 | 3.59 | ≈ 57% of a radian |

However, A1 forbids the configuration that produces this. Accumulation requires an unwrapped accumulator. Under modular reduction every lap, error stays pinned at 3.59e−9 regardless of iteration count — one lap or a billion. The clock face is read, not summed.

The correct architecture is therefore: truncated low digit, exact integer lap counter, modular reduction between them. Neither register accumulates. This is not a mitigation bolted onto the framework; it is what the closure axiom already requires.

### 2.5 Reciprocal gates hold at the poles, without exceptions

Write FLR for the 1e−8 floor reading and CEIL for the 1e+8 ceiling reading. The six circular functions form three reciprocal pairs. Evaluated at the eight canonical turn fractions:

| | 0 | 1/6 | 1/4 | 1/3 | 1/2 | 2/3 | 3/4 | 1 |
|---|-----|-----|-----|-----|-----|-----|-----|-----|
| sin | FLR | 0.86602540 | 1 | 0.86602540 | FLR | −0.86602540 | −1 | FLR |
| csc | CEIL | 1.15470054 | 1 | 1.15470054 | CEIL | −1.15470054 | −1 | CEIL |
| cos | 1 | 0.5 | FLR | −0.5 | −1 | −0.5 | FLR | 1 |
| sec | 1 | 2 | CEIL | −2 | −1 | −2 | CEIL | 1 |
| tan | FLR | 1.73205081 | CEIL | −1.73205081 | FLR | 1.73205081 | CEIL | FLR |
| cot | CEIL | 0.57735027 | FLR | −0.57735027 | CEIL | 0.57735027 | FLR | CEIL |

Because the bounds are exact reciprocal duals (A2), FLR × CEIL = 1 exactly. Therefore:

* sin · csc = 1 at all eight columns, including 0 and 1/2

* cos · sec = 1 at all eight, including 1/4 and 3/4

* tan · cot = 1 at all eight, including 0, 1/4, 1/2, 3/4

Zero exceptions. Classical trigonometry must carve out four undefined points per reciprocal row. This framework does not, because the pole is not a discontinuity — it is the ceiling reached legitimately, the floor viewed from the reciprocal side. No signed zero, no signed infinity, no ±∞ branch.

These are three verification gates that exercise boundary conditions, unlike unity identities such as sin²+cos² = 1 which only exercise the interior.

## 3. Base selection

### 3.1 Criterion

The working modulus must maximize exact divisibility over the divisors the application actually uses, subject to A5. Divisor count is the headline metric; specific required divisors are hard constraints.

### 3.2 Candidates

| Modulus | Factorization | Divisors | /8 | /16 | /256 (byte) | /216 (6³) | /243 (3⁵) | /360 |
|---------|---------------|----------|----|-----|-------------|-----------|-----------|------|
| 360 | 2³·3²·5 | 24 | 45 | ✗ | ✗ | ✗ | ✗ | 1 |
| 2,160 | 2⁴·3³·5 | 40 | 270 | 135 | ✗ | 10 | ✗ | 6 |
| 11,520 | 2⁸·3²·5 | 54 | 1440 | 720 | 45 | ✗ | ✗ | 32 |
| 34,560 | 2⁸·3³·5 | 72 | 4320 | 2160 | 135 | 160 | ✗ | 96 |
| 77,760 | 2⁶·3⁵·5 | 84 | 9720 | 4860 | ✗ | 360 | 320 | 216 |
| 311,040 | 2⁸·3⁵·5 | 108 | 38880 | 19440 | 1215 | 1440 | 1280 | 864 |

Correction to an earlier working note: 77,760 = 360 × 216 factors as 2⁶·3⁵·5, giving 84 divisors — not 2⁵·3⁵·5 with 72. It divides by 64 exactly (77760/64 = 1215). An earlier draft understated both. 311,040 has 108 divisors, not 96.

### 3.3 The architectural fork

Two application constraints pull in opposite directions:

* Bit → byte → character requires 2⁸ | N. A pure binary demand.

* 36-cell / 6-contact voxel geometry requires 216 = 2³·3³ | N, and ternary depth requires 3⁵ = 243.

* 34,560 is byte-native. Bit/byte/character survives intact; voxels land at 160 steps. Ternary depth unavailable.

* 77,760 is voxel-native and trit-native. 6³ contacts land on 360 steps — one per degree, an unusually clean correspondence. Bytes break (77760/256 = 303.75).

* 311,040 is the union: bytes, 243, 216, 360, and every eighth, sixteenth, third, fifth and tenth. 108 divisors. The cost is size.

**Chaining.** Extremes are handled by 360×N chaining, with laps and higher-order structure carried as separate digits. Two independent observations:

The sexagesimal chain self-terminates at the reality floor:

```
360 → ×60 = 21,600   (arcmin)     resolution 4.63e-5
    → ×60 = 1,296,000 (arcsec)     resolution 7.72e-7
    → ×60 = 77,760,000 (arc-thirds) resolution 1.286e-8   ← finest honest level
    → ×60 = 4,665,600,000           resolution 2.14e-10   ← below reality floor
```

Exactly three refinements, landing at 1.286e−8 — just coarser than the 1e−8 floor. The bound is not imposed on the ladder; the ladder respects it natively.

Separately, 77,760 is exactly one-thousandth of 77,760,000. The voxel chain and the sexagesimal chain converge on the same numeral three decades apart, because 60³ = 216,000 and 216 = 6³ share the factor 6.

Fewer chaining levels means fewer conversion boundaries, and every boundary is a drift surface. This favors the larger single modulus over deeper chaining where the application permits.

### 3.4 Prime mitigation

| Case | Nature | Mitigation |
|------|--------|------------|
| 7, 11, 13, … | Prime, fails cleanly with known remainder | Exclusion where geometry permits; otherwise bracket between adjacent exact nodes and truncate. Output carries its own uncertainty interval rather than laundering it into a false exact value. |
| π | Irrational — categorically worse | No modulus divides it at any N. Mitigated by fiat: truncated at the 1e−8 floor to 3.14159265. |

The π case is worth stating plainly because it is the framework's hardest boundary. A prime fails cleanly — you get a known remainder and can bound it. An irrational has no exact representation at any modulus whatsoever. Where π appears as a unit artifact (radian measure), rebasing to degrees removes it entirely. Where it is geometric (unit-disk area), it cannot be removed and must be truncated. The schema should record which mechanism applies, since they have different guarantees.

Note that 1/7 turn = 51.428571…° is unrepresentable at any 5-smooth N, permanently. Heptagonal symmetry is outside the system by construction, not by oversight.

## 4. Why floating point is excluded

The three commitments — no zero, no infinity, no float — are one commitment in three forms, each implying the next:

1. A bounded domain (A2) forces a fixed modulus.

2. A fixed modulus forces integer representation.

3. Integer arithmetic gives floor-truncation natively (integer division on magnitudes already floors).

4. Floor-truncation gives the two-state error space (§2.2).

Float actively obstructs this:

* IEEE-754 defaults to round-half-to-even. Obtaining A4 semantics requires fighting the rounding mode at every operation.

* ULP is magnitude-dependent. For a double near 1e+8, ulp ≈ 1.49e−8 — already coarser than the 1e−8 floor. A uniform 8-decimal floor is not representable across the domain.

* Signed zero and ±∞ exist as distinct values, contradicting A2 and A3 directly.

Integer arithmetic at 34,560 / 77,760 / 311,040 has one ulp everywhere, exact division by every admissible divisor, and free wraparound at the modulus.

**Hardware note.** Unsigned integer overflow implements A1 directly: uint8 255+1 = 0, uint16 65535+1 = 0. Binary angular measurement (BAM) wraps with no modulo and no branch. But powers of two carry no factor of 3 or 5, so at uint16 one third of a turn is 21845.33 — error 5.1e−6, three orders above spec. At uint32, thirds land with error 7.8e−11, safely under the floor.

The practical split: exact divisor-rich base for the table, uint32 BAM for the runtime, with the table serving as the ground-truth fixture proving the runtime's wraparound never drifts.

## 5. Audit findings against the current table

Applied to a working fragment (Seq 170–187):

1. **Closure verified.** All 18 rows close under their own per-row modulus. The three apparent failures (171 arc length, 172 sector area, 173 segment area) resolve once modulus is recognized as per-row rather than global.

2. **Coverage gap at 5/6.** The column set is 0, 1/6, 1/4, 1/3, 1/2, 2/3, 3/4, 1 — a subsampled twelfth-lattice. Reflection pairs 1/4↔3/4 and 1/3↔2/3 are complete, but 1/6 has no 5/6 partner. Parity gates (cos θ = cos(N−θ), sin θ = −sin(N−θ)) cannot be run on that pair.
   This is a coverage problem, not an arithmetic one. 5/6 × 360 = 300 exactly, and cos(300°) = 0.5, sin(300°) = −0.86602540 — both values already present elsewhere in the table. No increase in N adds a column that was never sampled. Raising N increases resolution; it does not increase coverage. The two are independent axes.

3. **Grid completion is free.** All current columns are exact multiples of 1/12 turn. Completing 8 → 12 columns introduces zero new constants — 30° requires only 0.5 and 0.86602540, both already tabulated. This yields a complete DFT-12 / CORDIC clock lattice.

4. **Orphan lineage node.** Seq 187 (rotation determinant) is tagged method=computed with empty LinksTo/derived_from. Since det = cos²+sin², it should link to the sin² and cos² rows the way Seq 183 links to 181 and 182.

5. **Under-utilized half-angle rows.** Seq 161 sin(θ/2) and 162 cos(θ/2) currently feed only chord length and wave intensity. They are the quaternion vector and scalar parts, and 184 cos²(θ/2) is simultaneously the Born-rule probability, the complement of the Hann window, and — via its complement — the haversine. These are missing lineage edges, not missing computations: no new floats, no new drift surface.

### Recommended schema additions

| Field | Purpose |
|-------|---------|
| Modulus | Per-row N (§2.1). Most rows inherit 2π; those that don't must declare. |
| Closure | Verified col0 == colN under the row's own modulus. |
| PrimeMitigation | none / excluded / bracketed / truncated-by-fiat (§3.4). |
| Representation | New tier alongside derived: exact value plus fixed-point image plus error term, carried side by side and independently auditable. |

## 6. Open questions

Stated plainly so a reviewer does not have to find them.

1. The 1/3 error-reduction figure is empirical and unreproduced outside the author's run. The residual-after-bias-removal explanation in §2.2 is a hypothesis. Priority check: is the ratio stable across chunk sizes?

2. Sign-boundary coverage. Confirm the billion-iteration run exercised both halves of the circle. Rows crossing zero (SHM position, velocity, acceleration, rotation trace) are where a toward-zero/floor confusion would surface, and a non-negative test quantity would not reveal it.

3. Modulus for rows 172–173 is π — itself irrational and therefore untruncatable-without-loss. Under modular reduction this is bounded at 3.59e−9 and does not accumulate, but the choice between "modulus π (exact, unrepresentable)" and "modulus 3.14159265 (representable, drifts per lap)" should be made deliberately rather than left to the storage format.

4. Base commitment is unresolved. 34,560 (byte-native), 77,760 (voxel/trit-native), or 311,040 (union, largest). The choice is application-driven and should be recorded as a decision with its rationale, not defaulted into.

5. 7-prime handling is specified but untested. Bracketed truncation between adjacent exact nodes needs a worked example and an error bound.

## 7. Scope

This document makes a representational claim only: that a bounded-domain, integer, floor-truncated numeral system has better-behaved error than IEEE-754 for the class of cyclic quantities Tim's Tablets tabulates.

It makes no claim about physics. In particular, no argument here bears on the interpretation of dimensionality, and none should be read into it. The representational argument stands or falls on its own arithmetic and is reproducible by anyone willing to run it — which is a stronger position than one requiring interpretive agreement.

## Appendix: Reference values

```
Reality bounds       1e-8  (floor)   1e+8  (ceiling)
                     exact reciprocal duals: FLR × CEIL = 1

π                    3.14159265358979...
trunc(π, 8)          3.14159265
truncation error     -3.58979e-9   (sign-fixed under A3+A4)

Canonical columns    0, 1/6, 1/4, 1/3, 1/2, 2/3, 3/4, 1     (present)
                     + 1/12, 5/12, 5/6, 7/12, 11/12          (recommended)
base-360 integers    0, 60, 90, 120, 180, 240, 270, 360
                     + 30, 150, 300, 210, 330

√3/2                 0.86602540    √3       1.73205081
1/√3                 0.57735027    2/√3     1.15470054

Sexagesimal terminus 77,760,000    resolution 1.286e-8
Double ulp at 1e+8   ≈ 1.49e-8     (coarser than floor — float fails here)

1/7 turn             51.428571...°  unrepresentable at any 5-smooth N
```

---

Prepared as a working draft from a review session. Corrections identified during that review — per-row modulus, floor-versus-toward-zero semantics, the 5/6 coverage distinction, and the 77,760 factorization — are incorporated above and noted at their point of occurrence.
