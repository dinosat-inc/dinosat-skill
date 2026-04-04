# Decision Flowchart

Single decision path for classifying ANY property. Start at the top, follow arrows. First matching condition determines the classification.

---

## Master Classification Flow

```
INPUT: A security property P of function F

Step 1: Does F contain Math.sqrt?
├── YES → Is P about the sqrt output specifically?
│         ├── YES → Classification: FUZZ (Math.sqrt intractable for Z3)
│         └── NO  → Continue to Step 2 (property may not involve sqrt path)
└── NO  → Continue to Step 2

Step 2: Does F contain Math.mulDiv?
├── YES → Can all mulDiv products fit uint256 with bounded inputs?
│         ├── YES → Classification: INLINE
│         │         Use: (a*b)/c with vm.assume constraining a*b < uint256.max
│         │         Visibility: public pure
│         └── NO  → Classification: FUZZ (512-bit intractable)
└── NO  → Continue to Step 3

Step 3: Does F have 3+ branches (if/else if/else) with arithmetic in each?
├── YES → Classification: DECOMPOSE
│         Create: <Feature>Lemmas.t.sol with one theorem per branch
│         Each branch theorem uses INLINE if it contains mulDiv
└── NO  → Continue to Step 4

Step 4: Does the assertion involve division by a symbolic variable?
├── YES → Can the assertion be rewritten as a multiplication? (a/b <= c → a <= b*c)
│         ├── YES → Classification: CROSSMUL
│         └── NO  → Is the divisor from a small finite domain (<= 10 values)?
│                   ├── YES → Classification: ENUMERATE
│                   │         One test per concrete value
│                   └── NO  → Classification: BOUNDARY
│                             Fix divisor at worst-case, prove monotonicity
└── NO  → Continue to Step 5

Step 5: Does the assertion involve a product of two symbolic variables?
├── YES → Can input bounds constrain the product to a tractable range?
│         ├── YES → Classification: DIRECT (with tight vm.assume bounds)
│         └── NO  → Can cross-multiplication eliminate the product?
│                   ├── YES → Classification: CROSSMUL
│                   └── NO  → Classification: FUZZ
└── NO  → Continue to Step 6

Step 6: Does P involve 10**(18-decimals) or similar symbolic exponentiation?
├── YES → Classification: ENUMERATE
│         One test per concrete exponent value
└── NO  → Continue to Step 7

Step 7: Does F contain a loop?
├── YES → Is the loop bound known and small (<= 10)?
│         ├── YES → Classification: DIRECT (with --loop N)
│         └── NO  → Classification: FUZZ
└── NO  → Classification: DIRECT
```

---

## Technique Selection Flow

After classification, select the implementation technique:

```
Classification → Technique → Template

DIRECT      → Standard symbolic test         → T2 (bound), T3 (access), T6 (floor), T12 (gating)
INLINE      → Abstract spec helper           → T10 (helper) + T2/T8 (test using helper)
DECOMPOSE   → Per-branch lemma file          → T4 (per-branch theorem) + T5 (ratio) + composition
ENUMERATE   → Concrete value tests           → T9 (one per value)
BOUNDARY    → Worst-case parameterization    → T15 (boundary) + T11 (monotonicity)
CROSSMUL    → Multiplication rewrite         → T5 (ratio), T7 (withdrawal), T8 (deposit)
FUZZ        → Standard test (fuzz-verified)  → T2 (bound) with expectation of [TIMEOUT]
```

---

## Timeout Escalation Flow

When a test classified as provable times out:

```
[TIMEOUT] on test T classified as <CLASS>

Attempt 1: Check for accidental Math.mulDiv call
├── Found → Replace with inline (a*b)/c, rerun
│           ├── [PASS] → Done
│           └── [TIMEOUT] → Attempt 2
└── Not found → Attempt 2

Attempt 2: Tighten input bounds
├── Can bounds be reduced? (e.g., 1e27 → 1e24)
│   ├── YES → Tighten, rerun
│   │         ├── [PASS] → Done (document tighter bounds in report)
│   │         └── [TIMEOUT] → Attempt 3
│   └── NO → Attempt 3
└──

Attempt 3: Apply alternative technique
├── Can the assertion use cross-multiplication? → Rewrite, rerun
├── Can one variable be fixed at boundary? → Boundary param, rerun
├── Can branches be isolated? → Decompose, rerun
└── None applicable → Reclassify as FUZZ

Maximum: 3 attempts. After 3 failures → FUZZ unconditionally.
```

---

## Vacuous Test Detection

After Halmos runs, check for these warning signs that a test may be vacuous (trivially true):

```
Warning: "all paths have been reverted"
├── Cause: vm.assume constraints are contradictory (no valid input exists)
├── Fix: Review assumptions — are they satisfiable simultaneously?
│         Example: vm.assume(x >= 100 && x <= 50) → contradiction
└── Action: Fix constraints and rerun. A vacuous proof proves nothing.

Warning: "paths: 0" or "paths: 1" on a multi-parameter test
├── Cause: Constraints may be so tight that only one input is tested
├── Fix: Verify the vm.assume ranges admit diverse inputs
└── Action: If paths == 1 and test has 3+ parameters, investigate.

Warning: assertion is a tautology
├── Cause: assert(x >= 0) on uint256 (always true)
├── Cause: assert(x == x) or assert(a + b == a + b)
├── Cause: assert(result <= MAX) where result was just assigned MAX
├── Fix: Ensure the assertion tests a non-trivial property
└── Action: Review and strengthen the assertion.

Warning: assertion only re-states an assumption
├── Cause: vm.assume(x < P); ... assert(x < P);
├── Fix: The proof must do computation between assume and assert
└── Action: Add the actual computation and test the meaningful property.
```
