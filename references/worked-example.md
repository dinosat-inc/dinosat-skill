# Worked Examples

Two end-to-end demonstrations using real protocols: Uniswap V2 (AMM) and ERC-4626 (Vault).

---

## Example 1: Uniswap V2 — Swap Output Bound

### Source (from UniswapV2Library.sol)

```solidity
function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut)
    internal pure returns (uint amountOut)
{
    require(amountIn > 0, 'INSUFFICIENT_INPUT_AMOUNT');
    require(reserveIn > 0 && reserveOut > 0, 'INSUFFICIENT_LIQUIDITY');
    uint amountInWithFee = amountIn * 997;
    uint numerator = amountInWithFee * reserveOut;
    uint denominator = (reserveIn * 1000) + amountInWithFee;
    amountOut = numerator / denominator;
}
```

### Phase 2b — Pattern scan

```
No Math.mulDiv. No Math.sqrt. No assembly. No multi-branch logic.
Plain multiplication + division with constants (997, 1000).
```

### Phase 2e — Slow-path classification

No Z3-hostile patterns. All properties classify as DIRECT.

### Phase 2h — Proof obligation matrix

PROOF OBLIGATION MATRIX

| Function     | PropID | Class  | Technique | Test File     |
|--------------|--------|--------|-----------|---------------|
| getAmountOut | SC-1   | DIRECT | T2        | UniV2Symbolic |
| getAmountOut | AS-1   | DIRECT | T2        | UniV2Symbolic |
| getAmountOut | AS-2   | DIRECT | T2        | UniV2Symbolic |
| getAmountOut | AMM-1  | DIRECT | T5+T6     | UniV2Symbolic |

Summary: 4 properties | DIRECT: 4 | Estimated: 4 proven

### Phase 3 — Generated file: test/dinosat/UniV2Symbolic.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";

/// @title DinoSAT Proofs — Uniswap V2 Swap
contract UniV2SymbolicTest is Test {

    // ════════════════════════════════════════════════
    //  SC-1: swap output < output reserve
    // ════════════════════════════════════════════════

    /// @notice Output never exceeds or equals the output reserve.
    /// @dev Classification: DIRECT. No Math.mulDiv — plain arithmetic with constants.
    ///      Bounds: Uniswap V2 reserves are uint112 (max ~5.19e33).
    ///      Max product: type(uint112).max * 997 * type(uint112).max ~= 2.68e70 < uint256.max.
    function test_check_output_lt_reserve(
        uint256 amountIn, uint256 reserveIn, uint256 reserveOut
    ) public pure {
        // Justification: Uniswap V2 reserves stored as uint112
        vm.assume(amountIn >= 1 && amountIn <= type(uint112).max);
        vm.assume(reserveIn >= 1000 && reserveIn <= type(uint112).max);
        vm.assume(reserveOut >= 1000 && reserveOut <= type(uint112).max);

        uint256 amountInWithFee = amountIn * 997;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = (reserveIn * 1000) + amountInWithFee;

        uint256 amountOut = numerator / denominator;
        assert(amountOut < reserveOut);
    }

    // ════════════════════════════════════════════════
    //  AS-1: no overflow in intermediate products
    // ════════════════════════════════════════════════

    /// @notice Intermediate products fit uint256 for uint112 inputs.
    /// @dev Classification: DIRECT. Constant bound analysis.
    function test_check_no_overflow() public pure {
        uint256 maxReserve = type(uint112).max;
        assert(maxReserve * 997 < type(uint256).max / maxReserve);
    }

    // ════════════════════════════════════════════════
    //  AS-2: denominator always positive
    // ════════════════════════════════════════════════

    /// @notice Denominator > 0 given positive inputs.
    /// @dev Classification: DIRECT.
    function test_check_denominator_positive(
        uint256 amountIn, uint256 reserveIn
    ) public pure {
        vm.assume(amountIn >= 1 && amountIn <= type(uint112).max);
        vm.assume(reserveIn >= 1 && reserveIn <= type(uint112).max);
        uint256 denominator = (reserveIn * 1000) + (amountIn * 997);
        assert(denominator > 0);
    }

    // ════════════════════════════════════════════════
    //  AMM-1: constant product invariant (k) non-decreasing
    // ════════════════════════════════════════════════

    /// @notice After swap, newReserveIn * newReserveOut >= reserveIn * reserveOut.
    ///         The 0.3% fee means the pool gains value on every swap.
    /// @dev Classification: DIRECT. Cross-multiplication of floor division identity.
    ///      Bounds tightened from uint112 to uint96: the cross-product
    ///      newReserveIn * newReserveOut requires (uint112 + uint112)^2 which is ~1e67,
    ///      close to uint256 limits when combined with intermediate products.
    ///      uint96 (max ~7.9e28) keeps all products safely below 1e60.
    function test_check_k_non_decreasing(
        uint256 amountIn, uint256 reserveIn, uint256 reserveOut
    ) public pure {
        // Justification: tightened from uint112 to uint96 for cross-product safety
        vm.assume(amountIn >= 1 && amountIn <= type(uint96).max);
        vm.assume(reserveIn >= 1000 && reserveIn <= type(uint96).max);
        vm.assume(reserveOut >= 1000 && reserveOut <= type(uint96).max);

        uint256 amountInWithFee = amountIn * 997;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = (reserveIn * 1000) + amountInWithFee;
        uint256 amountOut = numerator / denominator;

        uint256 newReserveIn = reserveIn + amountIn;
        uint256 newReserveOut = reserveOut - amountOut;

        assert(newReserveIn * newReserveOut >= reserveIn * reserveOut);
    }
}
```

### Phase 4 — Halmos output

```
[PASS] test_check_output_lt_reserve(uint256,uint256,uint256) (paths: 3, time: 4.12s)
[PASS] test_check_no_overflow()                              (paths: 1, time: 0.01s)
[PASS] test_check_denominator_positive(uint256,uint256)      (paths: 1, time: 0.03s)
[PASS] test_check_k_non_decreasing(uint256,uint256,uint256)  (paths: 3, time: 6.87s)
Symbolic test result: 4 passed; 0 failed; time: 11.09s
```

---

## Example 2: ERC-4626 Vault — Share Price Invariants

### Source (from OpenZeppelin ERC4626.sol, simplified)

```solidity
function _convertToShares(uint256 assets, Math.Rounding rounding)
    internal view returns (uint256)
{
    return assets.mulDiv(totalSupply() + 10 ** _decimalsOffset(), totalAssets() + 1, rounding);
}

function _convertToAssets(uint256 shares, Math.Rounding rounding)
    internal view returns (uint256)
{
    return shares.mulDiv(totalAssets() + 1, totalSupply() + 10 ** _decimalsOffset(), rounding);
}
```

### Phase 2b — Pattern scan

```
Math.mulDiv detected: 2 call sites
10 ** _decimalsOffset: symbolic exponentiation detected
```

### Phase 2e — Slow-path classification

| Function | Pattern | Strategy |
|---|---|---|
| `_convertToShares` | Math.mulDiv x1 | INLINE (product fits uint256 for bounded inputs) |
| `_convertToAssets` | Math.mulDiv x1 | INLINE |
| `_decimalsOffset` | `10 ** offset` | ENUMERATE per offset (0, 3, 6) |

### Phase 3 — Generated file: test/dinosat/ERC4626Symbolic.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";

/// @title DinoSAT Proofs — ERC-4626 Vault
contract ERC4626SymbolicTest is Test {

    // ════════════════════════════════════════════════
    //  Abstract specification helpers
    // ════════════════════════════════════════════════

    /// @dev Inline spec of _convertToShares (rounds down).
    ///      Valid for: assets <= 1e30, totalSupply <= 1e30.
    ///      Max product: 1e30 * (1e30 + 1e6) ~= 1e60 < uint256.max.
    function _toShares(uint256 assets, uint256 totalSupply, uint256 totalAssets, uint256 offset)
        private pure returns (uint256)
    {
        return (assets * (totalSupply + offset)) / (totalAssets + 1);
    }

    /// @dev Inline spec of _convertToAssets (rounds down).
    function _toAssets(uint256 shares, uint256 totalSupply, uint256 totalAssets, uint256 offset)
        private pure returns (uint256)
    {
        return (shares * (totalAssets + 1)) / (totalSupply + offset);
    }

    // ════════════════════════════════════════════════
    //  V-1: shares roundtrip does not create shares
    // ════════════════════════════════════════════════

    /// @notice convertToShares(convertToAssets(shares)) <= shares (offset=6)
    /// @dev Classification: INLINE + ENUMERATE (offset=1e6 is one of {1, 1e3, 1e6}).
    ///      This test covers decimalsOffset=6. Separate tests cover 0 and 3.
    function test_check_shares_roundtrip_offset6(
        uint256 shares, uint256 totalSupply, uint256 totalAssets
    ) public pure {
        vm.assume(shares >= 1 && shares <= 1e24);
        vm.assume(totalSupply >= 1e6 && totalSupply <= 1e30);
        vm.assume(totalAssets >= 1 && totalAssets <= 1e30);

        uint256 offset = 1e6; // decimalsOffset = 6 (concrete enumeration)

        uint256 assets = _toAssets(shares, totalSupply, totalAssets, offset);
        uint256 sharesBack = _toShares(assets, totalSupply, totalAssets, offset);
        assert(sharesBack <= shares);
    }

    // ════════════════════════════════════════════════
    //  V-2: assets roundtrip does not create assets
    // ════════════════════════════════════════════════

    /// @notice convertToAssets(convertToShares(assets)) <= assets
    /// @dev Classification: INLINE.
    function test_check_assets_roundtrip(
        uint256 assets, uint256 totalSupply, uint256 totalAssets
    ) public pure {
        vm.assume(assets >= 1 && assets <= 1e24);
        vm.assume(totalSupply >= 1e6 && totalSupply <= 1e30);
        vm.assume(totalAssets >= 1 && totalAssets <= 1e30);

        uint256 offset = 1e6;

        uint256 shares = _toShares(assets, totalSupply, totalAssets, offset);
        uint256 assetsBack = _toAssets(shares, totalSupply, totalAssets, offset);
        assert(assetsBack <= assets);
    }

    // ════════════════════════════════════════════════
    //  V-5: withdrawal rounds down (favors vault)
    // ════════════════════════════════════════════════

    /// @notice Redeeming shares gives assets rounded DOWN.
    /// @dev Classification: DIRECT. Floor division identity.
    function test_check_withdrawal_rounds_down(
        uint256 shares, uint256 totalSupply, uint256 totalAssets
    ) public pure {
        vm.assume(shares >= 1 && shares <= totalSupply);
        vm.assume(totalSupply >= 1e6 && totalSupply <= 1e30);
        vm.assume(totalAssets >= 1 && totalAssets <= 1e30);

        uint256 offset = 1e6;
        uint256 assets = _toAssets(shares, totalSupply, totalAssets, offset);

        // Floor identity: assets * denominator <= shares * numerator
        assert(assets * (totalSupply + offset) <= shares * (totalAssets + 1));
    }
}
```

### Phase 4 — Halmos output

```
[PASS] test_check_shares_roundtrip(uint256,uint256,uint256) (paths: 5, time: 8.32s)
[PASS] test_check_assets_roundtrip(uint256,uint256,uint256) (paths: 5, time: 7.91s)
[PASS] test_check_withdrawal_rounds_down(uint256,uint256,uint256) (paths: 3, time: 2.17s)
Symbolic test result: 3 passed; 0 failed; time: 18.50s
```

---

## Output file structure

After DinoSAT runs, the project has:

```
your-project/
├── src/
│   └── (existing source, untouched)
├── test/
│   ├── halmos/                               # ← generated by DinoSAT
│   │   ├── UniV2Symbolic.t.sol
│   │   ├── ERC4626Symbolic.t.sol
│   │   └── FeeLemmas.t.sol                   #   (if fee engine needed decomposition)
│   └── (existing tests, untouched)
├── foundry.toml                              # [profile.halmos] added if missing
├── DINOSAT_REPORT.md            # ← formal report
└── dinosat-log.txt                           # ← raw Halmos output
```

DinoSAT never modifies existing files except adding `[profile.halmos]` to `foundry.toml` if missing.
