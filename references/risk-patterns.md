# Risk Patterns Reference

Quick lookup database for the guardrail agent. Contains known dangerous function selectors,
unlimited approval detection logic, and honeypot indicators.

---

## Table of Contents
1. [Dangerous Function Selectors](#dangerous-function-selectors)
2. [Known Safe Selectors](#known-safe-selectors)
3. [Unlimited Approval Detection](#unlimited-approval-detection)
4. [Honeypot Indicators](#honeypot-indicators)
5. [Common Phishing Patterns](#common-phishing-patterns)

---

## Dangerous Function Selectors

These selectors indicate high-privilege operations or known exploit patterns.

`DANGEROUS_AUTO_BLOCK` set (auto-promote to score ≥70 per Override Rule #3):
- `0x8f283970`, `0x13af4035` — ownership theft
- `0x4f1ef286`, `0x3659cfe6` — proxy upgrade (replaces implementation)
- `0x9dc29fac` — burn(address,uint256) (forced burn)

| Selector | Signature | Risk Category | Score Behavior |
|----------|-----------|--------------|--------------|
| `0x8f283970` | `changeOwner(address)` | CRITICAL | **+70 auto-block** — Direct ownership theft |
| `0x13af4035` | `setOwner(address)` | CRITICAL | **+70 auto-block** — Alt ownership transfer |
| `0x4f1ef286` | `upgradeToAndCall(address,bytes)` | HIGH | **+70 auto-block** — Proxy upgrade |
| `0x3659cfe6` | `upgradeTo(address)` | HIGH | **+70 auto-block** — Proxy upgrade |
| `0x9dc29fac` | `burn(address,uint256)` | HIGH | **+70 auto-block** — Forced burn from any addr |
| `0xf2fde38b` | `transferOwnership(address)` | HIGH | +35 — OpenZeppelin Ownable |
| `0x715018a6` | `renounceOwnership()` | HIGH | +35 — Irreversible ownership removal |
| `0x00f714ce` | `withdraw(uint256,address)` | MEDIUM | +35 — Withdrawals to address |
| `0x42966c68` | `burn(uint256)` | LOW-MEDIUM | +15 — Self-burn (often legit) |
| `0x40c10f19` | `mint(address,uint256)` | MEDIUM | +20 — Token minting (verify caller is privileged) |
| `0xa9059cbb` | `transfer(address,uint256)` | SAFE | +0 — Standard ERC-20 transfer |
| `0x23b872dd` | `transferFrom(address,address,uint256)` | SAFE | +0 — Standard ERC-20 transferFrom |
| `0x095ea7b3` | `approve(address,uint256)` | DEPENDS | See Check 4 (unlimited detection) |
| `0xd0e30db0` | `deposit()` | SAFE | +0 — Common WETH/vault wrap |
| `0x2e1a7d4d` | `withdraw(uint256)` | SAFE | +0 — Common vault unwrap |
| `0x38ed1739` | `swapExactTokensForTokens(...)` | SAFE | +0 — Uniswap V2 swap |
| `0x7ff36ab5` | `swapExactETHForTokens(...)` | SAFE | +0 — Uniswap V2 ETH swap |
| `0x4cdad506` | `previewRedeem(uint256)` | SAFE | +0 — ERC-4626 view (read-only) |
| `0x01e1d114` | `totalAssets()` | SAFE | +0 — ERC-4626 view |
| `0x6e553f65` | `deposit(uint256,address)` | SAFE | +0 — ERC-4626 deposit |
| `0xba087652` | `redeem(uint256,address,address)` | SAFE | +0 — ERC-4626 redeem |

---

## Known Safe Selectors

These selectors are standard ERC-20/ERC-721 operations. No score increase.

| Selector | Signature | Notes |
|----------|-----------|-------|
| `0xa9059cbb` | `transfer(address,uint256)` | Standard ERC-20 transfer |
| `0x23b872dd` | `transferFrom(address,address,uint256)` | Standard ERC-20 transferFrom |
| `0x70a08231` | `balanceOf(address)` | Read-only query |
| `0x18160ddd` | `totalSupply()` | Read-only query |
| `0xd0e30db0` | `deposit()` | Wrap/deposit — common in WETH and vaults |
| `0x2e1a7d4d` | `withdraw(uint256)` | Unwrap/withdraw — standard vault pattern |
| `0x38ed1739` | `swapExactTokensForTokens(...)` | Uniswap V2 swap |
| `0x7ff36ab5` | `swapExactETHForTokens(...)` | Uniswap V2 ETH swap |

---

## Unlimited Approval Detection

### What it is
`approve(address spender, uint256 amount)` with `amount = type(uint256).max`.

### Selector
`0x095ea7b3`

### MAX_UINT256 constant
```
ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```
(64 hex chars = 32 bytes = 2^256 - 1)

### Detection logic
```
calldata_lower = lowercase(calldata).replace("0x", "")
selector = calldata_lower[0:8]

if selector == "095ea7b3":
    if "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff" in calldata_lower:
        → UNLIMITED APPROVAL DETECTED (+40 risk score)
    else:
        → Limited approval — show amount
```

### Extracting the spender address
```
calldata (no 0x): [selector 8] [spender padded 64] [amount 64]
spender_padded = calldata_lower[8:72]
spender = "0x" + spender_padded[24:64]  # last 20 bytes
```

### Why this is dangerous
An unlimited approval gives the spender **permanent, irrevocable** permission to take ALL
tokens of that type from the wallet — now and in the future. If the spender contract is
later exploited, phished, or turns malicious, the attacker can drain everything.

**Always recommend**: approve only the exact amount needed for the current transaction.

---

## Honeypot Indicators

A honeypot lets you buy tokens but blocks selling. Look for these during simulation:

1. **Simulation succeeds for `buy/deposit` but reverts for `sell/withdraw`** — the contract
   blocks exit. Test by simulating a withdraw call after a deposit call.
2. **Bytecode < 100 bytes** — too small to contain a real ERC-20 implementation; likely a proxy
   to a hidden malicious contract, or a stub that captures funds.
3. **Unverified source code + high APY claim** — cannot audit the logic.
4. **No prior transactions on the contract** — brand new with no track record.

---

## Common Phishing Patterns

### 1. Address lookalike
Real USDC: `0xE0BE08c77f415F577A1B3A9aD7a1Df1479564ec8`  
Fake USDC: `0xE0BE08c77f415F577A1B3A9aD7a1Df1479564eCc8` ← extra char

Always verify addresses character by character or use the tokens.json registry.

### 2. Unverified proxy
Contract appears verified but delegates all calls to an unverified implementation.
A verified proxy with an unverified implementation is still a risk.

### 3. Router impersonation
Phishing sites deploy fake DEX routers with legitimate-looking names.
Always verify the DEX router address against the official Pharos ecosystem contracts.

### 4. Fake permit2 / approve front-running
Some phishing flows trick users into signing `permit` messages that grant unlimited access
without an on-chain `approve` call. The guardrail does not cover off-chain signatures —
remind users to check what they're signing in their wallet.
