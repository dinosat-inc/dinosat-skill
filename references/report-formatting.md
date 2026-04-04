# Report Formatting

Exact output format for `DINOSAT_REPORT.md`. Follow this template precisely. Do not reorder sections or omit required fields. The markdown report is the source of truth — the PDF is generated from it via `generate_pdf.py`.

---

## Output Files

```
DINOSAT_REPORT.md    # Markdown (always generated)
DINOSAT_REPORT.pdf   # PDF (generated if weasyprint available)
dinosat-log.txt                   # Raw Halmos output
```

Place at project root.

---

## Template

```markdown
# <Protocol Name> — Formal Verification Report

**Protocol:** <Protocol Name>
**Date:** <YYYY-MM-DD>
**Tool:** Halmos v<version> (Z3 SMT Solver)
**Solidity:** <version> (<EVM target>)
**Protocol type:** <AMM | Lending | Vault | Staking | Governance | Token | Bridge | Other>
**Scope:** <N> contracts, <L> nSLOC
**Methodology:** Symbolic execution with exhaustive input space exploration

---

## Verification Status Legend

| Status | Meaning |
|---|---|
| PROVEN | Z3 exhausted all paths within the `vm.assume` input domain -- no counterexample exists. Mathematical proof. |
| PROVEN (composed) | End-to-end test is FUZZ, but all building-block lemmas are individually PROVEN. Compositional proof. |
| FUZZ | Property verified empirically via Forge fuzz testing (1,000,000 runs, 0 failures). Not a mathematical proof. |
| COUNTEREXAMPLE | Z3 found concrete input values that violate the assertion. Potential vulnerability -- see Detailed Results. |
| TIMEOUT | Z3 could not resolve within the solver timeout. Reclassified to FUZZ for empirical verification. |

---

## Threat Model

### Assets at Risk

<List the protocol's assets that formal verification aims to protect. Derive from source analysis.>

- **User deposits** -- <token type> deposited into <contract(s)>
- **Outstanding debt / liabilities** -- <if lending: borrowed amounts + accrued interest>
- **Fee / reward accounting** -- <fee splits, incentive distributions, protocol revenue>
- **Share / position value** -- <LP tokens, vault shares, staked positions used for solvency>
- **Protocol solvency state** -- <bad debt, write-offs, reserve ratios>

### Actors

- **Depositor / LP** -- provides liquidity, earns yield; harmed by share dilution or value extraction
- **Borrower** -- takes loans against collateral; may become liquidatable
- **Trader / Swapper** -- interacts with AMM; may attempt sandwich or arbitrage
- **Admin / Governance** -- sets parameters; assumed to act in good faith (out of scope)
- **External contracts** -- oracles, callbacks, token contracts; assumed to behave per spec

<Include only actors relevant to the detected protocol type. Omit inapplicable roles.>

### Trust Assumptions

- Price oracles return correct, manipulation-resistant values (oracle compromise is out of scope)
- ERC-20 tokens comply with the standard (no fee-on-transfer, rebasing, or callback behavior)
- Protocol parameters are set to economically sound values at deployment
- Admin/governance acts in good faith and does not maliciously reconfigure the protocol
- <Add any project-specific assumptions derived from constructor validations or documentation>

### Attack Vectors Considered

<For each detected protocol type, list the primary attack vectors that the formal verification properties are designed to catch. Reference the property IDs.>

| Attack Vector | Properties | Mitigated By |
|---|---|---|
| <e.g., Fee exceeding input drains user> | FB-1, FB-2 | DECOMPOSE per-branch proofs |
| <e.g., Share dilution on deposit> | LI-1 | CROSSMUL + floor identity |
| <e.g., Overflow in swap math> | AS-1 | INLINE with bounded inputs |
| <e.g., Unauthorized parameter change> | AC-1 | DIRECT access control proof |
| ... | ... | ... |

---

## Results Summary

| Contract | File | Tests | Proven | Fuzz | Status |
|---|---|---|---|---|---|
| <Name>SymbolicTest | test/dinosat/<file> | <N> | <P> | <F> | <ALL PROVEN / ALL VERIFIED / COUNTEREXAMPLE FOUND> |
| <Name>LemmasTest | test/dinosat/<file> | <N> | <P> | <F> | <status> |
| ... | ... | ... | ... | ... | ... |
| **TOTAL** | | **<N>** | **<P>** | **<F>** | |

**Verdict:** <P> formal proofs, <F> fuzz-verified, <C> counterexamples found.

> If C > 0: **VULNERABILITIES DISCOVERED. See Counterexamples Found below.**

---

## Detailed Results

### Counterexamples Found (if any)

> This section appears ONLY if Halmos found a real counterexample. It MUST be the first detailed section.

**[COUNTEREXAMPLE] <test_name>**
- **Severity:** <CRITICAL / HIGH / MEDIUM / LOW>
- **Affected function:** `<Contract>.<function>()`
- **Property violated:** <property ID> — <description>
- **Concrete values:**
  - `<param1>` = `<value1>`
  - `<param2>` = `<value2>`
- **Exploit trace:** <step-by-step description of what happens with these values>
- **Impact:** <what an attacker could achieve>

### <ContractName>SymbolicTest

File: `test/dinosat/<ContractName>Symbolic.t.sol`
<P> formally proven, <F> fuzz-verified

**Proven:**
- `[PASS]` `test_check_<name>(<params>)` (paths: <N>, time: <X>s) — <property ID>: <one-line description>
- ...

**Fuzz-verified:**
- `[FUZZ]` `test_check_<name>(<params>)` (1,000,000 runs, 0 failures) — <property ID>: <description>
- ...

**Composed proofs** (property proven via lemma decomposition, end-to-end test is FUZZ):
- `test_check_<name>` — FUZZ end-to-end, but **PROVEN via composition** of:
  - `test_check_lemma_<a>` [PASS] + `test_check_lemma_<b>` [PASS] + `test_check_theorem_<c>` [PASS]
  - Composition: <branch partition argument>

### <ContractName>LemmasTest
...

---

## Properties Verified

### P0 — Critical
| ID | Property | Function(s) | Status | Technique |
|---|---|---|---|---|
| AC-1 | Unauthorized caller reverts | `setFee()` | PROVEN | DIRECT |
| AS-1 | No overflow in fee calc | `calculateSwapFee()` | PROVEN | INLINE |
| ... | ... | ... | ... | ... |

### P1 — High
| ID | Property | Function(s) | Status | Technique |
|---|---|---|---|---|
| FB-1 | fee < amountIn | `calculateSwapFee()` | PROVEN (composed) | DECOMPOSE |
| ... | ... | ... | ... | ... |

### P2 — Medium
| ID | Property | Function(s) | Status | Technique |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |

### Cross-Contract
| ID | Property | Contracts | Status | Technique |
|---|---|---|---|---|
| CC-1 | Total staked consistency | StakingManager, Pool | FUZZ | — |
| ... | ... | ... | ... | ... |

---

## Coverage Gap Analysis

| Function | Tests | Gap | Reason |
|---|---|---|---|
| `emergencyRescue()` | 0 | AC-1 missing | Out of scope (admin-only) |
| `_internalHelper()` | 0 | — | Internal, not directly testable |
| ... | ... | ... | ... |

> All P0 gaps MUST be zero. P1/P2 gaps documented with justification.

---

## Methodology

### Symbolic Execution
Halmos translates EVM bytecode into SMT (Satisfiability Modulo Theories) constraints.
For each `assert()` statement, Z3 searches for any input assignment that violates the
assertion. If no such assignment exists, the property is mathematically proven for ALL
inputs satisfying the `vm.assume()` preconditions.

### Proof Techniques Applied

| Technique | Count | Applied to |
|---|---|---|
| Cross-multiplication | <N> | <property IDs> |
| Boundary parameterization | <N> | <property IDs> |
| Concrete enumeration | <N> | <property IDs> |
| Inline mulDiv | <N> | <property IDs> |
| Lemma decomposition | <N> | <property IDs> |
| Monotonicity | <N> | <property IDs> |
| Floor division identity | <N> | <property IDs> |
| Abstract specification | <N> | <property IDs> |

### Z3-Hostile Patterns Detected

| Pattern | Location | Strategy | Result |
|---|---|---|---|
| Math.mulDiv | `PoolMath.sol:L42` | INLINE | PROVEN |
| 3-branch fee engine | `FeeEngine.sol:L20` | DECOMPOSE | PROVEN (3 branches) |
| Math.sqrt | `PoolMath.sol:L108` | FUZZ | FUZZ (intractable) |
| ... | ... | ... | ... |

### Fuzz Testing (Complementary)
Properties are fuzz-verified via two Forge passes:
- **Pass 1 — Fuzz wrappers** (`test_fuzz_`): 1,000,000 runs per test. Uses `bound()` for near-zero rejection and real input coverage.
- **Pass 2 — Halmos tests** (`test_check_`): 100,000 runs per test. High rejection expected due to tight `vm.assume` bounds — primary coverage comes from Pass 1.
- Failures: 0

### Input Domain Bounds

| Variable | Range | Justification |
|---|---|---|
| Reserves | [1e18, 1e24] | <source: constructor validation / max supply / default> |
| Amounts | [1, 1e27] | <source: MIN_SWAP_AMOUNT to max supply> |
| Rates | [1e12, 1e24] | <source: baseRate validation range> |
| Fees | [BASE_FEE_MIN, BASE_FEE_MAX] | <source: Constants.sol> |

---

## Soundness Caveats

### Proof Domain Boundaries
Formal proofs are valid ONLY within the `vm.assume()` input domains documented in
the Input Domain Bounds section. Values outside these ranges are not covered by any proof.

### Abstract Specification Equivalence
Tests using inline `(a*b)/c` instead of `Math.mulDiv(a,b,c)` are equivalent ONLY when
`a*b <= type(uint256).max`. The `vm.assume()` constraints ensure this condition. Outside
these bounds, the abstract specification may diverge from the real implementation.

### Compositional Soundness
Decomposed proofs establish per-branch correctness. The composition requires that branch
conditions partition the input space exhaustively. Each decomposed property documents its
branch partition and composition status in Detailed Results.

### Halmos Verification Scope
Halmos verifies single-transaction properties. It does NOT verify:
- **Reentrancy:** callback-based attacks across multiple calls
- **Gas limits:** all paths explored regardless of gas cost
- **Storage layout:** proxy patterns, upgradeable contract compatibility
- **Cross-contract callbacks:** ERC-777 hooks, flash loan callbacks
- **External call failures:** revert propagation from called contracts
- **ERC-20 transfer quirks:** fee-on-transfer, rebasing, permit

### Fuzz Limitations
FUZZ-classified properties provide empirical evidence from 1,000,000 random inputs.
A counterexample may exist outside the sampled space. Where a FUZZ property is also
covered by decomposed lemma proofs, this is documented as "PROVEN via composition"
in Detailed Results.

---

## Conclusion

**Total properties:** <N>
**Formally proven (Halmos/Z3):** <P> — mathematical proof, no counterexample possible
**Proven via composition:** <C> — end-to-end FUZZ, building blocks all PROVEN
**Fuzz-verified (Forge):** <F> — 1,000,000 runs, 0 failures
**Counterexamples found:** <X>

<If X > 0: summary of discovered vulnerabilities>
<If X == 0: "No vulnerabilities were discovered during formal verification.">

---

## Disclaimer

This report was generated by DinoSAT, an automated formal verification tool. It is
provided "as is" without warranty of any kind, express or implied. This report does
not constitute a security audit, and does not guarantee the absence of vulnerabilities,
bugs, or exploits in the verified contracts.

Formal proofs are valid only within the documented input domains (vm.assume bounds).
Fuzz-verified properties are empirical observations, not mathematical proofs. Properties
outside the scope of this report are not covered. These include: reentrancy (Halmos
models single-call execution, not callback chains), gas optimization (all paths explored
regardless of gas cost), business logic correctness (requires domain-specific invariants
beyond arithmetic), economic incentive alignment (game-theoretic analysis is outside SMT
solving), and cross-contract interactions (Halmos verifies contracts in isolation —
multi-contract state dependencies require stateful integration testing).

This tool is free to use. Users assume all risk associated with reliance on this report.
The authors accept no liability for any loss or damage arising from the use of this
software or the information contained in this report.

https://dinosat.com
```

---

## Formatting Rules

1. **Counterexamples first.** If any `[FAIL]` with real counterexample, it MUST appear before all other detailed results.
2. **No fabrication.** Every `[PASS]` must correspond to an actual Halmos run. Every `[FUZZ]` must correspond to an actual Forge run.
3. **Composed proofs documented.** When a property is FUZZ end-to-end but PROVEN via lemma composition, list it under "Composed proofs" with the full chain.
4. **Bounds justified.** Input Domain Bounds must cite the source for every input range (protocol constant, constructor check, or "default — no protocol-specific bound").
5. **Caveats mandatory.** Soundness Caveats section must appear in every report, even if no issues were found.
6. **Sort by priority.** Within Properties Verified, P0 properties first, P1 second, P2 third.
7. **Threat Model mandatory.** All four subsections (Assets at Risk, Actors, Trust Assumptions, Attack Vectors Considered) must appear. Omit inapplicable actor roles but keep the subsection headers.
8. **Status Legend mandatory.** Must appear before Results Summary so readers understand status labels.
9. **Protocol name mandatory.** Protocol field must appear in report metadata.
