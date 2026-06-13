# Transaction Guardrail — Reference

All operations are read-only. No private key is required. The RPC URL is read from
`assets/networks.json` for the current network (default: `atlantic-testnet`).

> **Setup**: Before running any command, load the network config:
> ```bash
> RPC=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
> EXPLORER_API=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .explorerApiUrl' assets/networks.json)
> NATIVE=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .nativeToken' assets/networks.json)
> ```

---

## Full Assessment

Run all six checks in sequence, accumulate the risk score, then output the combined report.

### Step-by-step workflow

```
1. Read network config from assets/networks.json
2. Run: Simulate Transaction     → adds to score
3. Run: Contract Check           → adds to score
4. Run: Selector Check           → adds to score
5. Run: Approval Check           → adds to score
6. Run: Value Check (if value>0) → adds to score
7. Run: Gas Estimation           → adds to score
8. Compute total score (cap at 100)
9. Determine risk level and recommendation
10. Print full report
```

Each check below documents exactly what to run and how to interpret the result.

---

## Check 1: Simulate Transaction

**Purpose**: Detect transactions that will revert. A reverting transaction suggests wrong
calldata, wrong contract, wrong state — any of which could mean the agent hallucinated the target.

**Command Template**

```bash
cast call <target> <calldata> \
  [--from <sender>] \
  [--value <value_wei>] \
  --rpc-url <rpc>
```

**Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `<target>` | address | Yes | Contract address being called |
| `<calldata>` | hex | Yes | ABI-encoded function call (0x-prefixed) |
| `<sender>` | address | No | Simulated caller address |
| `<value_wei>` | uint256 | No | Native token amount in wei (omit if 0) |
| `<rpc>` | URL | Yes | From `assets/networks.json` |

**Output Parsing**

| Result | Interpretation | Score Impact |
|--------|---------------|-------------|
| Returns any hex data | Simulation succeeded | +0 |
| Command exits with error / `execution reverted` | Transaction would fail on-chain | +20 |
| RPC connection refused / timeout | Cannot simulate, network unreachable | +20 |

**Agent Guidelines**

1. Run the cast call exactly as the user would send the transaction.
2. If stderr contains `execution reverted`, extract the revert reason if present: look for text after `revert:`.
3. On success, show the first 66 chars of the return value.
4. If RPC unreachable, mark this check FAILED and continue — do not abort the entire assessment.

---

## Check 2: Contract Check

**Purpose**: Confirm the target is an actual smart contract. Sending a write transaction to
an EOA (externally owned account) or empty address almost certainly means the agent used a
wrong address. Also, very small contracts are suspicious (honeypot / proxy minimal bytecode).

**Step 2a — Check contract existence**

```bash
cast code <target> --rpc-url <rpc>
```

**Output Parsing**

| Output | Interpretation | Score Impact |
|--------|---------------|-------------|
| `0x` or `0x0` | No contract at this address — it is an EOA or empty | +50 |
| Hex string, length = `(len-2)/2` bytes | Contract exists. Check bytecode length. | see below |

**Step 2b — Check bytecode size**

After confirming a contract exists, calculate bytecode size:
```
bytecode_bytes = (len(output) - 2) / 2
```

| Bytecode size | Interpretation | Score Impact |
|---------------|---------------|-------------|
| < 100 bytes | Suspiciously minimal — possible proxy stub or honeypot | +15 |
| ≥ 100 bytes | Normal contract size | +0 |

**Step 2c — Check verification status (optional)**

```bash
curl -s "<EXPLORER_API>/v1/contracts/<target>" \
  -H "accept: application/json"
```

| Response | Interpretation | Score Impact |
|----------|---------------|-------------|
| `"is_verified": true` | Source code is public and auditable | +0 |
| `"is_verified": false` or empty | Unverified — cannot audit what the contract does | +30 |
| API unavailable (non-200 / network error) | Verification unknown | +10 |

**Agent Guidelines**

1. Always run Step 2a first. If no contract code, report CRITICAL and stop further checks.
2. Run Step 2b if contract exists.
3. Attempt Step 2c. If the API returns an unexpected response, treat as unknown (+10) and continue.
4. Show the bytecode size in the report so the user can judge.

---

## Check 3: Selector Check

**Purpose**: Identify what function is being called. Some selectors are associated with high-risk
or administrative operations that should never be called by a regular agent workflow.

**Step 3a — Extract selector**

The first 4 bytes (8 hex chars after `0x`) of the calldata are the function selector.

```
selector = calldata[0:10]   (e.g., "0x095ea7b3")
```

**Step 3b — Look up selector**

```bash
cast 4byte <selector>
```

**Output Parsing**

- Returns one or more function signatures matching the selector (from 4byte.directory).
- Multiple results are possible — take the first match.
- No output means the selector is unknown.

**Known Risk Classifications**

| Selector | Function | Risk | Score Impact |
|----------|----------|------|-------------|
| `0x8f283970` | `changeOwner(address)` | CRITICAL | +35 |
| `0x13af4035` | `setOwner(address)` | CRITICAL | +35 |
| `0xf2fde38b` | `transferOwnership(address)` | HIGH | +35 |
| `0x715018a6` | `renounceOwnership()` | HIGH | +35 |
| `0x9dc29fac` | `burn(address,uint256)` | HIGH — forced token burn | +35 |
| `0x42966c68` | `burn(uint256)` | MEDIUM — self burn | +15 |
| `0x095ea7b3` | `approve(address,uint256)` | Requires Check 4 | (see Check 4) |
| All others   | Unknown or standard      | +0 |

**Agent Guidelines**

1. Extract the selector from the first 10 characters of calldata.
2. First check against the known risk table above.
3. Then run `cast 4byte <selector>` to decode the human-readable name.
4. If the decoded name contains words like "owner", "admin", "upgrade", "migrate" — flag as worth reviewing
   even if not in the table.
5. Report the full decoded signature in the output.

---

## Check 4: Approval Check

**Purpose**: Detect unlimited token approvals. An `approve(address, MAX_UINT256)` gives
permanent, irrevocable permission to drain all tokens of that type from the wallet.
This is one of the most common phishing vectors in DeFi.

**Conditions to trigger this check**

Only applies when:
- Selector is `0x095ea7b3` (`approve(address,uint256)`)

**Detection**

```
MAX_UINT256 = "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"

If calldata (lowercased, 0x stripped) contains MAX_UINT256 → unlimited approval detected
```

**Score Impact**

| Finding | Score Impact |
|---------|-------------|
| Not an approval transaction | +0 |
| Approval with specific amount | +0 (safe pattern) |
| Approval with MAX_UINT256 | +40 |

**Agent Guidelines**

1. Only run this check when selector is `0x095ea7b3`.
2. Lowercase the entire calldata and strip the `0x` prefix before searching.
3. If unlimited approval detected, extract the spender address (bytes 12–31 of the calldata
   after stripping selector, i.e., calldata[10:74] → last 40 chars are the spender address).
4. Always warn the user and suggest approving only the exact amount needed instead.
5. Show the spender address explicitly in the warning.

---

## Check 5: Value Check

**Purpose**: Alert when a large portion of the wallet's balance is being sent. Only run when
`value_wei > 0`.

**Step 5a — Get native token balance**

```bash
cast balance <sender> --rpc-url <rpc>
```

**Step 5b — Get USD value via Chainlink oracle**

```bash
FEED=$(jq -r '."atlantic-testnet".feeds."PROS/USD"' assets/oracles.json)
cast call $FEED "latestAnswer()(int256)" --rpc-url <rpc>
```

Result is in 18-decimal fixed-point. Convert: `price_usd = raw / 1e18`.

**Score Impact**

| Finding | Score Impact |
|---------|-------------|
| `value_wei = 0` | Skip this check, +0 |
| Sending ≤ 10% of balance | +0 |
| Sending 10–50% of balance | +5 |
| Sending > 50% of balance | +20 |

**Agent Guidelines**

1. Only run this check when value_wei > 0.
2. Calculate `percent = value_wei * 100 / balance_wei`.
3. Calculate `usd_value = (value_wei / 1e18) * price_usd`.
4. Always show both the native token amount and the USD equivalent in the report.
5. If sender address is unknown, skip balance comparison — only show USD value.

---

## Check 6: Gas Estimation

**Purpose**: Verify the transaction can be included in a block and catch unexpectedly high gas.

**Command Template**

```bash
cast estimate <target> <calldata> \
  [--from <sender>] \
  [--value <value_wei>] \
  --rpc-url <rpc>
```

**Output Parsing**

| Result | Interpretation | Score Impact |
|--------|---------------|-------------|
| Returns gas number | Show estimate to user | +0 |
| `execution reverted` | Already caught in Check 1 | +0 (already scored) |
| Any other error | Estimation failed | +5 |

**Agent Guidelines**

1. Only run after simulation succeeds (if simulation reverted, estimation is moot).
2. Convert gas to approximate USD cost: `gas_cost_usd = (gas_units * gas_price_gwei * 1e-9) * native_price_usd`.
3. Fetch gas price: `cast gas-price --rpc-url <rpc>` (result in wei, divide by 1e9 for gwei).
4. Show gas estimate and estimated USD cost in the report.

---

## USD Value

Standalone USD price lookup for any supported token using Chainlink.

**Command Template**

```bash
FEED=$(jq -r '."<network>".feeds."<PAIR>"' assets/oracles.json)
cast call $FEED "latestAnswer()(int256)" --rpc-url <rpc>
```

**Supported Pairs** (Atlantic testnet)

| Pair | Feed Address |
|------|-------------|
| PROS/USD | `0x67488Fac9Bc4174a53a485b11F2066498Cd34b3A` |
| ETH/USD  | `0xCd47D1843f3D6313836303fE1434BA26D257d500` |
| BTC/USD  | `0x82d0e03ea6d94120B92EA4Ea236DcFA273D42994` |
| USDC/USD | `0xDF6afcf662345Ea29ceACa6DA06141d828c516EA` |
| USDT/USD | `0x2f7796B346d01a3f2264Ff0D93dDdFF8680b8B66` |

**Output Parsing**

- Returns `int256` with 18 decimal places.
- `price_usd = raw_value / 1e18`
- Example: `1800000000000000000000` → `$1800.00`

---

## Risk Score Computation

After all checks complete, compute total score:

```
total = sum of all score_delta values from checks above
total = min(100, max(0, total))
```

Map to level:

| Score | Level | Emoji | Recommendation |
|-------|-------|-------|---------------|
| 0–29  | SAFE | ✅ | PROCEED |
| 30–49 | LOW  | ✅ | PROCEED (with note) |
| 50–69 | MEDIUM | ⚠️ | WARN AND CONFIRM |
| 70–84 | HIGH | 🔴 | BLOCK (explain each flag) |
| 85–100 | CRITICAL | 🚨 | BLOCK |
