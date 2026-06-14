# pharos-tx-guardrail

**Pre-execution transaction safety guardrail for Pharos Network AI Agents.**

Every write transaction on Pharos passes through this skill first. It simulates the transaction, checks the target contract, detects dangerous patterns, and returns a risk score **0–100** with a clear **PROCEED / WARN / BLOCK** recommendation — before anything is signed or sent.

[![Pharos Network](https://img.shields.io/badge/Pharos-Mainnet%201672-6B4FFF?style=flat-square)](https://pharos.xyz)
[![Hackathon](https://img.shields.io/badge/AI%20Agent%20Carnival-Phase%201-00C2A8?style=flat-square)](https://dorahacks.io/hackathon/pharos-phase1/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)

---

## What it does

When an AI agent is about to call `cast send` or `forge script --broadcast`, this skill runs 6 read-only checks:

| # | Check | How | Risk Contribution |
|---|-------|-----|-------------------|
| 1 | **Simulation** | `eth_call` with `--from sender` | +20 if reverts |
| 2 | **Contract existence** | `eth_getCode` | +50 if empty / +15 if < 100 bytes |
| 3 | **Selector decode** | `cast 4byte` + known dangerous table | +35 for ownership/upgrade ops |
| 4 | **Unlimited approval** | Calldata pattern match on MAX_UINT256 | +70 → auto-BLOCK |
| 5 | **Value check** | `cast balance` + Chainlink PROS/USD | +20 if > 50% wallet balance |
| 6 | **Gas estimation** | `eth_estimateGas` | +5 if estimate fails |

**Score → Action:**

| Score | Level | Action |
|-------|-------|--------|
| 0 – 29 | SAFE ✅ | PROCEED |
| 30 – 49 | LOW ✅ | PROCEED with note |
| 50 – 69 | MEDIUM ⚠️ | WARN — require confirmation |
| 70 – 84 | HIGH 🔴 | BLOCK |
| 85 – 100 | CRITICAL 🚨 | BLOCK |

**Override rules** (applied after scoring):
- Unlimited approval → score = max(score, 70) → always BLOCK
- No contract code → score = max(score, 85) → always CRITICAL
- changeOwner / upgradeTo / setOwner in calldata → score = max(score, 70)

---

## Quick example

```
🛡️ Pharos Transaction Guardrail
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Target:      0xc879c018...1815 (USDC on Pharos mainnet)
Function:    approve(address,uint256)
Value:       0 PROS

Risk Score:  70/100 — HIGH 🔴
Action:      BLOCK

Checks:
  ✅ simulation: approval would succeed
  ✅ contract_exists: real contract, 1798 bytes
  ✅ selector: approve(address,uint256)
  🔴 unlimited_approval: MAX_UINT256 detected — permanent access to ALL your USDC
  ✅ value_check: no native value sent
  ✅ gas_estimate: 71,897 gas

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚫 BLOCKED — Use exact amount instead (e.g., 500000000 for 500 USDC)
```

---

## Installation

### Pharos Skill Engine / Claude Code

```bash
git clone https://github.com/hosein-ul/pharos-tx-guardrail \
  ~/.claude/skills/pharos-tx-guardrail
```

The skill is auto-triggered when the agent detects any transaction-sending intent.

### Anvita Flow

Submit `https://github.com/hosein-ul/pharos-tx-guardrail` in the Skill Hub.

### Manual / Any AI Agent

```bash
git clone https://github.com/hosein-ul/pharos-tx-guardrail
# Point your agent to pharos-tx-guardrail/SKILL.md as the entry point
```

**Prerequisite — Foundry:**

```bash
curl -L https://foundry.paradigm.xyz | bash && foundryup
cast --version   # verify
```

---

## File structure

```
pharos-tx-guardrail/
├── SKILL.md                  ← Agent entry point (read this first)
├── assets/
│   ├── networks.json         ← RPC URLs, chain IDs, explorer URLs
│   ├── tokens.json           ← USDC, WETH, WPHRS addresses
│   └── oracles.json          ← Chainlink Push Engine feed addresses
├── references/
│   ├── tx-guardrail.md       ← 6-check pipeline with exact cast commands
│   └── risk-patterns.md      ← Dangerous selectors DB + detection logic
└── evals/
    └── evals.json            ← 4 test scenarios
```

---

## Networks

| Network | Chain ID | RPC | Native |
|---------|----------|-----|--------|
| Pacific Mainnet | 1672 | `https://rpc.pharos.xyz` | PROS |
| Atlantic Testnet | 688689 | `https://atlantic.dplabs-internal.com` | PHRS |

Default: mainnet (chain 1672).

---

## Live contract addresses (mainnet)

Chainlink oracles used for USD value calculation:

| Feed | Address |
|------|---------|
| PROS/USD | `0x9356C87a48F913d11C87a0d4b8cD16CD04624BF3` |
| ETH/USD | `0x092ff0175Be8B2e83Ca5740d3EB13C6225901fa7` |
| USDC/USD | `0x8d08eA83A55ad1e805b5660F5eC76C99C6aF5eaf` |

---

## This skill is part of the Pharos AI Agent Carnival — Phase 1

[Hackathon page](https://dorahacks.io/hackathon/pharos-phase1/) · [Pharos Network](https://pharos.xyz) · [Companion skill: pharos-rwa-yield-router](https://github.com/hosein-ul/pharos-rwa-yield-router)
