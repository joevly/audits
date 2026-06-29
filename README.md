<div align="center">

<img src="logo.png" alt="Joev logo" width="140"/>

# Joev

### Smart Contract Security Researcher

Independent security researcher focused on the EVM ecosystem, reviewing Solidity
protocols and hunting high-impact bugs before attackers find them.

[![GitHub](https://img.shields.io/badge/GitHub-joevly-181717?style=flat-square&logo=github)](https://github.com/joevly)
&nbsp;
[![Twitter](https://img.shields.io/badge/X-@Joevly7-000000?style=flat-square&logo=x)](https://x.com/Joevly7)
&nbsp;
[![Discord](https://img.shields.io/badge/Discord-joevly-5865F2?style=flat-square&logo=discord)](https://discord.com/)

</div>

---

## About

I'm a smart contract security researcher specializing in **EVM / Solidity**. I review DeFi
protocols, AMMs, lending, flash loans, token standards, and proxy patterns. Pairing careful
**manual review** with **fuzzing**, **invariant testing**, and **static analysis** to surface real, high-impact vulnerabilities.

This repository is my smart contract security audit portfolio. Every report documents the
**scope**, **methodology**, and each finding with a clear **description**, **impact**,
**proof of concept**, and **recommended fix**.

## Skills & Tools

![Solidity](https://img.shields.io/badge/Solidity-363636?style=flat-square&logo=solidity&logoColor=white)
![Foundry](https://img.shields.io/badge/Foundry-FE5d26?style=flat-square)
![Slither](https://img.shields.io/badge/Slither-4B275F?style=flat-square)
![Aderyn](https://img.shields.io/badge/Aderyn-1040A4?style=flat-square)

- **Languages:** Solidity · learning Vyper / Yul
- **Testing:** Foundry · `forge` unit tests, fuzzing, invariant / handler-based testing
- **Static analysis:** Slither, Aderyn
- **Approach:** line-by-line manual review + reproducible PoC exploit development
- **Bug classes:** access control · reentrancy · weak randomness · AMM invariants & slippage
  (`x * y = k`) · oracle / price manipulation · flash-loan abuse · ERC-20 edge cases ·
  UUPS / proxy upgrade safety

## Independent Security Reviews

Security reviews carried out with my own audit process and report template. My first independent
reviews are in progress and will be published here.

## Practice & Course Audits

Reviews of educational codebases from the **[Cyfrin Updraft](https://updraft.cyfrin.io/)** security
curriculum — the foundation of my training. **40 findings across 5 protocols.**

| Date       | Protocol           | Type                     | Findings                         | Report |
| ---------- | ------------------ | ------------------------ | -------------------------------- | :----: |
| 2026-06-29 | **Boss Bridge**    | Token bridge (L1 → L2)   | `3C`                             | [↗](https://github.com/joevly/audits/tree/main/reports/2026-06-29-boss-bridge/boss-bridge-audit.pdf) |
| 2026-06-24 | **Thunder Loan**   | Flash loan · UUPS proxy  | `3H` · `1M`                      | [↗](https://github.com/joevly/audits/tree/main/reports/2026-06-24-thunder-loan-audit/thunder-loan-audit.pdf) |
| 2026-06-14 | **TSwap**          | ERC-20 AMM (DEX)         | `4H` · `1M` · `2L` · `5I`        | [↗](https://github.com/joevly/audits/tree/main/reports/2026-06-14-t-swap-audit/t-swap-audit.pdf) |
| 2026-06-06 | **Puppy Raffle**   | ERC-721 raffle           | `3H` · `2M` · `1L` · `7I` · `2G` | [↗](https://github.com/joevly/audits/tree/main/reports/2026-06-06-puppy-raffle-audit/puppy-raffle-audit.pdf) |
| 2026-05-31 | **Password Store** | Storage / access control | `2H` · `3L` · `1I`               | [↗](https://github.com/joevly/audits/tree/main/reports/2026-05-31-password-store-audit/password-store-audit.pdf) |

<sub>Severity: **C** Critical · **H** High · **M** Medium · **L** Low · **I** Informational · **G** Gas</sub>

Full PDF + Markdown reports live in my **[audits](https://github.com/joevly/audits)** repository.

## Competitive Audits & Bug Bounties

Actively sharpening my edge on competitive platforms (**CodeHawks**, **Sherlock**, **Cantina**)
and bug-bounty programs (**Immunefi**). Placements and bounties will be listed here as I earn them.

## Get in Touch

Open to **private audits**, **collaborations**, and **security discussions**.

| | |
| --- | --- |
| **GitHub**  | [@joevly](https://github.com/joevly) |
| **Twitter/X** | [@Joevly7](https://x.com/Joevly7) |
| **Discord**  | `joevly` |

