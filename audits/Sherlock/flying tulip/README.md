# Flying Tulip (ftPUT) - Protocol Overview

> **Disclosure:** This write-up is published publicly with the permission of the Flying Tulip team, for research and educational purposes.

> Sherlock Bug Bounty #248, https://audits.sherlock.xyz/bug-bounties/248
> Reviewed: on-chain verified sources (Ethereum mainnet), Review date 2026-07-09

---

## 1. Program Info

| Item | Value |
|---|---|
| Status | LIVE (started 2026-06-18) |
| Max reward | 1,000,000 USDC |
| Critical | 100,000 → 1,000,000 (10% of directly affected funds, up to 1M) |
| High | 50,000 → 100,000 |
| Medium | 1,000 → 5,000 |
| Required | Runnable coded PoC |
| Exploit window | If the team can respond (pause), damage is capped to a **1-hour** window |
| "funds affected" | Based on the Circuit Breaker's max withdrawal limit. **If a bug bypasses the CB = full TVL** |

**Trust model:** the msig admin is fully trusted (including implementation upgrades). Other roles (strategyManager, yieldClaimer, keeper, etc.) are partially trusted. Even if a privileged role can harm the protocol/users, it is treated as **informational** → **attack scenarios are only valid from the perspective of an unprivileged external attacker.**

---

## 2. Product Overview

Flying Tulip's **ftPUT** is an on-chain put-option / collateral product.

```
User deposits collateral (USDC/USDT/WETH/USDS/USDtb/USDe)
        │  PutManager.invest()
        ▼
   pFT NFT minted (PUT position)   ← stores {ft, amountRemaining, strike, ftPerUSD}
        │
        ├── divest()          → exercise the put: reclaim collateral (capped by amountRemaining)
        ├── divestUnderlying() → reclaim collateral in-kind as positionToken (aToken/vault share)
        └── withdrawFT()       → (after transferable) receive FT tokens; collateral reclaimed by msig

Collateral deposited into ftYieldWrapper (1:1 share)
        │  keeper deploys
        ▼
   Deployed to a Strategy → yield from Spark ERC4626 / Aave
        ▲
   CircuitBreaker rate-limits wrapper withdrawals (5% / 6h)
```

- **Nature of the PUT:** deposit collateral, receive an FT claim + a put (right to sell back at strike). Each position's reclaimable collateral is capped by `amountRemaining` (the deposit) → cannot reclaim more than deposited even with oracle manipulation.
- **The pFT NFT is tradable after `transferable`** → bought/sold on the pFTMarketplace.

---

## 3. In-scope Contracts (Ethereum, 17 addresses)

| Contract | Address | Type |
|---|---|---|
| FT (OFT token) | 0x5DD1A7A369e8273371d2DBf9d83356057088082c | LayerZero OFT (same address on Avax/Base/Sonic/BNB) |
| PutManager (impl) | 0x90AE2Cac15F8d58A258f7B4a243657754469922a | UUPS |
| pFT (impl) | 0xc55253Ea84050700E1EfA8878D4A5053b6Bf7c5E | UUPS ERC721 |
| pFTMarketplace (impl) | 0x2a35f9f1B4Ab24F377a06edA61BDa382F7b2Da7F | UUPS |
| YieldClaimer | 0x88432bB6EA62e774cB6d87995CC5277568d01397 | owner/keeper |
| FlyingTulipOracle | 0xC8C895E2be9511006287Ce02E51B5B198AB36793 | Aave price + bounds |
| CircuitBreaker | 0xCb170bc873b3a1F69F433C25a4b6d0fd4D4D90De | rate limiter |
| ftACL | 0xA09d08E5A850B26d39Ea2a69f8f99Fd8AA1359EB | Merkle whitelist |
| ftYieldWrapper ×6 | (per token, see table below) | collateral vault |
| ERC1967Proxy ×3 | 0xa421…04F2, 0xbA49…ebaA, 0x3124…570c | proxies |

### Live production config (on-chain query 2026-07-09)

**CircuitBreaker:** maxDrawRate = 5% (5e16 wad), mainWindow = 6h, elasticWindow = 2h, not paused, all 6 wrappers registered as protectedContracts.

| Pool | wrapper | TVL | CB wired | Strategy | vault |
|---|---|---|---|---|---|
| USDC | 0x095d…bf59 | ~15.75M | ✅ | SparkSavingsStrategy | spUSDC ~$307M |
| WETH | 0x9d96…E305 | ~4,814 | ✅ | SparkSavingsStrategy | spETH ~$170M+ |
| USDT | 0x267d…cB36 | ~18.02M | ✅ | SparkSavingsStrategy | spUSDT ~$455M |
| USDS | 0xA143…7573 | ~2,376 | ❌ (0x0) | AaveStrategy | Aave |
| USDtb | 0xE527…97b6 | ~110 | ❌ (0x0) | AaveStrategy | Aave |
| USDe | 0xe688…5625 | ~71,944 | ❌ (0x0) | AaveStrategy | Aave |

> ⚠️ **Scope note:** `SparkSavingsStrategy`, which actually manages most of the funds (the 3 large pools), is **not** in the Sherlock scope file tree (only `AaveStrategy.sol` is listed for strategies). (→ findings FT-04)
> All 3 Spark vaults use **internal accounting** (`totalAssets ≫ own token balance`) → share price cannot be manipulated via donation.

---

## 4. Chains & Tokens

- **Chains:** Ethereum, Base, BSC, Avalanche, Sonic. **However, the ftPUT system (PutManager/wrapper/strategies) is Ethereum-only**; other chains only have the FT OFT token.
- **Tokens:** wrapped native (wETH/wBNB, etc.), USDC, USDT, USDS, USDTB, USDE (the per-network `deployments.toml` is the judging reference).
- **FT token:** LayerZero OFT, 10B initial supply minted only on Sonic (chainid 146), cross-chain mint/burn.

---

## 5. Known Issues (out of scope)

github.com/flyingtulipdotcom/security, `KNOWN_ISSUES.md` - the following are already-accepted issues, hence **duplicate/invalid**:

- **PM-01** rounding dust for low-decimal tokens at small amounts, **PM-02** protocol-level losses not handled on-chain, **PM-03** missing slippage in `invest()`
- **SM-01~04** timelock, strategyManager, cap front-run, removing a non-empty strategy
- **AAVE-01** `availableToWithdraw` doesn't check liquidity, **YC-01** yield claimer responsiveness
- **LEV-01** cancel/fill share the hash, **MKT-01~03** offer snapshot not bound, stale listings, native payout failure

---

*For the detailed analysis see [analysis.md](analysis.md), for findings/judgment see [findings.md](findings.md), and the reproduction harness is under `../fuzz/`.*
