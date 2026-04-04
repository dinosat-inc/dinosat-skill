# Test Templates

Reusable code patterns for Halmos symbolic tests. Each template specifies: applicable property IDs, classification, and complete implementation.

---

## T1: File Boilerplate

Every test file MUST begin with this structure.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20; // Match your project's pragma version exactly

import {Test} from "forge-std/Test.sol";
// Import project constants:
// import {Constants} from "../../src/libraries/Constants.sol";
// Import libraries under test:
// import {PoolMath} from "../../src/libraries/PoolMath.sol";
// import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

/// @title Halmos Symbolic Proofs — <ContractName>
/// @notice Halmos: FOUNDRY_PROFILE=$HALMOS_PROFILE halmos --contract <Name>Test --function "test_check_" --forge-build-out out-halmos --solver-timeout-assertion 30000
/// @notice Forge:  forge test --match-contract <Name>Test --match-test "test_check_"
contract <Name>Test is Test {
    uint256 constant P = 1e18; // Replace with project PRECISION

    // ════════════════════════════════════════════════
    //  <Section>
    // ════════════════════════════════════════════════
}
```

---

## T2: Constant Bound (DIRECT)

**Properties:** AS-1, FB-2, OR-3, IC-4

```solidity
/// @notice Proves <value> is strictly bounded by <BOUND>
function test_check_<name>_bounded(uint256 x) public pure {
    vm.assume(x >= MIN && x <= MAX);
    uint256 result = /* computation */;
    assert(result < BOUND);
}
```

---

## T3: Access Control Guard (DIRECT)

**Properties:** AC-1

For stateful tests (deploying the actual contract):
```solidity
/// @notice Proves <function> reverts for unauthorized callers
function test_check_<function>_rejects_unauthorized(address caller) public {
    vm.assume(caller != address(0));
    vm.assume(caller != authorizedAddress);
    vm.prank(caller);
    try contract.<function>(<args>) {
        assert(false); // should have reverted
    } catch {
        assert(true); // correctly rejected
    }
}
```

For pure logic tests (when the guard condition is modeled without deployment):
```solidity
/// @notice Proves role separation: admin and validator are always different addresses
function test_check_admin_validator_different(address admin, address validator) public pure {
    vm.assume(admin != address(0));
    vm.assume(validator != address(0));
    vm.assume(admin != validator); // constructor enforces this
    // After any role change, the invariant holds if the change function checks it
    assert(admin != validator);
}
```

**Note:** The pure version is limited — it only proves the guard condition logic, not that the contract actually enforces it. Use the stateful version (with `try/catch` or `vm.prank`) when Halmos supports the needed cheat codes.

---

## T4: Fee Per-Branch Theorem (DECOMPOSE)

**Properties:** FB-1, FB-2

```solidity
/// @notice Proves fee < amountIn for branch <X>
/// @dev Technique: lemma decomposition — branch isolated via vm.assume
function test_check_theorem_fee_branch<X>(
    uint256 amountIn, uint256 baseFee, uint256 deviation
) public pure {
    vm.assume(amountIn >= 1 && amountIn <= 1e27);
    vm.assume(baseFee >= BASE_FEE_MIN && baseFee <= BASE_FEE_MAX);
    vm.assume(/* branch condition */);

    uint256 effectiveFee = /* branch-specific computation */;
    uint256 fee = (amountIn * effectiveFee) / P;
    assert(fee < amountIn);
}
```

---

## T5: Cross-Multiplication Ratio (CROSSMUL)

**Properties:** IC-3, LI-1, LI-2

```solidity
/// @notice Proves ratio <= THRESHOLD via cross-multiplication
/// @dev Technique: cross-multiplication eliminates symbolic division
function test_check_<name>_ratio_bounded(uint256 numerator, uint256 denominator) public pure {
    vm.assume(numerator >= 1 && numerator <= MAX);
    vm.assume(denominator >= numerator && denominator <= MAX);
    // ratio = numerator * P / denominator <= THRESHOLD
    // Cross-multiply: numerator * P <= THRESHOLD * denominator
    assert(numerator * P <= THRESHOLD * denominator);
}
```

---

## T6: Floor Division Identity (DIRECT)

**Properties:** LI-2, LI-4, TS-2, IC-2

```solidity
/// @notice Proves floor(a*b/c) * c <= a*b (rounding identity)
/// @dev Technique: floor division identity — Z3 bitvector tautology
function test_check_<name>_floor_identity(uint256 a, uint256 c) public pure {
    vm.assume(a >= 1 && a <= MAX_A);
    vm.assume(c >= 1 && c <= MAX_C);
    uint256 result = a / c;
    assert(result * c <= a);
    assert(a - result * c < c);
}
```

---

## T7: No Value Extraction on Withdrawal (CROSSMUL + Floor Identity)

**Properties:** LI-2

```solidity
/// @notice Proves priceAfter >= priceBefore after withdrawal.
/// @dev Overflow check: outA (max 1e24) * totalSupply (max 1e24) = max 1e48 < uint256.max.
///      burnAmount (max 1e24) * resA (max 1e24) = max 1e48 < uint256.max.
///      (outA+outB) (max 2e24) * totalSupply (max 1e24) = max 2e48 < uint256.max.
/// @dev Technique: floor division identity + cross-multiplication
function test_check_noValueExtraction(
    uint256 resA, uint256 resB, uint256 totalSupply, uint256 burnAmount
) public pure {
    vm.assume(resA >= 1e18 && resA <= 1e24);
    vm.assume(resB >= 1e18 && resB <= 1e24);
    vm.assume(totalSupply >= 1e18 && totalSupply <= 1e24);
    vm.assume(burnAmount >= 1 && burnAmount <= totalSupply - 1);

    uint256 outA = (burnAmount * resA) / totalSupply;
    uint256 outB = (burnAmount * resB) / totalSupply;

    // Floor identity: outA * ts <= burn * resA
    assert(outA * totalSupply <= burnAmount * resA);
    assert(outB * totalSupply <= burnAmount * resB);
    // Sum: (outA+outB)*ts <= burn*(resA+resB)
    // Cross-multiplied form of priceAfter >= priceBefore
    assert((outA + outB) * totalSupply <= burnAmount * (resA + resB));
}
```

---

## T8: Share Price Non-Decreasing on Deposit (CROSSMUL + Floor Identity)

**Properties:** LI-1

```solidity
/// @notice Proves share price does not decrease when adding liquidity
/// @dev Technique: floor identity on LP mint + cross-multiplication
function test_check_sharePrice_nonDecreasing(
    uint256 resA, uint256 resB, uint256 totalSupply, uint256 addA
) public pure {
    vm.assume(resA >= 1e18 && resA <= 1e24);
    vm.assume(resB >= 1e18 && resB <= 1e24);
    vm.assume(totalSupply >= 1e18 && totalSupply <= 1e24);
    vm.assume(addA >= 1e18 && addA <= 1e24);

    uint256 addB = (addA * resB) / resA;
    vm.assume(addB > 0);

    uint256 liqA = (addA * totalSupply) / resA;
    uint256 liqB = (addB * totalSupply) / resB;
    uint256 liq = liqA < liqB ? liqA : liqB;
    vm.assume(liq > 0);

    // Floor identity: liq * resA <= addA * ts, liq * resB <= addB * ts
    // Therefore: liq * (resA+resB) <= (addA+addB) * ts
    // Cross-multiplied form of priceAfter >= priceBefore
    assert(liq * resA <= addA * totalSupply);
    assert(liq * resB <= addB * totalSupply);
}
```

---

## T9: Concrete Enumeration (ENUMERATE)

**Properties:** IC-2, TS-1, TS-2

```solidity
/// @notice Proves <property> for N=<VALUE>
/// @dev Technique: concrete enumeration — constant divisor is Z3-tractable
function test_check_<name>_<VALUE>(uint256 x) public pure {
    vm.assume(x >= 1 && x <= 1e30);
    uint256 share = x / CONCRETE_DIVISOR;
    vm.assume(share > 0);
    uint256 remainder = x - (share * (CONCRETE_DIVISOR - 1));
    assert(remainder >= share);
    assert(remainder <= share + CONCRETE_DIVISOR - 1);
}
```

---

## T10: Inline mulDiv Helper (INLINE)

**Properties:** OR-1, OR-2, SC-1, SC-4

```solidity
/// @dev Inline mulDiv replacement. REQUIRES: a * b fits uint256.
///      Use vm.assume() to constrain inputs before calling.
function _mulDiv(uint256 a, uint256 b, uint256 c) private pure returns (uint256) {
    return (a * b) / c;
}
```

---

## T11: Monotonicity Proof (BOUNDARY)

**Properties:** LI-1, IC-3, FB-5

```solidity
/// @notice Proves f is non-decreasing in x via cross-multiplication
/// @dev Technique: monotonicity — x/(x+c) increasing when c > 0
function test_check_<name>_monotonic(
    uint256 x1, uint256 x2, uint256 c
) public pure {
    vm.assume(x1 >= MIN && x1 <= MAX - 1);
    vm.assume(x2 >= x1 + 1 && x2 <= MAX);
    vm.assume(c >= MIN_C && c <= MAX_C);
    // x2/(x2+c) >= x1/(x1+c) iff x2*(x1+c) >= x1*(x2+c) iff x2*c >= x1*c
    assert(x2 * c >= x1 * c);
}
```

---

## T12: State Gating (DIRECT)

**Properties:** SM-1, OR-5

```solidity
/// @notice Proves period-based gating is consistent with timestamp math.
///         Two timestamps in the same period produce the same period index.
///         Two timestamps in different periods produce different indices.
/// @dev Technique: direct — floor division properties.
function test_check_<name>_gating(uint256 ts1, uint256 ts2) public pure {
    vm.assume(ts1 >= 1 hours && ts1 <= type(uint128).max);
    vm.assume(ts2 >= 1 hours && ts2 <= type(uint128).max);
    uint256 period1 = ts1 / PERIOD;
    uint256 period2 = ts2 / PERIOD;
    // If both timestamps are in the same period window, their indices match
    if (ts1 / PERIOD == ts2 / PERIOD) {
        assert(period1 == period2);
    }
    // Prove: advancing by at least PERIOD guarantees a new period
    if (ts2 >= ts1 + PERIOD) {
        assert(period2 > period1);
    }
}
```

---

## T13: Sort Correctness (DIRECT + --loop)

**Properties:** SM-3

```solidity
/// @notice Proves bubble sort produces sorted output
/// @dev Requires: halmos --loop 5
function test_check_sort_correctness(
    uint256 a, uint256 b, uint256 c, uint256 d, uint256 e
) public pure {
    vm.assume(a <= 1e30 && b <= 1e30 && c <= 1e30 && d <= 1e30 && e <= 1e30);
    uint256[5] memory arr = [a, b, c, d, e];
    for (uint8 i = 0; i < 4; i++) {
        for (uint8 j = 0; j < 4 - i; j++) {
            if (arr[j] < arr[j + 1]) (arr[j], arr[j + 1]) = (arr[j + 1], arr[j]);
        }
    }
    assert(arr[0] >= arr[1]);
    assert(arr[1] >= arr[2]);
    assert(arr[2] >= arr[3]);
    assert(arr[3] >= arr[4]);
}
```

---

## T14: Conservation / Fee Split (DIRECT)

**Properties:** PD-1, PD-2, IC-1

```solidity
/// @notice Proves fee split components sum to total
/// @dev Technique: direct — linear arithmetic
function test_check_<name>_conservation(uint256 total) public pure {
    vm.assume(total >= 1 && total <= 1e30);
    uint256 partA = total / 4;
    uint256 partB = total - partA;
    assert(partA + partB == total);
    assert(partA <= total);
    assert(partB <= total);
    assert(partB >= total * 3 / 4);
}
```

---

## T15: Boundary Parameterization (BOUNDARY)

**Properties:** IC-3, LI-1

```solidity
/// @notice Proves cap at worst-case boundary
/// @dev Technique: boundary parameterization — valueA fixed at max, resB symbolic
///      Monotonicity: ratio increasing in valueA, so max valueA is worst case
function test_check_<name>_cap_at_boundary(uint256 resB) public pure {
    vm.assume(resB >= 1e6 && resB <= type(uint96).max);
    uint256 valueA = 99 * resB;  // boundary: maximum before cap
    uint256 total = valueA + resB;
    assert(valueA * P <= THRESHOLD * total);
}
```

---

## T16: Stateful Contract Interaction (DIRECT)

**Properties:** AC-1, AC-2, SM-1 (any property requiring deployed contract state)

**Note:** This template requires deploying the contract under test. It uses `setUp()` and `vm.prank`. Halmos supports these cheat codes. Use this when pure-math tests are insufficient (e.g., testing actual access control enforcement, not just the guard condition).

```solidity
import {<Contract>} from "../../src/<path>/<Contract>.sol";

contract <Contract>StatefulTest is Test {
    <Contract> target;
    address admin = address(0xAD);
    address attacker = address(0xBAD);

    function setUp() public {
        target = new <Contract>(admin, /* constructor args */);
    }

    /// @notice Proves <function> reverts when called by non-admin
    /// @dev Classification: DIRECT. Stateful — deploys contract.
    function test_check_<function>_rejects_nonAdmin(address caller) public {
        vm.assume(caller != admin);
        vm.assume(caller != address(0));
        vm.prank(caller);
        try target.<function>(/* args */) {
            assert(false); // must revert
        } catch {
            assert(true); // correctly rejected
        }
    }

    /// @notice Proves <function> succeeds when called by admin
    /// @dev Classification: DIRECT. Stateful.
    function test_check_<function>_succeeds_for_admin() public {
        vm.prank(admin);
        // Should not revert
        target.<function>(/* args */);
        assert(true);
    }
}
```

**Pragma note:** Match the project's Solidity pragma version exactly (e.g., `pragma solidity ^0.8.30;` if the project uses 0.8.30).

---

## T17: Fuzz Wrapper (FUZZ companion to any test_check_)

**Properties:** Any — mirrors the corresponding `test_check_` function

Halmos tests use `vm.assume` for tight bounds, which causes near-100% rejection under Forge fuzz (random `uint256` values almost never land in `[1e18, 1e21]`). Fuzz wrappers use `bound()` to map random inputs into valid ranges, giving real input coverage.

Generate one `test_fuzz_` wrapper for every `test_check_` that has 2+ `vm.assume` bounds on `uint256` parameters.

```solidity
/// @notice Fuzz wrapper for test_check_<name>
/// @dev Uses bound() instead of vm.assume() for effective fuzz coverage.
///      Halmos ignores test_fuzz_ prefix; Forge discovers and fuzzes it.
function test_fuzz_<name>(
    uint256 rawA, uint256 rawB, uint256 rawC
) public pure {  // match visibility of test_check_ counterpart
    // Map random inputs to valid ranges (replaces vm.assume bounds)
    uint256 a = bound(rawA, MIN_A, MAX_A);
    uint256 b = bound(rawB, MIN_B, MAX_B);
    uint256 c = bound(rawC, MIN_C, MAX_C);

    // Relationship constraints: use early return instead of vm.assume
    // (Forge counts returns as passes, not rejections)
    if (c > a) return;  // replaces: vm.assume(c <= a)
    if (b == 0) return; // replaces: vm.assume(b > 0)

    // ... exact same computation and assertions as test_check_<name> ...
    uint256 result = /* same logic */;
    assert(result < BOUND);
}
```

**Conversion rules:**
| `test_check_` pattern | `test_fuzz_` replacement |
|---|---|
| `vm.assume(x >= LO && x <= HI)` | `uint256 x = bound(rawX, LO, HI)` |
| `vm.assume(a <= b)` | `a = bound(rawA, MIN_A, b)` or `if (a > b) return` |
| `vm.assume(x != 0)` | `if (x == 0) return` (after `bound`) |
| `vm.assume(x >= y + 1)` | `if (x < y + 1) return` (after `bound`) |
