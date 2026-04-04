# Security Property Catalog

Ten categories of verifiable security properties. For each: detection heuristic, formal specification, typical classification, and priority level.

---

## P0 — Critical (verify first)

### 1. Access Control

**Detect:** Modifiers (`onlyOwner`, `onlyAdmin`, `onlyRole`), `require(msg.sender == ...)`, multi-step ceremonies (propose/execute).

**Properties:**
- AC-1: Every guarded function reverts for unauthorized callers
- AC-2: Role separation invariants (e.g., `admin != validator` always)
- AC-3: Irreversibility of locks (once `locked = true`, cannot set `false`)
- AC-4: Pending state clears after execution or cancellation
- AC-5: Two-step ownership transfer correctly updates both old and new roles

**Classification:** DIRECT (no arithmetic, state-based logic only)

**Note:** Tests using `vm.expectRevert` may fail in Halmos due to unsupported cheat codes. Alternative: test the guard condition as a pure boolean assertion.

### 2. Arithmetic Safety

**Detect:** All `*`, `/`, `-` operations in financial logic. Focus on `mulDiv`, `**`, and unchecked blocks.

**Properties:**
- AS-1: No overflow in constrained ranges (compute max product, verify < `uint256.max`)
- AS-2: Division by zero impossible (denominator > 0 given input constraints)
- AS-3: Subtraction underflow impossible (minuend >= subtrahend)
- AS-4: Cast safety (value fits target type after narrowing)

**Classification:** DIRECT (bound analysis is straightforward)

---

## P1 — High (verify second)

### 3. Fee Bounds

**Detect:** Fee computation: `amount * rate / PRECISION`, fee engine branches, min/max enforcement.

**Properties:**
- FB-1: `fee < amountIn` for all branches (critical: prevents draining user funds)
- FB-2: `effectiveFee < PRECISION` for all branches
- FB-3: `fee >= minFee` when minimum enforced
- FB-4: `fee <= maxFee` when maximum enforced
- FB-5: Fee monotonically non-decreasing with input size

**Classification:** DECOMPOSE for multi-branch engines. Per-branch theorems are DIRECT.

### 4. Liquidity Invariants

**Detect:** `mint`/`burn` functions, LP token issuance, `addLiquidity`/`removeLiquidity`.

**Properties:**
- LI-1: Share price non-decreasing on deposit (floor division rounds LP down)
- LI-2: No value extraction on withdrawal (floor division rounds output down)
- LI-3: Full burn (`liquidity == totalSupply`) returns all reserves exactly
- LI-4: Burn output bounded by reserves (`outA <= resA`, `outB <= resB`)
- LI-5: Proportional mint always produces positive LP tokens
- LI-6: Mint-burn roundtrip: user never profits

**Classification:** CROSSMUL + DIRECT. Use floor division identity for rounding proofs.

### 5. Swap Conservation

**Detect:** `swap` functions, AMM output calculations, reserve update logic.

**Properties:**
- SC-1: Swap output < output reserve (never drains pool)
- SC-2: Pool value non-decreasing after swap
- SC-3: Vault/protocol fee is correct fraction of total fee
- SC-4: Effective price bounded by base rate
- SC-5: Minimum swap fee enforced

**Classification:** DIRECT for SC-3/SC-5. INLINE or FUZZ for SC-1/SC-2/SC-4 (depends on Math.mulDiv usage).

---

## P2 — Medium (verify third)

### 6. Oracle Mechanics

**Detect:** Rate adjustment functions, TWAP accumulators, `updateRate`, adaptive parameters.

**Properties:**
- OR-1: Rate never goes to zero after adjustment
- OR-2: Adjustment direction correct (decrease when overbalanced, increase when under)
- OR-3: Adjustment magnitude bounded by maximum step
- OR-4: TWAP bounded by maximum tracked price
- OR-5: Gating: no double-update within same period

**Classification:** INLINE for OR-1/OR-2 (Math.mulDiv). DIRECT for OR-3/OR-4/OR-5.

### 7. Incentive Caps

**Detect:** Distribution functions, reward caps, `maxReserveRoom`, threshold enforcement.

**Properties:**
- IC-1: Distribution sums to total (no tokens created or lost)
- IC-2: Remainder bounded (last recipient gets at most N-1 extra wei)
- IC-3: Cap prevents threshold breach (value ratio stays below maximum)
- IC-4: Room calculation non-negative (no underflow)
- IC-5: No incentive when already at threshold

**Classification:** CROSSMUL for IC-3. ENUMERATE for IC-2. BOUNDARY for IC-3 with multiple variables. DIRECT for IC-4/IC-5.

### 8. Token Scaling

**Detect:** `scaleUp`/`scaleDown`, `10 ** (18 - decimals)`, decimal normalization.

**Properties:**
- TS-1: Scale up then down roundtrip is lossless
- TS-2: Scale down floor loss bounded by one unit of smallest denomination
- TS-3: No overflow in scaleUp for realistic token amounts

**Classification:** ENUMERATE per decimal value (6, 8, 12, 18).

### 9. Penalty/Fee Decomposition

**Detect:** Fee splits (protocol/pool/burn/treasury), penalty extraction, service fees.

**Properties:**
- PD-1: Components sum to total (conservation)
- PD-2: Each component bounded by total
- PD-3: Correct proportions (service fee = penalty * rate / PRECISION)

**Classification:** DIRECT (linear arithmetic)

### 10. State Machine Invariants

**Detect:** Queues, cycles, epochs, gating conditions, sorting algorithms.

**Properties:**
- SM-1: Gating enforced (hourly skip, one-action-per-block)
- SM-2: Queue ordering preserved
- SM-3: Sort correctness (output is sorted)
- SM-4: Pagination bounds (offset + count <= total)
- SM-5: Cycle advancement correct

**Classification:** DIRECT for SM-1/SM-4/SM-5. `--loop N` for SM-2/SM-3.

---

## Verification Priority Order

When analyzing a contract, extract and verify properties in this order:

1. **P0: Access Control** (AC-1 through AC-5)
2. **P0: Arithmetic Safety** (AS-1 through AS-4)
3. **P1: Fee Bounds** (FB-1 through FB-5)
4. **P1: Liquidity Invariants** (LI-1 through LI-6)
5. **P1: Swap Conservation** (SC-1 through SC-5)
6. **P2: Oracle Mechanics** (OR-1 through OR-5)
7. **P2: Incentive Caps** (IC-1 through IC-5)
8. **P2: Token Scaling** (TS-1 through TS-3)
9. **P2: Penalty Decomposition** (PD-1 through PD-3)
10. **P2: State Machine** (SM-1 through SM-5)

Skip categories that do not apply to the target contract.

---

## Cross-Contract Properties

When a protocol has multiple interacting contracts, verify these cross-boundary invariants:

### CC-1: Accounting Consistency
Total tracked across child contracts equals parent's aggregate.
Example: `stakingManager.totalStakedAcrossAllPools() == sum(totalStakedInPool[pool_i])`

### CC-2: Access Control Chain
If contract A calls contract B's guarded function, verify A has the required role.
Example: `pool.updateReward()` requires `msg.sender == STAKING_MANAGER`

### CC-3: Token Flow Conservation
Tokens leaving contract A must arrive at contract B. No creation or destruction in transit.
Example: `tinoSwap.receivePenalty(amount)` after `safeTransfer(tinoSwap, amount)`

### CC-4: Oracle Consistency
Rate updates in the oracle contract produce valid rates in all consumer contracts.
Example: `oracle.updatePool()` calls `pool.updateBaseRate(newRate)` — rate stays in valid range.

**Classification:** Cross-contract properties are typically FUZZ (require deployed state) or DIRECT (if modeled as pure arithmetic relationships between parameters).

**Technique:** Model the cross-contract relationship as a single pure function with parameters representing each contract's relevant state. This avoids deploying actual contracts in Halmos.
