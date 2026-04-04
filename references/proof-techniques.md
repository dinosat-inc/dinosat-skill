# Proof Techniques

Eight formal techniques for converting intractable symbolic properties into Z3-provable assertions. Each technique specifies: preconditions for applicability, algebraic transformation, soundness argument, and implementation pattern.

---

## Technique 1: Cross-Multiplication

**Precondition:** Property compares ratios or involves division by a symbolic positive denominator.

**Transform:**
```
a / b <= c / d   →   a * d <= b * c     (when b > 0, d > 0)
a / b <= c       →   a <= b * c         (when b > 0)
a / b >= c / d   →   a * d >= b * c     (when b > 0, d > 0)
```

**Soundness:** For positive integers with floor division, `a <= b * c` implies `floor(a/b) <= c` (conservative direction). Note: the reverse does NOT hold for floor division — `floor(a/b) <= c` does not imply `a <= b*c`. The cross-multiplied form is a sufficient condition, not an equivalence. This means proofs using cross-multiplication are sound but may be stricter than necessary.

**Pattern:**
```solidity
// BEFORE (intractable — symbolic division):
// assert(valueAfter / newTS >= valueBefore / totalSupply);

// AFTER (tractable — multiplication only):
assert(valueAfter * totalSupply >= valueBefore * newTotalSupply);
```

**Overflow guard:** Verify cross-products fit `uint256` for the constrained input ranges.

---

## Technique 2: Boundary Parameterization

**Precondition:** Multi-variable property where full symbolic exploration times out. A monotonicity relationship exists between one variable and the property's truth.

**Transform:**
1. Identify the variable whose increase makes the property hardest to satisfy (worst case)
2. Fix that variable at its boundary value (constant)
3. Leave remaining variables symbolic
4. Prove the property at the boundary

**Soundness:** If `f(x, y)` is non-decreasing in `x` and `f(x_max, y) <= bound` for all `y`, then `f(x, y) <= bound` for all `x <= x_max` and all `y`. The monotonicity argument MUST be stated explicitly in the test's NatSpec.

**Pattern:**
```solidity
/// @notice Proves ratio <= T at boundary valueA = 99*resB.
///         Monotonicity: ratio = valueA/(valueA+resB) is increasing in valueA,
///         so the boundary (maximum valueA) is the hardest case.
function test_check_lemma_ratio_bounded(uint256 resB) public pure {
    vm.assume(resB >= 1e6 && resB <= type(uint96).max);
    uint256 valueA = 99 * resB;  // boundary: worst case
    assert(valueA * P <= THRESHOLD * (valueA + resB));
}
```

---

## Technique 3: Concrete Enumeration

**Precondition:** Property parameterized over a finite domain of N values (N <= 10). Symbolic representation of the domain variable causes Z3 timeout (typically via `10 ** x` or symbolic divisor).

**Transform:** Replace one parametric test with N concrete tests. Each uses the domain value as a literal constant.

**Soundness:** The union of all N concrete proofs covers the entire domain. Each proof is exhaustive over the remaining symbolic variables. Document the domain exhaustively in a composition comment:
```
// Composition: {2, 3, 4, 5} covers all valid pool counts.
// test_check_remainder_2pools: PASS (formally proven)
// test_check_remainder_3pools: FUZZ (Z3 intractable on div-by-3)
// test_check_remainder_4pools: PASS (formally proven)
// test_check_remainder_5pools: FUZZ (Z3 intractable on div-by-5)
// Coverage: 4/4 domain values verified (2 proven + 2 fuzz).
```

**Pattern:**
```solidity
// One test per pool count (domain: {2, 3, 4, 5})
function test_check_remainder_2pools(uint256 total) public pure {
    vm.assume(total >= 5 && total <= 1e30);
    uint256 share = total / 2;  // constant divisor — Z3 tractable
    vm.assume(share > 0);
    uint256 last = total - share;
    assert(last >= share && last <= share + 1);
}
```

---

## Technique 4: Inline mulDiv

**Precondition:** Property uses `Math.mulDiv(a, b, c)` from OpenZeppelin. The product `a * b` fits in `uint256` for the constrained input ranges.

**Transform:** Replace `Math.mulDiv(a, b, c)` with `(a * b) / c`. Add `vm.assume()` constraining inputs so `a * b <= type(uint256).max`.

**Soundness:** `Math.mulDiv(a, b, c)` computes `floor(a * b / c)` using 512-bit intermediate to avoid overflow. When `a * b` fits `uint256`, the 512-bit path is unnecessary and `(a * b) / c` produces the identical result. The `vm.assume` constraints define the domain where this equivalence holds.

**Pattern:**
```solidity
/// @notice Uses inline arithmetic. Product fits uint256:
///         currentRate (max 1e24) * multiplier (max ~1e18) = max ~1e42.
function test_check_rate_nonzero(uint256 currentRate, uint256 multiplier) public pure {
    vm.assume(currentRate >= 1e12 && currentRate <= 1e24);
    vm.assume(multiplier >= 998e15 && multiplier <= 1002e15);
    uint256 newRate = (currentRate * multiplier) / P;
    assert(newRate > 0);
}
```

**Visibility:** Change from `public view` (Math.mulDiv uses assembly) to `public pure` (inline is pure).

---

## Technique 5: Lemma Decomposition

**Precondition:** Property depends on multi-branch logic (3+ branches) or multi-step computation where the combined path count exceeds Z3's capacity.

**Transform:**
1. Identify independent sub-properties (lemmas)
2. Prove each lemma in isolation
3. Write per-branch theorems that compose lemmas
4. The union of all branch theorems covers all inputs

**Soundness:** If branches B1, B2, B3 partition the input space (enforced by `vm.assume` branch conditions), and the property is proven for each branch independently, then the property holds universally. Each lemma's precondition must be satisfiable (non-vacuous).

**Compositional soundness documentation (mandatory):** Every decomposed proof MUST include a composition comment block documenting:
1. The branch conditions and that they partition the input space exhaustively
2. The status of each branch theorem (PASS or FUZZ)
3. Which lemmas each theorem depends on

```solidity
// ════════════════════════════════════════════════
//  COMPOSITION: fee < amountIn
//  Branch A: deviation <= UNBALANCING_THRESHOLD → effectiveFee = baseFee
//  Branch B: deviation > threshold, deviation < currentDeviation → effectiveFee = baseFee
//  Branch C: deviation >= currentDeviation → effectiveFee = k * dev² / P
//  Partition: A ∪ B ∪ C = all (deviation, currentDeviation) pairs. Exhaustive.
//  Depends on: Lemma 1 (baseFee < P), Lemma 3 (quadratic < P)
//  Status: Branch A [PASS], Branch B [PASS], Branch C [PASS]. QED.
// ════════════════════════════════════════════════
```

**Structure:**
```
Lemma 1: baseFee < P                          (constant bound)
Lemma 2: deviation <= P/2                      (range bound)
Lemma 3: quadratic effectiveFee < P            (composed from L1, L2)
Lemma 4: fee < amountIn when effectiveFee < P  (floor division)

Theorem A: Branch A → fee < amountIn           (uses L1 + L4)
Theorem B: Branch B → fee < amountIn           (uses L1 + L4)
Theorem C: Branch C → fee < amountIn           (uses L3 + L4)

Composition: A ∪ B ∪ C = all inputs. QED.
```

**File naming:** `<Feature>Lemmas.t.sol`

---

## Technique 6: Monotonicity Arguments

**Precondition:** Property `P(x)` holds at a boundary value `x_0`, and the function under test is monotone in `x`.

**Transform:** Prove `P(x_0)` formally. State the monotonicity argument in NatSpec. The composition `P(x_0) ∧ monotone(f, x)` implies `P(x)` for all `x` in the range.

**Soundness:** If `f` is non-decreasing and `f(x_max) <= bound`, then `f(x) <= bound` for all `x <= x_max`. The monotonicity of common operations:
- `floor(a * x / c)` is non-decreasing in `x` when `a, c > 0`
- `x / (x + c)` is increasing in `x` when `c > 0`
- `a + x` is increasing in `x`

**Pattern:**
```solidity
/// @notice valueRatio is monotonic in resA.
///         Proof: valueA = floor(resA * rate / P) is non-decreasing in resA.
///         ratio = valueA / (valueA + resB) is increasing in valueA (cross-multiply).
function test_check_ratio_monotonic_in_A(
    uint256 resA1, uint256 resA2, uint256 resB, uint256 baseRate
) public pure {
    vm.assume(resB >= 1e18 && resB <= 1e24);
    vm.assume(baseRate >= 1e15 && baseRate <= 1e21);
    vm.assume(resA1 >= 1e18 && resA1 <= 1e24 - 1);
    vm.assume(resA2 >= resA1 + 1 && resA2 <= 1e24);

    // Step 1: valueA is non-decreasing in resA (product monotonicity)
    uint256 valueA1 = (resA1 * baseRate) / P;
    uint256 valueA2 = (resA2 * baseRate) / P;

    // Step 2: ratio = valueA / (valueA + resB) is increasing in valueA.
    // Cross-multiply: ratio2 >= ratio1 iff valueA2 * resB >= valueA1 * resB.
    // Since valueA2 >= valueA1 (Step 1) and resB > 0, this holds.
    assert(valueA2 * (valueA1 + resB) >= valueA1 * (valueA2 + resB));
}
```

---

## Technique 7: Floor Division Identity

**Precondition:** Property depends on integer division rounding behavior (share price, scaling, distribution).

**Core identities** (Z3 bitvector tautologies):
```
floor(a / b) * b <= a                    [floor rounds down]
a - floor(a / b) * b < b                 [remainder < divisor]
floor(a / b) <= floor(c / b)  when a <= c  [monotone in numerator]
```

**Applications:**
- Share price non-decreasing: `outA * totalSupply <= burnAmount * resA` (from `outA = floor(burn * resA / ts)`)
- No value extraction: `(outA + outB) * ts <= burn * (resA + resB)` (sum of two floor identities)
- Scale roundtrip: `(rawAmount * scale) / scale == rawAmount` (exact when no remainder)
- Distribution remainder: `total - floor(total / N) * (N-1) <= floor(total / N) + N - 1`

**Pattern:**
```solidity
function test_check_noValueExtraction(
    uint256 resA, uint256 totalSupply, uint256 burnAmount
) public pure {
    uint256 outA = (burnAmount * resA) / totalSupply;
    // Floor identity: floor(a*b/c) * c <= a*b
    assert(outA * totalSupply <= burnAmount * resA);
}
```

---

## Technique 8: Abstract Specification

**Precondition:** Contract under test uses Z3-hostile operations (Math.mulDiv, Math.sqrt, assembly) in its core logic. Multiple properties depend on the same computation.

**Transform:** Build a private helper function in the test contract that mirrors the contract's logic using Z3-friendly inline arithmetic. All tests for that contract call the helper instead of the real library.

**Soundness:** The abstract spec computes `floor(a * b / c)` identically to `Math.mulDiv(a, b, c)` when `a * b` fits in `uint256`. The `vm.assume()` constraints define the domain where the equivalence holds. The helper and the real function produce the same output for all inputs in this domain.

**When to use:** When a contract has 3+ properties that all depend on the same Math.mulDiv-based computation (e.g., `calculateValueRatio` used by fee engine, swap validation, and incentive caps). Writing one abstract helper avoids duplicating the inline logic across many tests.

**Pattern:**
```solidity
contract TokenPoolAbstractSymbolicTest is Test {
    uint256 constant P = 1e18;

    /// @dev Abstract spec of PoolMath.calculateValueRatio.
    ///      Uses inline (a*b)/c instead of Math.mulDiv.
    ///      Valid for: resA <= 1e24, baseRate <= 1e24 (product <= 1e48).
    function _valueRatio(uint256 resA, uint256 resB, uint256 baseRate) private pure returns (uint256) {
        uint256 valueA = (resA * baseRate) / P;
        uint256 totalValue = valueA + resB;
        if (totalValue == 0) return 0;
        return (valueA * P) / totalValue;
    }

    /// @dev Abstract spec of FeeEngine.calculateSwapFee (branch C only).
    function _quadraticFee(uint256 baseFee, uint256 deviation) private pure returns (uint256) {
        uint256 k = baseFee * FEE_K_MULTIPLIER;
        uint256 devSquared = (deviation * deviation) / P;
        return (k * devSquared) / P;
    }

    /// @dev Abstract spec of PoolMath.calculateLiquidityMint.
    function _liquidityMint(uint256 addA, uint256 addB, uint256 resA, uint256 resB, uint256 ts)
        private pure returns (uint256)
    {
        uint256 liqA = (addA * ts) / resA;
        uint256 liqB = (addB * ts) / resB;
        return liqA < liqB ? liqA : liqB;
    }

    // All tests below use _valueRatio, _quadraticFee, _liquidityMint
    // instead of calling PoolMath/FeeEngine directly.
    // This makes them `public pure` and Z3-tractable.
}
```

**File naming:** `<Contract>AbstractSymbolic.t.sol` — the "Abstract" suffix signals that inline arithmetic replaces library calls.

**Relationship to Technique 4 (Inline mulDiv):** Technique 4 is a point replacement of a single `Math.mulDiv` call. Technique 8 is a systematic abstraction of an entire contract's computation into a Z3-friendly specification. Use Technique 8 when 3+ tests share the same computation; use Technique 4 for isolated one-off replacements.

**Overflow documentation (mandatory):** Every abstract helper MUST document:
1. Which real function it mirrors
2. The maximum input ranges (`resA <= 1e24, baseRate <= 1e24`)
3. The maximum intermediate product (`product <= 1e48 < uint256.max`)
4. Why the inline version is equivalent within those bounds
