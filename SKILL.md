---
name: dinosat-skill
description: "Formal verification of Solidity smart contracts via Halmos symbolic execution and Z3 SMT solver. Generates proof tests, runs Halmos per-contract, produces a verification report. Triggers on 'formally verify', 'halmos', 'symbolic tests', 'prove this contract', 'formal verification', 'dinosat'."
---

# DinoSAT — Formal Verification for Solidity

Orchestrator for a formal verification pipeline. Reads Solidity contracts, detects Z3-hostile patterns, generates symbolic test suites, runs Halmos/Z3, and produces a verification report.

`$SKILL_DIR` = the directory containing this SKILL.md file. Resolve it from the path you loaded this skill from.

## Mode Selection

- **Default** (no arguments): verify all `.sol` files in `src/` (exclude `interfaces/`, `lib/`, `mocks/`, `test/`).
- **`$filename ...`**: verify the specified file(s) only.
- **`--report-only`**: skip test generation, run Halmos on existing `test/dinosat/` tests and produce report.

---

Before doing anything else, print this exactly:

Before doing anything else, output this text directly (not via a tool call — just print it as your response text):

```
██████╗ ██╗███╗   ██╗ ██████╗ ███████╗ █████╗ ████████╗
██╔══██╗██║████╗  ██║██╔═══██╗██╔════╝██╔══██╗╚══██╔══╝
██║  ██║██║██╔██╗ ██║██║   ██║███████╗███████║   ██║
██║  ██║██║██║╚██╗██║██║   ██║╚════██║██╔══██║   ██║
██████╔╝██║██║ ╚████║╚██████╔╝███████║██║  ██║   ██║
╚═════╝ ╚═╝╚═╝  ╚═══╝ ╚═════╝ ╚══════╝╚═╝  ╚═╝   ╚═╝

Formal Verification Engine | https://dinosat.com
```

---

## Orchestration

### Phase 1 — Environment & Discovery

Make these parallel tool calls in one message:

a. Bash `forge --version` — verify Foundry installed
b. Bash `halmos --version` — verify Halmos installed (require v0.1.13+)
c. Read `foundry.toml` — check for `[profile.halmos]` section
d. Bash `find src/ -name "*.sol" -not -path "*/interfaces/*" -not -path "*/lib/*" -not -path "*/mocks/*" -not -path "*/test/*"` — enumerate in-scope contracts
e. Glob `**/references/z3-tractability.md` — resolve `$SKILL_DIR/references/` path
f. Bash `ls test/dinosat/ 2>/dev/null` — check if halmos tests already exist

**Gate 1 — Prerequisites.** If `halmos --version` fails, print: `Halmos is required. Install: pip install halmos`. Stop.

**Foundry profile setup.** Check `foundry.toml` for an existing `[profile.halmos]` section:

1. **No `[profile.halmos]` exists** — append the block below using profile name `halmos`. Set `$HALMOS_PROFILE = halmos`.
2. **`[profile.halmos]` exists and its `test` field is `test/dinosat`** — use it as-is. Set `$HALMOS_PROFILE = halmos`.
3. **`[profile.halmos]` exists but its `test` field points elsewhere** (e.g., `test/halmos`) — do NOT modify it. Instead, append a new `[profile.dinosat]` block. Set `$HALMOS_PROFILE = dinosat`. Inform the user: `Existing [profile.halmos] points to <path> — created [profile.dinosat] to avoid conflict.`

Block to append (use `$HALMOS_PROFILE` as the profile name):

```toml
[profile.$HALMOS_PROFILE]
src = "src"
test = "test/dinosat"
out = "out-halmos"
libs = ["lib"]
optimizer = false
evm_version = "shanghai"
ast = true
```

All subsequent commands that reference `FOUNDRY_PROFILE=halmos` MUST use `FOUNDRY_PROFILE=$HALMOS_PROFILE` instead. This includes Phase 3d (compilation), Phase 4a (Halmos execution), and any test file doc comments.

Create `test/dinosat/` directory if it does not exist.

### Phase 2 — Source Analysis & Pattern Detection

**Phase 2a — Parallel reads.** Make these parallel tool calls in one message:

a. Read ALL in-scope `.sol` files from Phase 1d (batch into multiple Read calls)
b. Read `$SKILL_DIR/references/z3-tractability.md`
c. Read `$SKILL_DIR/references/proof-techniques.md`
d. Read `$SKILL_DIR/references/property-catalog.md`
e. Read `$SKILL_DIR/references/test-templates.md`

**Phase 2b — Pattern scanner.** After all reads complete, make these parallel Grep tool calls in one message to detect Z3-hostile patterns across ALL source files. Use the Grep tool (NOT bash grep) with ripgrep syntax:

1. Detect Math.mulDiv calls — pattern: `Math\.mulDiv|mulDiv\(`, path: `src/`, glob: `*.sol`
2. Detect Math.sqrt calls — pattern: `Math\.sqrt|sqrt\(`, path: `src/`, glob: `*.sol`
3. Detect assembly blocks — pattern: `assembly\s*\{`, path: `src/`, glob: `*.sol`
4. Detect symbolic exponentiation patterns — pattern: `\*\*\s*[a-z]`, path: `src/`, glob: `*.sol`
5. Count if/else branch depth per function — pattern: `\} else if|\} else \{`, path: `src/`, glob: `*.sol`

All Grep calls should use `output_mode: "content"` with line numbers enabled (`-n: true`).

**Phase 2c — Protocol-type detection.** Classify the protocol archetype(s) using `$SKILL_DIR/references/protocol-types.md`. Scan for detection signals (e.g., `swap` + `addLiquidity` → AMM, `borrow` + `liquidate` → Lending). A protocol may be hybrid — apply properties from ALL matching archetypes.

Print to user: `Protocol type: <type> [+ <type2> if hybrid]`

**Phase 2c.1 — Threat model.** Using the detected protocol type(s), source code analysis, and `$SKILL_DIR/references/protocol-types.md`, build a threat model for the report. This goes into the "Threat Model" section of `DINOSAT_REPORT.md`. Identify:

1. **Assets at risk** — what value can be lost or stolen (deposits, debt accounting, fee splits, share value, protocol solvency). Derive from actual contract storage and token flows.
2. **Actors** — who interacts with the protocol and their capabilities. Include only roles that exist in the code (e.g., skip "Borrower" for a pure AMM). Note which actors are adversarial vs trusted.
3. **Trust assumptions** — what is assumed safe and out of scope. Derive from: constructor `require()` checks (parameter bounds), `onlyOwner` modifiers (admin trust), external calls to oracles/tokens (assumed correct).
4. **Attack vectors** — for each detected protocol type, list the primary attack surfaces from `protocol-types.md` and map them to the property IDs that mitigate them. This creates traceability from threat → property → proof.

Store this internally for use in Phase 5 (report generation). Do NOT print the full threat model to the user at this stage — it will appear in the report.

**Phase 2d — Contract inventory.** For each contract (internal reasoning):
- List all `public`/`external` functions with their modifiers
- Identify imported libraries and which functions call them
- Find constants library (PRECISION, thresholds, fee limits)
- Note Solidity version from `pragma`

**Phase 2e — Slow-path classification.** Using the grep results from 2b, build a slow-path map. For EACH function that contains a Z3-hostile pattern, record:

| Function | Pattern | Impact | Strategy |
|---|---|---|---|
| `calculateSwapFee` | Math.mulDiv x3, 3 branches | INTRACTABLE as-is | DECOMPOSE into per-branch lemmas + INLINE mulDiv |
| `calculateLiquidityMint` | Math.mulDiv x2, Math.sqrt | Math.sqrt INTRACTABLE | FUZZ for sqrt path, INLINE for mulDiv paths |
| `updateBaseRate` | Math.mulDiv x1 | Product fits uint256 | INLINE only |
| `calculateValueRatio` | Math.mulDiv x1 | Product fits uint256 | INLINE only |
| `_scaleDown` | `10 ** (18 - decimals)` | Symbolic exponent | ENUMERATE per decimal |

**Gate 2 — Slow-path mandatory.** Every function with a detected Z3-hostile pattern MUST have a strategy assigned. Do NOT generate a naive direct test for these functions — it will waste time on guaranteed timeouts.

**Phase 2f — Property extraction.** For each function, identify security properties from `property-catalog.md`. Assign priority: P0 (access control, arithmetic safety), P1 (fee bounds, liquidity invariants, swap conservation), P2 (oracle, incentives, scaling, state machine).

**Phase 2g — Tractability classification.** For each property, assign classification using `z3-tractability.md`. The slow-path map from 2e OVERRIDES default classification:

| Classification | Meaning | Test Strategy |
|---|---|---|
| DIRECT | Z3 proves as-is | Standard symbolic test |
| INLINE | Has Math.mulDiv but product fits uint256 | Replace with `(a*b)/c` + overflow guard |
| DECOMPOSE | Multi-branch (3+) with arithmetic | Per-branch lemma files |
| ENUMERATE | Small finite domain (decimals, pool counts) | One test per concrete value |
| BOUNDARY | Multi-variable nonlinear constraint | Fix worst-case param, prove at boundary |
| CROSSMUL | Ratio comparison with symbolic denominator | Rewrite as multiplication inequality |
| FUZZ | Intractable (Math.sqrt, symbolic x symbolic, nested mulDiv) | Fuzz-verify via Forge |

**Rule:** If a function is in the slow-path map (2e), use the assigned strategy. If a function has NO detected patterns, classify as DIRECT. If the function calls another function that is in the slow-path map, the caller inherits the callee's classification.

Use `$SKILL_DIR/references/decision-flowchart.md` for the master classification flow. Every property must go through this flow before test generation.

**Phase 2h — Proof obligation matrix.** Produce a complete matrix (print to user). Use ONLY plain markdown tables (`| col |` with `|---|` separator rows). NEVER use Unicode box-drawing characters (`╔═╗║╚╝╬╠╣╦╩` or similar) — they misalign in terminals when cell content varies in width:

```
PROOF OBLIGATION MATRIX

| Function       | Property | Class.     | Technique | Test File    |
|----------------|----------|------------|-----------|--------------|
| swap()         | FB-1     | DECOMPOSE  | T4        | FeeLemmas    |
| swap()         | SC-1     | INLINE     | T10       | SwapSymbolic |
| addLiquidity() | LI-1     | CROSSMUL   | T8        | LiqLemmas    |
| constructor    | AC-1     | DIRECT     | T3        | ACSymbolic   |
| ...            | ...      | ...        | ...       | ...          |

Summary: N properties total
  DIRECT: X | INLINE: X | DECOMPOSE: X | ENUMERATE: X
  BOUNDARY: X | CROSSMUL: X | FUZZ: X
Estimated: ~X formally proven, ~X fuzz-verified
```

Wait for user confirmation before proceeding to Phase 3.

### Phase 3 — Test Generation

**Phase 3a — Abstract specification helpers.** For contracts that contain Math.mulDiv (detected in Phase 2b), generate private helper functions that replicate the contract's core logic using inline arithmetic:

```solidity
/// @dev Abstract spec of PoolMath.calculateValueRatio using inline arithmetic.
///      Equivalent to Math.mulDiv(resA, baseRate, P) when product fits uint256.
///      Valid for: resA <= 1e24, baseRate <= 1e24 (product <= 1e48 < uint256.max)
function _valueRatio(uint256 resA, uint256 resB, uint256 baseRate) private pure returns (uint256) {
    uint256 valueA = (resA * baseRate) / P;
    uint256 totalValue = valueA + resB;
    if (totalValue == 0) return 0;
    return (valueA * P) / totalValue;
}
```

For EACH Math.mulDiv call site found in Phase 2b, produce an abstract helper that:
1. Documents the overflow constraint (`resA <= X, baseRate <= Y → product <= Z < uint256.max`)
2. Uses inline `(a * b) / c` instead of `Math.mulDiv(a, b, c)`
3. Matches the contract's exact computation logic (same variable names, same order of operations)

**Phase 3b — Test file generation.** Generate test files using the proof obligation matrix (Phase 2g) and templates from `test-templates.md`. Follow these non-negotiable rules.

**File naming:**
- `<Contract>Symbolic.t.sol` — direct symbolic and inline tests
- `<Feature>Lemmas.t.sol` — decomposed lemma proofs for complex properties

**Function naming:**
- ALL functions MUST start with `test_check_` (Halmos discovery convention)
- Lemmas: `test_check_lemma_<name>`
- Theorems: `test_check_theorem_<name>`
- Concrete enumeration: `test_check_<name>_<value>` (e.g., `_6dec`, `_2pools`)

**Assertions:**
- Use `assert()` ONLY. Halmos does not recognize `require()`, `assertTrue()`, or Forge assertions.
- Use `vm.assume()` for input constraints — these become SMT preconditions.

**Visibility:**
- `public pure` — tests using inline arithmetic, constants, or abstract spec helpers
- `public view` — tests calling library functions containing assembly (e.g., `Math.mulDiv`)
- **Rule:** If you used an abstract spec helper (Phase 3a) instead of the real library call, the test MUST be `public pure`, not `public view`.

**Input bounds (mandatory — must be justified):**

Default ranges (override with protocol-specific values when available):
- Reserves: `vm.assume(x >= 1e18 && x <= 1e24)`
- Amounts: `vm.assume(x >= 1 && x <= 1e27)`
- Rates/prices: `vm.assume(x >= 1e12 && x <= 1e24)`
- Fees: use project's `MIN`/`MAX` constants
- NEVER allow `0` to `type(uint256).max` — makes Z3 intractable

**Bounds justification (mandatory per test):** Every `vm.assume` bound MUST have a justification comment:
```solidity
// Justification: max token supply is 1e27 (1 billion tokens with 18 decimals)
vm.assume(resA >= 1e18 && resA <= 1e27);
// Justification: BASE_FEE_MIN and BASE_FEE_MAX from Constants.sol
vm.assume(baseFee >= Constants.BASE_FEE_MIN && baseFee <= Constants.BASE_FEE_MAX);
// Justification: rate bounded by constructor validation [1e10, 1e26]
vm.assume(baseRate >= 1e10 && baseRate <= 1e26);
```

Derive bounds from: protocol constants, constructor validations, token max supply, realistic price ranges, or explicit `require()` guards in the source code. If no source-derived bound exists, use the default ranges above and document "default range — no protocol-specific bound found."

**Constants:**
- Import and use the project's constants library
- Define `uint256 constant P = <PRECISION>;` at contract top

**Documentation (mandatory per test):**
- `@notice` — what property is proved (reference property ID from catalog)
- `@dev Technique:` — which proof technique (reference from proof-techniques.md)
- `@dev Classification:` — DIRECT/INLINE/DECOMPOSE/ENUMERATE/BOUNDARY/CROSSMUL/FUZZ
- Section separators: `// ════════════════════════════════════════════════`

**Boilerplate:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";
```

**Test generation order:**
1. Generate DECOMPOSE files first (lemmas — they are the foundation)
2. Generate DIRECT/INLINE/CROSSMUL/BOUNDARY/ENUMERATE files second (they may reference lemmas conceptually)
3. Generate FUZZ tests last (they are the fallback)
4. Generate fuzz wrapper functions (see Phase 3e below) in the same test files

Write all test files in one message (parallel Write calls).

**Phase 3c — Coverage gap analysis.** After generating all tests, verify every public/external function has at least one associated test. Compare the proof obligation matrix (Phase 2g) against the contract inventory (Phase 2c):

```
COVERAGE GAP ANALYSIS

| Function           | Tests | Gap                  |
|--------------------|-------|----------------------|
| swap()             | 5     | --                   |
| addLiquidity()     | 3     | --                   |
| emergencyRescue()  | 0     | MISSING: AC-1, AS-3  |
| setFee()           | 0     | MISSING: AC-1        |
```

**Gate 3a — No P0 gaps.** If any P0 property (access control, arithmetic safety) has zero coverage, generate the missing tests before proceeding. P1/P2 gaps are acceptable but MUST be documented in the report.

**Phase 3d — Compilation gate.** After writing all test files:

```bash
FOUNDRY_PROFILE=$HALMOS_PROFILE forge build --force 2>&1
```

**Gate 3 — Must compile.** If compilation fails, fix all errors before proceeding to Phase 4. Common fixes:
- `public pure` vs `public view` mismatch (assembly in called function requires `view`)
- Missing imports
- Pragma version mismatch (match project's Solidity version)
- `vm.assume()` in non-test function (only valid in `test_check_` functions)

Re-run `forge build` after each fix. Do NOT proceed to Phase 4 until compilation succeeds with zero errors.

**Phase 3e — Fuzz wrapper generation.** Halmos tests use tight `vm.assume` bounds (e.g., `vm.assume(x >= 1e18 && x <= 1e21)`) to keep Z3 tractable. When Forge fuzzes these same functions, random `uint256` values almost never satisfy the constraints — the rejection rate approaches 100% for tests with 3+ bounded parameters, and Forge will abort after exceeding `max_test_rejects`.

For EVERY `test_check_` function that has 2 or more `vm.assume` bounds on `uint256` parameters, generate a corresponding `test_fuzz_` wrapper in the same test file. The wrapper uses Forge's `bound()` helper to map random inputs into valid ranges instead of rejecting them:

```solidity
/// @notice Fuzz wrapper for test_check_<name> — uses bound() for input coverage
/// @dev See T17 in test-templates.md
function test_fuzz_<name>(uint256 rawA, uint256 rawB, uint256 rawC) public {
    uint256 a = bound(rawA, MIN_A, MAX_A);
    uint256 b = bound(rawB, MIN_B, MAX_B);
    uint256 c = bound(rawC, MIN_C, MAX_C);
    // ... same logic and assertions as test_check_<name> ...
}
```

**Rules for fuzz wrappers:**
- Mirror the `test_check_` function's logic and assertions exactly
- Replace every `vm.assume(x >= LO && x <= HI)` with `x = bound(rawX, LO, HI)`
- Omit any `vm.assume` that constrains relationships between parameters (e.g., `vm.assume(b <= a)`) — instead use `b = bound(rawB, MIN_B, a)` or a conditional return
- For relationship constraints that cannot be expressed with `bound()`, use an early `return` (not `vm.assume`) to skip unsatisfiable combinations — Forge counts these as passes, not rejections
- Naming: `test_fuzz_` prefix (Halmos ignores these, Forge discovers them)
- Visibility: match the `test_check_` counterpart (`public pure` or `public view`)

### Phase 4 — Execution

**Phase 4a — Progressive execution.** Run contracts in this order (fastest first, heaviest last):

1. **Lemma contracts** (`*Lemmas.t.sol`) — typically finish in seconds
2. **Light symbolic contracts** (contracts with only DIRECT tests) — seconds to minutes
3. **Medium contracts** (contracts with INLINE/CROSSMUL tests) — minutes
4. **Heavy contracts** (contracts with DECOMPOSE/BOUNDARY tests) — minutes to hours

For each contract:

```bash
FOUNDRY_PROFILE=$HALMOS_PROFILE halmos --contract <Name>Test \
  --function "test_check_" \
  --forge-build-out out-halmos \
  --solver-timeout-assertion 30000
```

**Flags:**
- Default timeout: `--solver-timeout-assertion 30000` (30 seconds per assertion)
- Loop unrolling: add `--loop 5` when test contains `for` loops (sorting, iteration)
- Single test debug: `--function "test_check_specificName"`
- Extended timeout: `--solver-timeout-assertion 60000` for complex proofs (try once before declaring FUZZ)

**Progress tracking.** Before starting Phase 4, compute total test count from the proof obligation matrix (Phase 2h). Track completed tests across all contracts. After each contract finishes, print:

```
DinoSAT in progress... [████████░░░░░░░░░░░░] 42% (38/90 tests)
  ✓ FeeLemmasTest: 8 passed, 0 timeout | 12s
  ✓ SwapLemmasTest: 9 passed, 0 timeout | 1s
  ▸ PoolSymbolicTest: running...
```

Update the progress bar after each contract completes:
- `completed_tests / total_tests * 100` = percentage
- Filled blocks: `█` for done, `░` for remaining (20-character bar)
- `✓` for completed contracts, `▸` for currently running, `○` for pending

On the FIRST line of output (before any contract runs), print:
```
DinoSAT in progress... [░░░░░░░░░░░░░░░░░░░░] 0% (0/<TOTAL> tests)
```

**Phase 4b — Interpret results.**

| Result | Meaning | Action |
|---|---|---|
| `[PASS]` | Formally proven — Z3 exhausted all paths, no counterexample | Record as PROVEN |
| `[TIMEOUT]` | Solver timed out | See timeout protocol below |
| `[FAIL]` + counterexample | Potential bug — Z3 found a concrete violation | See counterexample protocol below |
| `[ERROR]` | Unsupported cheat code or compilation issue | See error protocol below |

**Counterexample protocol.** When Z3 produces a `[FAIL]` with concrete values:

1. **Extract values.** Record every symbolic variable's concrete assignment from the Halmos output.
2. **Verify manually.** Substitute the concrete values into the test function by hand. Confirm the assertion actually fails with those values. If it does not fail, the counterexample is spurious (Z3 artifact) — retry with tighter bounds.
3. **Trace exploit path.** If the counterexample is real, trace through the contract logic: which function is called, what state changes occur, what value is extracted or lost.
4. **Classify severity:**
   - **CRITICAL**: Direct fund loss, unauthorized access, infinite mint
   - **HIGH**: Bounded fund loss, economic manipulation, DoS
   - **MEDIUM**: Governance bypass, incorrect accounting with limited impact
   - **LOW**: Rounding unfavorable to protocol, dust-level value leak
5. **Report immediately.** Print counterexample values, affected function, severity, and exploit path to user BEFORE continuing with remaining tests.

**Error protocol.** When Halmos produces `[ERROR]`:

Common causes and fixes:

| Error Pattern | Cause | Fix |
|---|---|---|
| `Unsupported cheat code: 0xc31eb0e0` | `vm.label()` | Remove `vm.label` calls |
| `Unsupported cheat code` (general) | Unsupported `vm.*` function | See cheat code table below |
| `all paths have been reverted` | `vm.assume` constraints too tight | Widen bounds or check for contradictory assumptions |
| `paths not fully explored due to loop` | Loop exceeds `--loop` bound | Increase `--loop N` |

**Halmos cheat code compatibility** (Halmos v0.1.13):

| Cheat Code | Supported | Notes |
|---|---|---|
| `vm.assume(bool)` | YES | Core — defines SMT preconditions |
| `vm.prank(address)` | YES | Sets `msg.sender` for next call |
| `vm.startPrank(address)` | YES | Persistent `msg.sender` override |
| `vm.stopPrank()` | YES | Clears prank |
| `vm.deal(address, uint256)` | YES | Sets ETH balance |
| `vm.warp(uint256)` | YES | Sets `block.timestamp` |
| `vm.roll(uint256)` | YES | Sets `block.number` |
| `vm.store(address, bytes32, bytes32)` | YES | Direct storage write |
| `vm.load(address, bytes32)` | YES | Direct storage read |
| `vm.expectRevert()` | NO | Use assert-based alternatives |
| `vm.expectEmit()` | NO | Cannot verify events |
| `vm.label(address, string)` | NO | Remove from test code |
| `vm.toString()` | NO | Remove from test code |
| `vm.getNonce()` | NO | Use concrete values |
| `vm.computeCreateAddress()` | NO | Use concrete addresses |

**Rule:** Only use supported cheat codes in Halmos tests. If a property requires an unsupported cheat code, restructure the test to use assert-based logic instead.

**Phase 4c — Vacuous test detection.** After each contract's Halmos run, check for vacuous proofs using the criteria in `$SKILL_DIR/references/decision-flowchart.md` (Vacuous Test Detection section):

1. Any test with warning "all paths have been reverted" → contradictory `vm.assume` — fix constraints
2. Any multi-parameter test with `paths: 0` or `paths: 1` → constraints may be too tight — investigate
3. Any assertion that is a tautology (`assert(x >= 0)` on uint256) or re-states an assumption → strengthen
4. Any test where the computation between `vm.assume` and `assert` is trivial or missing → add real logic

**Gate 4 — No vacuous P0 proofs.** If a P0 property test is vacuous, it MUST be fixed before proceeding. P1/P2 vacuous tests should be documented but do not block.

**Phase 4d — Timeout protocol.** When a test times out, check if it was already classified as FUZZ in the proof obligation matrix. If yes, record as FUZZ and move on. If it was classified as DIRECT/INLINE/CROSSMUL/BOUNDARY/ENUMERATE but still timed out:

1. Check if the test accidentally calls `Math.mulDiv` instead of using the abstract spec helper → fix and retry
2. Check if input bounds are too wide (e.g., `1e27` can be tightened to `1e24`) → tighten and retry
3. Check if a cross-multiplication can eliminate a remaining division → rewrite and retry
4. If none of the above work, reclassify as FUZZ

**Maximum retries per test: 3.** After 3 failed attempts, mark as FUZZ unconditionally.

**Phase 4e — Fuzz verification.** After all Halmos runs complete, run Forge fuzz tests. Execute two fuzz passes per contract:

**Pass 1 — Fuzz wrappers** (primary fuzz coverage). Run the `test_fuzz_` functions generated in Phase 3e. These use `bound()` for near-zero rejection and provide real input coverage:

```bash
forge test --match-contract <Name>Test --match-test "test_fuzz_" --fuzz-runs 1000000 -v
```

**Pass 2 — Halmos tests under fuzz** (regression). Run the `test_check_` functions under Forge fuzz as a sanity check. These will have high rejection rates due to tight `vm.assume` bounds — this is expected. The primary coverage comes from Pass 1:

```bash
forge test --match-contract <Name>Test --match-test "test_check_" --fuzz-runs 100000 -v
```

**Rules:**
- `--fuzz-runs 1000000` on Pass 1 is mandatory — Forge defaults to 256 runs, which is insufficient for security claims.
- Pass 2 uses 100,000 runs (not 1M) because high rejection makes more runs wasteful. If Forge aborts Pass 2 due to exceeding `max_test_rejects`, record this as "high rejection — covered by fuzz wrapper" and continue. This is not a failure.
- Record run count and failure count for both passes.
- A `test_fuzz_` failure is a real finding — investigate immediately (same counterexample protocol as Phase 4b).

### Phase 5 — Report

Generate `DINOSAT_REPORT.md` at the project root. Follow the exact template in `$SKILL_DIR/references/report-formatting.md`. All sections are mandatory. See `$SKILL_DIR/references/worked-example.md` for a concrete example of how the report should look.

**Hard gates for report:**
- If ANY test produced `[FAIL]` with a counterexample, it MUST be the FIRST item in the report, prominently flagged as a discovered vulnerability.
- Every `[PASS]` in the report MUST correspond to an actual Halmos run. No fabrication.
- Every `[FUZZ]` MUST correspond to an actual Forge run with zero failures.
- The Disclaimer section MUST appear in BOTH the markdown report AND the PDF. It is not optional.
- The **Verification Status Legend** section MUST appear before Results Summary.
- The **Threat Model** section (from Phase 2c.1) MUST appear between the Status Legend and Results Summary. All four subsections are mandatory: Assets at Risk, Actors, Trust Assumptions, Attack Vectors Considered.
- The **Protocol** and **Date** fields MUST appear in the report metadata header.

Also generate `dinosat-log.txt` with raw per-test results (PASS/FUZZ lines with paths and timing).

### Phase 6 — PDF Generation

Generate a branded PDF from the markdown report.

**Phase 6a — Check dependencies.**

```bash
pip install fpdf2 2>/dev/null || pip3 install fpdf2 2>/dev/null
```

If install fails, skip PDF generation and inform the user:
```
PDF generation skipped — fpdf2 not available.
Install: pip install fpdf2
The markdown report is available at: DINOSAT_REPORT.md
```

`fpdf2` is a pure Python library with zero system dependencies — no Pango, GLib, or LaTeX needed.

**Phase 6b — Generate PDF.**

```bash
python3 $SKILL_DIR/references/generate_pdf.py DINOSAT_REPORT.md DINOSAT_REPORT.pdf
```

The script:
1. Reads the markdown report
2. Adds DinoSAT cover page with logo (prefers `logo.png` over `logo.svg` — PNG renders correctly for complex paths)
3. Generates a Table of Contents page from H2/H3 headings
4. Parses and renders tables, code blocks, headers, lists
5. Color-codes results: green for PROVEN, blue for FUZZ, red for COUNTEREXAMPLE
6. Proportional column widths (no text truncation)
7. Outputs a paginated A4 PDF with page numbers and "Generated by DinoSAT" footer

**Unicode limitation:** The PDF uses Helvetica (a core PDF font limited to Latin-1 encoding). Non-ASCII characters (em-dashes, smart quotes, math symbols) are auto-converted to ASCII equivalents. Characters outside Latin-1 with no mapping are replaced with `?`. When writing `DINOSAT_REPORT.md`, use only ASCII characters in text content — e.g., `--` instead of `—`, `<=` instead of `≤`, straight quotes instead of smart quotes. Code blocks and Solidity identifiers are already ASCII-safe.

All output files go to the **project root directory** (same level as `foundry.toml`). Print to user:

```
DinoSAT complete.

Output files (project root):
  Report (markdown): ./DINOSAT_REPORT.md
  Report (PDF):      ./DINOSAT_REPORT.pdf
  Raw log:           ./dinosat-log.txt
  Tests:             ./test/dinosat/
```

---

## Constraints

- No fabrication. Every claim must correspond to an actual tool execution.
- Single pass per property. Do not re-run tests that already passed.
- Always run contracts sequentially (one at a time) to avoid memory pressure on Z3.
- Never use `--solver-timeout-assertion` below `10000` (10s) — too aggressive.
- If a contract has >30 tests, split into logical groups and run each group separately.
- Print progress to user after each contract completes.
- `optimizer = false` in the halmos profile is NON-NEGOTIABLE — optimizer confuses Z3.
- `ast = true` in the halmos profile is REQUIRED — Halmos cannot parse artifacts without it.
- Maximum 3 retry attempts per timed-out test (see decision-flowchart.md). After 3 retries, mark FUZZ.
- Never generate a direct test for a function in the slow-path map — always use the assigned strategy.
- Vacuous proofs (tautological assertions, contradictory assumptions) must be detected and fixed.
- ALL tables printed to the user MUST use plain markdown format (`| col |` rows with `|---|` separators). NEVER use Unicode box-drawing characters (`╔═╗║╚╝╬╠╣╦╩`) — they break alignment when cell content varies in width.

---

## Reference Files

| File | Purpose |
|---|---|
| `references/z3-tractability.md` | 7 classifications, safe product table, solver configuration |
| `references/proof-techniques.md` | 8 techniques with soundness arguments and code patterns |
| `references/property-catalog.md` | 10 categories + cross-contract, 47+ property IDs |
| `references/protocol-types.md` | 8 protocol archetypes with targeted property selection |
| `references/test-templates.md` | 15 code templates mapped to property IDs |
| `references/decision-flowchart.md` | Master classification flow, technique selection, timeout escalation, vacuous detection |
| `references/report-formatting.md` | Exact report template with formatting rules |
| `references/worked-example.md` | End-to-end example: source → tests → Halmos output → report |
