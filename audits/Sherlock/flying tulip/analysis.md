# Flying Tulip (ftPUT) - Analysis

> Methodology: manual review of on-chain verified sources (static) + Foundry invariant/directed fuzzing (dynamic)
> Result: **52 tests / 13 suites all pass.** No unprivileged-attacker fund-drain path found.

---

## 1. Methodology & Coverage

1. **Source acquisition** - extracted the 17 in-scope addresses + deployed strategies from Sourcify/Blockscout/Etherscan (the private repo `ftPUT` is not public).
2. **Static review** - full review of 9 core contracts + 2 strategies (4,559 + 405 lines).
3. **Live verification** - measured the production config with `cast` (CB parameters, wrapper↔strategy↔vault, TVL, vault accounting model).
4. **Dynamic fuzzing** - deployed the core system behind proxies, fuzzed invariants from an unprivileged-user perspective + directed PoCs.

### Coverage matrix

| Attack surface | Method | Harness | Result |
|---|---|---|---|
| Core accounting (PutManager/pFT/wrapper) | 6 invariants + 4 directed | `FTInvariant`, `FTUnit` | ✅ |
| Pricing conversion math (full param space) | pure-function, 150k runs | `PricingFuzz` | ✅ |
| CircuitBreaker | math invariant + bypass PoC | `CBInvariant` | ✅ |
| Marketplace | conservation invariants + signed offers | `MarketInvariant`, `MarketDirected` | ✅ |
| FT OFT (LayerZero) | pause, permit, burn, supply | `FTOFT` | ✅ |
| Strategy: Spark ERC4626 | end-to-end + inflation PoC | `SparkInvariant`, `SparkInflationPoC`, `SparkChar` | ✅ |
| Strategy: multi-strategy withdraw loop | 3 strategies, 30k calls | `MultiStrategy` | ✅ |

---

## 2. Key Analysis Per Contract

### PutManager
- `invest` → pFT.mint; `divest`/`divestUnderlying`/`withdrawFT` are the exit paths.
- **Key safeguard:** pFT caps with `amountDivested ≤ amountRemaining` and `amount ≤ ft`. `amountRemaining` starts at the deposit and only decreases → **cannot reclaim more than the per-position deposit, even with oracle manipulation.**
- `strike`/`ftPerUSD` are fixed at mint time → invest↔divest round trip is safe.
- All state-changing functions are `nonReentrant` (transient) + CEI. pFT.mint (receiver hook) is the last call after state updates.

### pFT (ERC721)
- Position struct: `{token, amount, ft, ft_bought, withdrawn, burned, strike, amountRemaining, ftPerUSD}`.
- `divest` (burned++) / `withdrawFT` (withdrawn++) both check the ft, amountRemaining ceilings before decrementing. When ft==0, burn + zero the amountRemaining dust.

### ftYieldWrapper
- 1:1 collateral↔share model (not proportional shares) → no first-depositor inflation.
- `withdraw`/`withdrawUnderlying` drain strategies in order. `deployed`/`deployedToStrategy` clamp accounting.
- **The CircuitBreaker is fail-open** (`try…catch{}`) - the intent is documented in the code comments.

### CircuitBreaker
- ERC-7265-based dual buffer: main (5% of TVL, time-replenishing) + elastic (deposit-tracking).
- Arithmetic is sound (30k fuzz calls never authorized beyond the budget). But elastic grows 1:1 with deposits (→ findings FT-01).

### FlyingTulipOracle
- Aave price + min/max bounds + fixed `ftPerUSD` (=10). divest uses the stored value → round trip is safe.

### pFTMarketplace
- Direct-listing sale (`buy`) + EIP-712 signed offers (`acceptBuyOffer`) + Permit2 payment.
- Fund-flow conservation: buyer = price+takerFee, seller = price−makerFee, fee = sum. The marketplace holds no funds/NFTs.

### FT (OFT)
- LayerZero OFT + pause `_update` gating (configurator/endpoint exceptions) + custom permit (ECDSA + ERC-1271).
- Cross-chain mint/burn is inherited from the LZ-audited base (custom surface = pause + permit only, both safe).

### Strategies (Spark ERC4626 / Aave)
- Both value-preserving. Aave = aToken 1:1. Spark = ERC4626 share conversion (rounds down, protocol-favorable).
- `claimYield` has a `valueOfCapital ≥ totalSupply` guard → never dips below principal.

---

## 3. Empirically-verified safety invariants (52 tests)

- **No profit:** total user withdrawals ≤ total deposits (even under oracle pump/dump, yield).
- **Solvency:** vault recoverable value ≥ collateralSupply (including multi-strategy, liquidity changes).
- **Global consistency:** `collateralSupply == wrapper.totalSupply`, `Σ position.ft == ftAllocated`, `FT.balanceOf(PutManager) == ftOfferingSupply`, `deployed == Σ deployedToStrategy`.
- **Backing:** `collateralSupply ≥ Σ amountRemaining + capitalDivesting` (the surplus is the known PM-01 dust, in the protocol-favorable direction).
- **Pricing math:** across the full param space, `collateralFromFT(ftFromCollateral(x)) ≤ x`, chunk-splitting yields no advantage, no FT inflation.
- **Marketplace:** fund/NFT conservation; offer cancellation, replay, signer, non-qualifying NFT all rejected.
- **CB:** never authorizes beyond the rate-limit budget.
- **FT OFT:** pause gating, permit, burn, supply conservation all correct.

---

## 4. Reproduction

```bash
cd ../fuzz
forge test                              # all (52 tests)
forge test --match-contract FTInvariant   # core accounting
forge test --match-contract CBInvariant   # CircuitBreaker + FT-01/02 PoC
forge test --match-contract SparkInflationPoC -vv   # FT-05 PoC
forge test --match-contract PricingFuzz --fuzz-runs 50000
```

- Dependencies: OpenZeppelin v5.3.0 (matches the deployed impls; v5.5.0 removed `__UUPSUpgradeable_init`), solc 0.8.30, evm cancun, via_ir.
- LayerZero sources are compiled from `../_libs` (`@layerzerolabs/=lib-lz/at_layerzerolabs/`).

### Environment gotchas (noted)
- **foundry:** an inline external call passed as an argument (`mkt.getCurrentPutHash(id)`) consumes the preceding `vm.prank`/`vm.expectRevert` → precompute into a local variable.
- **macOS case-insensitive FS:** `IftPut.sol` ↔ `IftPUT.sol` collide → the marketplace's one was split out to `IPutView.sol`.
- **harness gotcha:** if the handler doesn't implement `onERC721Received`, every invest silently reverts → the invariants pass trivially on an empty system. **Always verify the harness actually mutates state via activity counters + directed tests.**

---

*See [findings.md](findings.md) for the findings and reasoning.*
