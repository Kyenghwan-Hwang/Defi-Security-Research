# Flying Tulip (ftPUT) - Findings + Reasoning

> **Final verdict:** **No Critical / High / Medium vulnerability** allowing an unprivileged attacker to lose funds on current production.
> The 5 items below (FT-01~05) are all **informational** - either no loss is reachable (they presuppose a separate drain),
> they fall under trusted roles, or the real vaults neutralize them. **For team informational sharing, not for bounty submission.**

---

## Judgment framework

To reach Medium or above in this bounty: **an unprivileged attacker + not relying on trusted-role misbehavior + a loss of funds (or long-term freeze) in the actual deployed state + a working PoC.**
Each item is annotated with where it fails to clear this bar.

| ID | Title | Real loss PoC? | Reason not submittable | Rating |
|---|---|---|---|---|
| FT-01 | CB elastic buffer limit amplification (21x) | ❌ | Presupposes a separate drain + locked capital, not flash-loanable. elastic is intended per ERC-7265 | Info |
| FT-02 | CB fail-open + buffer rollback | ❌ | Trigger (amount>preTvl) not reachable via divest. fail-open is documented as intended in the interface | Info |
| FT-03 | 3 small pools with CB unset (`circuitBreaker=0`) | ❌ | strategyManager (trusted role) responsibility → informational per the rules | Info |
| FT-04 | SparkSavingsStrategy scope mismatch | ❌ | Not a bug (scope question) | Info |
| FT-05 | Spark deposit missing slippage/minShares (ERC4626 inflation) | ✅ (fresh vault) | Real vaults are mature + internal accounting → not reachable | Info |

---

## FT-01 - CircuitBreaker elastic buffer inflates the withdrawal limit 1:1 with deposits

**Target:** `CircuitBreaker.recordInflow` (`elasticBuffer += amount`), `ftYieldWrapper.deposit`
**Demonstrated:** `CBInvariant.t.sol::test_A_elasticInflatesRateLimit`

Effective CB limit = `5% of TVL + (deposits in the last 2h)`. Depositing a TVL-sized amount inflates the limit from 5% → 105% (**21x**, measured). Weakens the team's "1-hour response" assumption.

**Reasoning - why Info:** zero standalone loss. To consume elastic capacity for a drain the attacker must **lock equal collateral**, and recovering it needs capacity again → an in-tx `invest→drain→divest-back` is again rate-limited, so **flash-loans do not work**. It's an amplifier, not fund creation. + the elastic buffer is the intended ERC-7265 deposit-tracking behavior.
**Fix:** clamp the elastic contribution to `cap * K` in the `available` computation, or add an absolute per-window ceiling.

---

## FT-02 - CircuitBreaker fail-open + buffer rollback on underflow

**Target:** `ftYieldWrapper.withdraw:494 / withdrawUnderlying:595` (`} catch {}`), `CircuitBreaker.checkAndRecordOutflow:154` (`preTvl - amount`)
**Demonstrated:** `CBInvariant.t.sol::test_B_failOpen_bufferRollback` (buffer rollback confirmed), `test_B2_wrapperInputsAreSafe`

The wrapper swallows a CB revert with `catch{}` (fail-open). The CB decrements the buffers, then in `emit Outflow(…, preTvl - amount)` it underflows and reverts if `amount > preTvl` → the decrement is rolled back → the rate limit isn't updated. Repeating withdrawals in that band neuters the limiter.

**Reasoning - why Info:** since `amount ≤ collateralSupply ≈ preTvl`, the underflow branch is **not reachable via divest** (test_B2). Currently not triggerable. fail-open is also documented as intended in the interface comment ("designed for fail-open"). A latent defect only.
**Fix:** (1) remove the underflow with `postTvl = amount > preTvl ? 0 : preTvl - amount` (harmless, must-do). (2) make the rate-limit decision fail-closed.

---

## FT-03 - 3 production pools have no CircuitBreaker wired to the wrapper

**Target:** `ftYieldWrapper.setCircuitBreaker` (allows 0), live state
**Demonstrated:** on-chain query - USDS/USDtb/USDe wrappers have `circuitBreaker() == 0x0`

Those three pools have no rate limit at all. With a separate drain, full withdrawal is possible immediately with no CB throttling. Combined with FT-02's fail-open, the missing CB is also masked.

**Reasoning - why Info:** CB configuration is the strategyManager's (**fully-trusted role**) responsibility → informational per the bounty rules. Small pools (~$74k combined). No loss primitive.
**Recommendation:** the strategyManager should call `setCircuitBreaker(CB)` on the three wrappers.

---

## FT-04 - SparkSavingsStrategy is outside the audited file list (scope question)

**Target:** deployed strategies (the 3 large USDC/WETH/USDT pools, ~$49M)
**Demonstrated:** on-chain - strategy(0) is `SparkSavingsStrategy` (scope only lists `AaveStrategy.sol`)

Code managing most of the funds is not in the audited file list → coverage/judging risk. (Not a bug.)

**Recommendation:** ask the judges whether it's in scope. If included, focus on the ERC4626 integration (share rounding, liquidity, in-kind withdrawals).

---

## FT-05 - SparkSavingsStrategy.deposit has no slippage/minShares protection (ERC4626 inflation)

**Target:** `SparkSavingsStrategy.deposit` (`vault.deposit(amount, this)` - no validation of received shares)
**Demonstrated:** `SparkInflationPoC.t.sol` - fresh vault: deposit 1M → recover 990k (**10k loss**) / mature vault ($300M): 1 wei
+ the end-to-end fuzzer (`SparkInvariant`) autonomously discovered the `invest → donate → deployIdle` sequence, reproducing a ~8,961 USDC shortfall

Depositing a large amount while the vault share price is inflated causes a rounding loss borne by the protocol. The well-known ERC4626 donation/inflation surface.

**Reasoning - why Info (blocked two ways):**
1. **All 3 current vaults are mature** ($307M/$170M+/$455M) → inflating the share price needs hundreds of millions, economically infeasible.
2. **All use internal accounting** (`totalAssets ≫ own token balance`: spUSDC reports 307M vs holds 10M USDC) → **a token donation does not move the share price** → donation attack fundamentally blocked. (verified on-chain)
3. All 6 wrappers have 1 strategy each, no pending → no low-liquidity strategy either.

→ **Not reachable in the current state.** The risk only materializes if an immature/naive ERC4626 vault is onboarded in the future.
**Fix:** add a `convertToAssets(shares) + tolerance ≥ amount` guard right after depositing; verify sufficient TVL when onboarding a new vault.

---

## Final verdict & reasoning

**No submittable vulnerability.** The common disqualifier across all 5 items is the same - **the absence of a loss-of-funds path reachable by an unprivileged attacker in the actual deployed state.** Static review and 52 dynamic tests empirically demonstrate the robustness of every layer: core accounting, pricing math, CB, marketplace, OFT, strategies.

**Conditions that would flip the verdict:**
- FT-05 → a **low-liquidity/naive ERC4626 vault** exists as a strategy target in the private `deployments.toml` (which is stated to be referenced during judging).
- FT-01/02/03 → if a **separate, real drain primitive** is found, bundle them as a CB-bypass argument to escalate the rating to TVL (Critical).

**Recommendation:** rather than submitting the 5 items to the bounty (each risking a $250 deposit), **share them with the team as informational**. In particular FT-05 (strategy slippage guard) and FT-04 (scope clarification) are useful to the team.
