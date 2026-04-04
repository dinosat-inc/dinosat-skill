# DinoSAT

Formal verification of Solidity smart contracts using Halmos and Z3.

https://dinosat.com

## What it does

Reads your Solidity contracts, builds a threat model, identifies security-critical properties, generates Halmos symbolic tests with fuzz wrappers, runs them, and writes a branded verification report with PDF.

## Prerequisites

- [Foundry](https://book.getfoundry.sh/) (`forge`)
- [Halmos](https://github.com/a16z/halmos) v0.1.13+ (`pip install halmos`)
- `pip install fpdf2` for PDF output (optional, pure Python, no system deps)

## Install

```bash
cp -r skills/dinosat ~/.claude/skills/dinosat
```

## Run

```
/dinosat
/dinosat src/MyContract.sol
/dinosat --report-only
```

Or just say: "formally verify this contract", "run halmos", "prove the security properties".

## How it works

1. Detects protocol type (AMM, Lending, Vault, etc.) and builds a threat model
2. Scans source for Z3-hostile patterns (`Math.mulDiv`, `Math.sqrt`, assembly, deep branching)
3. Classifies every property before writing tests, so no time is wasted on guaranteed timeouts
4. Generates `test_check_` tests using the right technique per property
5. Generates `test_fuzz_` wrappers using `bound()` for effective fuzz coverage
6. Runs Halmos per-contract, lightest first
7. Runs dual-pass fuzz: 1M runs on fuzz wrappers, 100K on Halmos tests
8. Detects vacuous proofs (tautological assertions, contradictory assumptions)
9. Writes a report with threat model, status legend, soundness caveats, and branded PDF

## Generated output

```
your-project/
├── test/dinosat/
│   ├── PoolSymbolic.t.sol
│   ├── FeeLemmas.t.sol
│   ├── LiquidityLemmas.t.sol
│   └── AccessControlSymbolic.t.sol
├── DINOSAT_REPORT.md
├── DINOSAT_REPORT.pdf
└── dinosat-log.txt
```

## Progress tracking

While running, DinoSAT shows live progress:

```
DinoSAT in progress... [████████████░░░░░░░░] 62% (56/90 tests)
  ✓ FeeLemmasTest: 8 passed, 0 timeout | 12s
  ✓ SwapLemmasTest: 9 passed, 0 timeout | 1s
  ✓ AccessControlSymbolicTest: 5 passed, 24 timeout | 3s
  ▸ PoolSymbolicTest: running...
  ○ VaultSymbolicTest: pending
```

### What a generated test looks like

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";

contract UniswapV2LemmasTest is Test {

    /// @notice Proves swap output never exceeds output reserve.
    ///         Based on Uniswap V2 getAmountOut with 0.3% fee.
    /// @dev Classification: INLINE. Product: (997 * amountIn) * reserveOut fits uint256.
    function test_check_output_lt_reserve(
        uint256 amountIn, uint256 reserveIn, uint256 reserveOut
    ) public pure {
        vm.assume(amountIn >= 1 && amountIn <= type(uint112).max);
        vm.assume(reserveIn >= 1000 && reserveIn <= type(uint112).max);
        vm.assume(reserveOut >= 1000 && reserveOut <= type(uint112).max);

        uint256 amountInWithFee = amountIn * 997;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = (reserveIn * 1000) + amountInWithFee;

        uint256 amountOut = numerator / denominator;
        assert(amountOut < reserveOut);
    }

    /// @notice Fuzz wrapper for test_check_output_lt_reserve
    /// @dev Uses bound() instead of vm.assume() for effective fuzz coverage.
    function test_fuzz_output_lt_reserve(
        uint256 rawIn, uint256 rawResIn, uint256 rawResOut
    ) public pure {
        uint256 amountIn = bound(rawIn, 1, type(uint112).max);
        uint256 reserveIn = bound(rawResIn, 1000, type(uint112).max);
        uint256 reserveOut = bound(rawResOut, 1000, type(uint112).max);

        uint256 amountInWithFee = amountIn * 997;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = (reserveIn * 1000) + amountInWithFee;

        uint256 amountOut = numerator / denominator;
        assert(amountOut < reserveOut);
    }
}
```

### Report excerpt

```markdown
# MyProtocol — Formal Verification Report

**Protocol:** MyProtocol
**Date:** 2026-04-04
**Tool:** Halmos v0.1.13 (Z3 SMT Solver)
**Protocol type:** AMM + Staking

## Verification Status Legend

| Status | Meaning |
|---|---|
| PROVEN | Z3 exhausted all paths. No counterexample exists. Mathematical proof. |
| FUZZ | Verified via 1,000,000 fuzz runs, 0 failures. Not a mathematical proof. |

## Threat Model

### Assets at Risk
- User deposits (ERC-20 tokens deposited as liquidity)
- Share/position value (LP tokens representing pool ownership)

### Attack Vectors Considered
| Attack Vector | Properties | Mitigated By |
|---|---|---|
| Fee exceeding input | FB-1, FB-2 | DECOMPOSE per-branch proofs |
| Share dilution on deposit | LI-1 | CROSSMUL + floor identity |

## Results Summary

| Contract | Tests | Proven | Fuzz | Status |
|---|---|---|---|---|
| FeeLemmasTest | 8 | 8 | 0 | ALL PROVEN |
| PoolSymbolicTest | 15 | 11 | 4 | ALL VERIFIED |
| **TOTAL** | **23** | **19** | **4** | |
```

## Properties checked

| Priority | Category | Count |
|---|---|---|
| P0 | Access control, arithmetic safety | 9 IDs |
| P1 | Fee bounds, liquidity, swap conservation | 16 IDs |
| P2 | Oracle, incentives, scaling, state machine | 21 IDs |
| | Cross-contract | 4 IDs |

## Proof techniques

| # | Technique | What it does |
|---|---|---|
| 1 | Cross-multiplication | Turns `a/b <= c` into `a <= b*c`, eliminating symbolic division |
| 2 | Boundary parameterization | Fixes worst-case variable, proves at the boundary |
| 3 | Concrete enumeration | One test per value for small domains (decimals, pool counts) |
| 4 | Inline mulDiv | Replaces 512-bit `Math.mulDiv` with `(a*b)/c` under safe bounds |
| 5 | Lemma decomposition | Splits multi-branch logic into per-branch proofs |
| 6 | Monotonicity | Proves at boundary, extends via monotonicity argument |
| 7 | Floor division identity | `floor(a/b)*b <= a`, which Z3 handles natively |
| 8 | Abstract specification | Builds Z3-friendly mirror of Math.mulDiv-heavy contracts |

## Protocol types supported

AMM/DEX, Lending, Vault/ERC-4626, Staking, Governance, Token, Bridge, Oracle, and hybrids.

## Z3-hostile patterns and how DinoSAT handles them

| Pattern | Challenge | DinoSAT strategy |
|---|---|---|
| `Math.mulDiv` (512-bit assembly) | Z3 cannot parse the assembly | Replaced with inline `(a*b)/c` under safe bounds (INLINE) |
| `Math.sqrt` | Nonlinear, Z3 intractable | Fuzz-verified with 1M runs (FUZZ) |
| 3+ branches with arithmetic | Combinatorial explosion | Split into per-branch lemma proofs (DECOMPOSE) |
| Symbolic exponentiation (`10**decimals`) | Variable exponent | One test per concrete value (ENUMERATE) |
| Ratio with symbolic denominator | Division intractable | Rewritten as multiplication inequality (CROSSMUL) |

## Out of scope

Halmos verifies single-transaction arithmetic properties. The following require different tools and are not covered:

- Reentrancy (callback-based, multi-call). Use Slither or manual audit.
- Gas optimization. All paths are explored regardless of gas cost.
- Storage layout and proxy compatibility. Use storage layout diffing tools.
- ERC-20 quirks (fee-on-transfer, rebasing). Tokens are assumed standard-compliant.
- Economic incentive alignment. Requires game-theoretic analysis, not SMT.

## Disclaimer

DinoSAT is provided "as is" without warranty of any kind, express or implied. It does not constitute a security audit, and does not guarantee the absence of vulnerabilities, bugs, or exploits. Formal proofs are valid only within the documented input domains. Fuzz-verified properties are empirical observations, not mathematical proofs. Properties outside the scope of the report are not covered, including reentrancy, gas optimization, business logic correctness, economic incentive alignment, and cross-contract interactions. Users assume all risk. The authors accept no liability for any loss or damage arising from the use of this software.

https://dinosat.com
