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
The guardrail adds +35 to the risk score for any of these.

| Selector | Signature | Risk Category | Why Dangerous |
|----------|-----------|--------------|--------------|
| `0x8f283970` | `changeOwner(address)` | CRITICAL | Direct ownership theft — replaces contract owner |
| `0x13af4035` | `setOwner(address)` | CRITICAL | Alternative ownership transfer pattern |
| `0xf2fde38b` | `transferOwnership(address)` | HIGH | OpenZeppelin Ownable — only legitimate in admin workflows |
| `0x715018a6` | `renounceOwnership()` | HIGH | Permanently removes ownership — irreversible |
| `0x9dc29fac` | `burn(address,uint256)` | HIGH | Forced token burn from any address — used in exploits |
| `0x00f714ce` | `withdraw(uint256,address)` | MEDIUM | Triggers withdrawals — verify the target address |
| `0x4f1ef286` | `upgradeToAndCall(address,bytes)` | HIGH | Proxy upgrade — can replace contract implementation |
| `0x3659cfe6` | `upgradeTo(address)` | HIGH | Proxy upgrade — can replace contract implementation |
| `0x42966c68` | `burn(uint256)` | LOW-MEDIUM | Self-burn, legitimate in some protocols |

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
