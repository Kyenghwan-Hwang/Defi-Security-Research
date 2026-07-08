# Defi-Security-Research

![Solidity](https://img.shields.io/badge/-Solidity-363636?style=flat-square&logo=solidity&logoColor=white)
![Security Research](https://img.shields.io/badge/-Security%20Research-DC3545?style=flat-square)
![Foundry](https://img.shields.io/badge/-Foundry-000000?style=flat-square)
![Slither](https://img.shields.io/badge/-Slither-6f42c1?style=flat-square)

Personal DeFi / smart contract security research: protocol risk assessments, static analysis, and post-disclosure incident write-ups.

## Protocol Threat Reports
- **Morpho Blue** - Full protocol risk assessment covering admin control, governance, dependencies, oracle risk, and economic security, with quantitative Risk Radar scoring.

## Slither Analysis
- **Morpho Blue** - Slither static analysis walkthrough: detector findings, false positives, and manual review notes.

## DeFi Oneday Research
Post-disclosure ("1-day") incident write-ups with root-cause analysis and mainnet-fork PoC reproduction.
- **Kelp DAO (rsETH)** - LayerZero DVN compromise, ~$290M cross-chain bridge drain
- **Penpie** - $27M cross-contract reentrancy via a malicious Pendle market
- **Sonne Finance** - Empty-market donation attack on a Compound V2 fork, ~$20M
- **Stream Finance (xUSD)** - Recursive-looping insolvency and oracle-frozen bad debt, ~$93M
