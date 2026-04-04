# Z3 Tractability Classification

Decision reference for classifying symbolic verification properties. Every property MUST be classified before test generation.

---

## Classification: DIRECT

Z3 proves exhaustively within the solver timeout. No transformation needed.

**Patterns:**
- Constant comparisons: `assert(A < B)` where both are literals
- Single-variable bounded: `vm.assume(x >= a && x <= b); assert(f(x) <= c)` where f is linear
- Floor division identity: `assert(q * b <= a)` where `q = a / b`
- Remainder bound: `assert(a - (a / b) * b < b)`
- Division by constant: `assert(x / CONSTANT <= BOUND)`
- Address/boolean logic: `assert(caller != authorized)`
- Quadratic with bounded base: `assert(x * x <= MAX)` where `vm.assume(x <= sqrt(MAX))`
- Linear arithmetic: `assert(a + b == c)`, `assert(a >= b)` with bounded inputs
- Conditional branching (2-3 branches): simple `if/else` with tractable body per branch
- Bounded loop with `--loop N`: array sorting, fixed iteration

---

## Classification: INLINE

Property uses `Math.mulDiv(a, b, c)` (OpenZeppelin 512-bit assembly). Z3 cannot parse the assembly. Replace with `(a * b) / c` and constrain inputs to prevent overflow.

**Precondition:** Verify `a * b <= type(uint256).max` for the constrained ranges.

**Safe product ranges:**

| Variable A | Variable B | Max Product | Fits uint256? |
|---|---|---|---|
| Reserve (1e24) | PRECISION (1e18) | 1e42 | YES |
| Amount (1e27) | Fee rate (1e18) | 1e45 | YES |
| Rate (1e24) | Multiplier (1.002e18) | ~1e42 | YES |
| Reserve (1e27) | Reserve (1e27) | 1e54 | YES |
| Reserve (1e30) | Rate (1e24) | 1e54 | YES |
| Amount (1e36) | Rate (1e36) | 1e72 | YES (fits, but tighten if possible) |
| Amount (1e39) | Rate (1e39) | 1e78 | NO — exceeds uint256.max (~1.16e77) |

**Visibility change:** `Math.mulDiv` requires `public view` (assembly). Inline `(a*b)/c` allows `public pure`.

---

## Classification: DECOMPOSE

Property involves multi-branch logic (3+ branches, each with arithmetic). The combined path explosion exceeds Z3's capacity. Split into one test per branch.

**Structure:**
1. Identify independent branches (e.g., fee engine: flat / improving / quadratic)
2. Write `test_check_lemma_<name>` for each building block
3. Write `test_check_theorem_<name>_branch<X>` for each branch
4. Branch-specific `vm.assume()` selects the branch
5. Composition: the union of all branch theorems covers all inputs

**Naming:** `<Feature>Lemmas.t.sol` for the decomposed file.

---

## Classification: ENUMERATE

Property parameterized over a small finite domain. Symbolic representation of the domain variable (e.g., `10 ** (18 - decimals)` with symbolic `decimals`) is intractable. Replace with one test per concrete value.

**Common domains:**
- Token decimals: {6, 8, 12, 18} — suffix `_6dec`, `_8dec`, `_12dec`, `_18dec`
- Pool counts: {2, 3, 4, 5} — suffix `_2pools`, `_3pools`, etc.
- Enum values: {0, 1, 2} — suffix `_typeA`, `_typeB`, etc.

**Note:** Z3 handles even-constant division (2, 4, 8) better than odd (3, 5, 7). Odd-constant tests may still timeout — this is expected, mark as FUZZ.

---

## Classification: BOUNDARY

Multi-variable property with nonlinear `vm.assume` constraints (e.g., `vm.assume(a * b <= c * d)` with all symbolic). Z3 cannot solve nonlinear bitvector quantification.

**Transform:** Fix one variable at its worst-case boundary value. Leave remaining variables symbolic. Prove at the boundary. Invoke monotonicity to cover all values.

**Requirement:** A separate monotonicity proof or argument MUST justify that the boundary is the worst case.

---

## Classification: CROSSMUL

Property compares ratios or involves division by symbolic denominator. Rewrite as multiplication inequality.

**Transforms:**
- `a / b <= c` → `a <= b * c`
- `a / b >= c / d` → `a * d >= b * c`
- `floor(a/b) * b <= a` → direct assertion (Z3 tautology)

**Precondition:** Both denominators must be provably positive within the `vm.assume` constraints.

---

## Classification: FUZZ

Property is intractable for Z3 under any transformation. Verify via Forge fuzz testing (1,000,000 random inputs).

**Intractable patterns (no known workaround):**
- Symbolic x symbolic product in assertion with full 256-bit range
- Nested `Math.mulDiv` (two layers of 512-bit intermediate)
- 4+ symbolic variables through complex nonlinear arithmetic
- Transcendental functions (`Math.sqrt` with symbolic input)
- Multi-branch fee engine with 6+ symbolic divisions (even after decomposition)

**FUZZ is not a failure.** It provides empirical verification with 1,000,000 runs. Combined with decomposed lemma proofs of the same property's building blocks, FUZZ tests achieve high confidence.

---

## Solver Configuration

| Parameter | Value | Rationale |
|---|---|---|
| `--solver-timeout-assertion` | `30000` (30s) | Default. Covers most tractable properties. |
| `--solver-timeout-assertion` | `60000` (60s) | Try once before declaring intractable. |
| `--loop` | `5` | For tests with bounded loops (sorting, iteration). |
| `optimizer` | `false` | NON-NEGOTIABLE. Optimizer transforms confuse Z3. |
| `ast` | `true` | REQUIRED. Halmos cannot parse artifacts without AST. |
| `evm_version` | `shanghai` | Compatibility with Halmos 0.1.13. Cancun opcodes unsupported. |
