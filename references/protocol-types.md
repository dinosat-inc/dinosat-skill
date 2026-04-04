# Protocol Type Profiles

Protocol-type detection and targeted property selection. Identify the protocol archetype first, then prioritize the properties most relevant to that type's attack surface.

---

## Detection

Scan source code for these indicators to classify the protocol:

| Archetype | Detection Signals |
|---|---|
| **AMM / DEX** | `swap`, `addLiquidity`, `removeLiquidity`, `getAmountOut`, reserve pairs, constant product/sum, LP tokens |
| **Lending** | `borrow`, `repay`, `liquidate`, `collateral`, `healthFactor`, interest rate models |
| **Vault / ERC-4626** | `deposit`, `withdraw`, `redeem`, `convertToShares`, `convertToAssets`, `totalAssets` |
| **Staking** | `stake`, `unstake`, `claimRewards`, `rewardPerToken`, penalty mechanisms |
| **Governance** | `propose`, `vote`, `execute`, `queue`, timelock, quorum, voting power |
| **Token** | `mint`, `burn`, `transfer`, `approve`, `permit`, supply caps, blacklists |
| **Bridge** | `lock`, `unlock`, `relay`, `prove`, message passing, merkle proofs, nonce tracking |
| **Oracle** | `updatePrice`, `getPrice`, TWAP, deviation checks, staleness checks, multi-source aggregation |

A protocol may be **hybrid** (e.g., AMM + Staking + Oracle). Apply properties from ALL matching archetypes.

---

## AMM / DEX

**Primary attack surfaces:** price manipulation, reserve drainage, sandwich attacks, rounding exploitation.

**Critical properties (always verify):**

| ID | Property | Why Critical |
|---|---|---|
| FB-1 | fee < amountIn | Fee exceeding input drains user funds |
| SC-1 | swap output < output reserve | Output exceeding reserve drains pool |
| SC-2 | pool value non-decreasing after swap | Value leak enables arbitrage theft |
| LI-1 | share price non-decreasing on deposit | Share dilution steals from LPs |
| LI-2 | no value extraction on withdrawal | Burn rounding extracts LP value |
| LI-3 | full burn returns all reserves | Locked funds if burn is incomplete |
| AS-1 | no overflow in swap math | Overflow can produce wrong output |

**AMM-specific properties:**

| ID | Property | Technique |
|---|---|---|
| AMM-1 | Constant product/sum invariant holds post-swap | INLINE + CROSSMUL |
| AMM-2 | First-deposit dead shares prevent inflation attack | DIRECT |
| AMM-3 | Imbalance threshold prevents extreme ratio (e.g., 99:1) | BOUNDARY |
| AMM-4 | Price oracle (TWAP) bounded by max tracked price | DIRECT |
| AMM-5 | Minimum swap amount enforced | DIRECT |

---

## Lending

**Primary attack surfaces:** oracle manipulation, liquidation race conditions, interest rate manipulation, bad debt.

**Critical properties:**

| ID | Property | Why Critical |
|---|---|---|
| LEND-1 | Health factor computation correct | Wrong HF allows undercollateralized borrows |
| LEND-2 | Liquidation threshold enforced | Bypassing threshold creates bad debt |
| LEND-3 | Interest accrual monotonically increasing | Decreasing interest = protocol loss |
| LEND-4 | Collateral value >= debt value at borrow time | Undercollateralized position from rounding |
| LEND-5 | Collateral seized <= repaid debt value * (1 + bonus rate) | Over-seizure drains borrower |
| AS-2 | Division by zero impossible in rate model | Rate model /0 bricks protocol |

**Technique:** Most lending properties involve `Math.mulDiv` for rate/price calculations. Use INLINE + DECOMPOSE for multi-step interest accrual.

---

## Vault / ERC-4626

**Primary attack surfaces:** share inflation, donation attacks, first-depositor manipulation, rounding direction.

**Critical properties:**

| ID | Property | Why Critical |
|---|---|---|
| V-1 | `convertToShares(convertToAssets(shares)) <= shares` | Round-trip must not create shares |
| V-2 | `convertToAssets(convertToShares(assets)) <= assets` | Round-trip must not create assets |
| V-3 | Share price non-decreasing after deposit | Inflation attack steals from depositors |
| V-4 | First deposit: dead shares prevent share price manipulation | Classic ERC-4626 attack vector |
| V-5 | Withdrawal rounds down (favors vault) | Wrong rounding direction leaks value |
| V-6 | Deposit rounds down shares (favors vault) | Wrong rounding direction leaks value |
| V-7 | `totalAssets >= totalSupply * minSharePrice` | Share price floor maintained |

**Technique:** CROSSMUL for rounding proofs. DIRECT for dead shares. Floor division identity is the core tool.

---

## Staking

**Primary attack surfaces:** reward manipulation, flash-stake, penalty avoidance, accounting desync.

**Critical properties:**

| ID | Property | Why Critical |
|---|---|---|
| STK-1 | Stake accounting conservation | Tracking error creates phantom stakes |
| STK-2 | Unstake penalty correctly applied | Penalty avoidance = free unstaking |
| STK-3 | Reward per token monotonically non-decreasing | Decreasing RPT = lost rewards |
| STK-4 | Total staked across pools == sum of per-pool totals | Desync enables overwithdrawal |
| STK-5 | Penalty decomposition sums to total | Fee leak from rounding in split |
| STK-6 | One-stake-per-block enforced | Flash-stake captures disproportionate rewards |

**Technique:** DIRECT for accounting (linear arithmetic). INLINE for reward calculations with mulDiv.

---

## Governance

**Primary attack surfaces:** flash-loan voting, quorum manipulation, timelock bypass, proposal poisoning.

**Critical properties:**

| ID | Property | Why Critical |
|---|---|---|
| GOV-1 | Quorum threshold enforced | Low quorum enables minority takeover |
| GOV-2 | Timelock delay cannot be bypassed | Instant execution enables rug pull |
| GOV-3 | Vote weight snapshot at proposal time | Flash-loan voting with current balance |
| GOV-4 | Proposal cannot be executed twice | Double execution of state changes |
| GOV-5 | Cancel clears all pending state | Stale proposals execute later |

**Technique:** DIRECT for state machine properties. SM-1 through SM-5 apply directly.

---

## Token (ERC-20 / Custom)

**Primary attack surfaces:** infinite mint, supply manipulation, approval race, blacklist bypass.

**Critical properties:**

| ID | Property | Why Critical |
|---|---|---|
| TOK-1 | Total supply == sum of all balances | Supply desync enables inflation |
| TOK-2 | Transfer: sender balance decreases, receiver increases by same amount | Value creation/destruction |
| TOK-3 | Supply cap enforced on mint | Uncapped mint = infinite inflation |
| TOK-4 | Burn reduces supply and balance equally | Burn/supply desync |
| TOK-5 | Approve does not transfer tokens | Approval confusion |

**Technique:** DIRECT for all (linear arithmetic, state tracking).

---

## Bridge

**Primary attack surfaces:** double-spend, replay attacks, message forgery, nonce reuse.

**Critical properties:**

| ID | Property | Why Critical |
|---|---|---|
| BRG-1 | Nonce strictly increasing (no reuse) | Replay attack re-executes messages |
| BRG-2 | Lock amount == unlock amount (cross-chain conservation) | Value creation in transit |
| BRG-3 | Message hash includes all critical fields | Forgery from incomplete hash |
| BRG-4 | Merkle proof verification correct | False proof unlocks without lock |

**Technique:** DIRECT for nonce/hash properties. Merkle proofs may require `--loop N`.

---

## Oracle

**Primary attack surfaces:** stale prices, manipulation via donation/flashloan, extreme rate drift.

**Critical properties:**

| ID | Property | Why Critical |
|---|---|---|
| OR-1 | Rate never zero after adjustment | Zero rate bricks the protocol |
| OR-2 | Adjustment direction correct | Wrong direction amplifies imbalance |
| OR-3 | Adjustment magnitude bounded | Unbounded adjustment = instant manipulation |
| OR-4 | TWAP bounded by max price | Unbounded TWAP = price oracle attack |
| OR-5 | Staleness gating (no double-update per period) | Stale price enables arbitrage |
| OR-6 | Cumulative accumulator monotonic | Decreasing accumulator corrupts TWAP |

**Technique:** INLINE for rate math. DIRECT for gating/bounds.

---

## Hybrid Protocol Selection

For hybrid protocols, merge property lists:

1. Detect all matching archetypes
2. Union all "Critical properties" tables
3. Deduplicate by property ID (same ID from different archetypes = one test)
4. Add archetype-specific properties
5. Prioritize: P0 from any archetype first, then P1, then P2
