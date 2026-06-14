---
name: pharos-tx-guardrail
description: >
  Mandatory pre-execution security checkpoint for any write transaction on Pharos Network.
  Simulates the transaction with eth_call, checks contract existence and code size, detects
  unlimited token approvals (MAX_UINT256), identifies dangerous function selectors (ownership
  transfers, forced burns), estimates gas, reads the transaction value in USD via Chainlink,
  and returns a deterministic risk score 0–100 with a SAFE / LOW / MEDIUM / HIGH / CRITICAL
  rating and a clear PROCEED / WARN / BLOCK recommendation.
  Invoke this skill BEFORE every cast send, forge script --broadcast, or any on-chain write
  that moves user funds. Also invoke when a user asks "is this contract safe?", "should I
  approve this?", "check this transaction", "scan this address", "verify this contract",
  "is this a honeypot?", or any security question about a Pharos transaction or contract.
  Agents MUST call this skill before executing any transaction over $10 in value. Never skip
  it for approval transactions regardless of amount — unlimited approvals are always high risk.
version: 0.1.0
requires:
  anyBins:
    - cast
---

# Pharos Transaction Safety Guardrail

Pre-execution security checkpoint for Pharos Network. Every write transaction passes through
this skill first. One bad transaction can drain a wallet — this skill is the last line of defense.

## Prerequisites

1. **Install Foundry** — check first with `which cast` or `cast --version`.
   If missing:
   ```bash
   curl -L https://foundry.paradigm.xyz | bash
   source ~/.zshenv && foundryup
   cast --version
   ```
2. **Network configuration** — read `assets/networks.json`. Default network is `atlantic-testnet`.
   ```bash
   RPC=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
   ```
3. No private key is required — this skill is **read-only**.

## Capability Index

| User Need | Capability | Detailed Instructions |
|-----------|-----------|----------------------|
| Assess transaction risk before sending | Full 6-check pipeline: simulate, contract check, selector, approval, value, gas | → `references/tx-guardrail.md#full-assessment` |
| Simulate a transaction without sending | `cast call` eth_call | → `references/tx-guardrail.md#simulate-transaction` |
| Check if a contract is safe / verified | `cast code` + explorer API | → `references/tx-guardrail.md#contract-check` |
| Detect unlimited token approval | Calldata pattern match on selector + amount | → `references/tx-guardrail.md#approval-check` |
| Identify a function selector | `cast 4byte` lookup | → `references/tx-guardrail.md#selector-check` |
| Estimate gas for a transaction | `cast estimate` | → `references/tx-guardrail.md#gas-estimation` |
| Get USD value of a token amount | Chainlink oracle `latestAnswer()` | → `references/tx-guardrail.md#usd-value` |
| Get risk score as structured JSON | Full assessment with `--json` flag logic | → `references/tx-guardrail.md#full-assessment` |

## Risk Score Reference

```
Base score: 0

+ 50  Target address has no contract code (sending to EOA/empty)
+ 70  Unlimited approval detected (approve MAX_UINT256) — auto-promotes to HIGH/BLOCK
+ 35  Selector matches known dangerous function (ownership transfer, forced burn)
+ 30  Contract source code not verified on explorer
+ 20  Transaction simulation reverted
+ 20  Sending > 50% of wallet's native balance
+ 15  Contract bytecode < 100 bytes (suspiciously minimal)

Risk Level:
  0–29:   SAFE     → PROCEED
 30–49:   LOW      → PROCEED with note
 50–69:   MEDIUM   → WARN user, require explicit confirmation
 70–84:   HIGH     → BLOCK — strong warning, do not execute
 85–100:  CRITICAL → BLOCK — do not proceed under any circumstances
```

### Override Rules (apply AFTER base score calculation)

These rules override the numeric score when triggered:

1. **Any unlimited approval** → final score = `max(score, 70)` → forces BLOCK
2. **Contract code = `0x`** → final score = `max(score, 85)` → forces CRITICAL BLOCK
3. **Selector ∈ {`changeOwner`, `setOwner`, `upgradeTo`, `burn(address,uint256)`}** → final score = `max(score, 70)` → forces BLOCK
4. **Simulation reverted AND value > 0** → final score = `max(score, 50)` → forces at least MEDIUM

Always show all checks to the user, including passed ones. Transparency builds trust.

## Decision Rules for Agents

After running the assessment:

| Result | Action |
|--------|--------|
| `risk_score < 50` AND no BLOCK flag | Execute |
| `risk_score 50–69` | Show full report, ask user for explicit confirmation before executing |
| `risk_score >= 70` | Show full report with each flag explained. Do NOT execute. |
| Any check returns `recommendation: block` | ALWAYS stop — even if user insists |
| Assessment script fails / RPC unreachable | Treat as HIGH risk (fail-safe, not fail-open) |

## Security Reminders

- This skill makes only `cast call` (read-only) and `curl` requests. It never sends transactions.
- Private keys are never read, stored, or referenced in this skill.
- If the RPC is unreachable, the simulation check fails — report HIGH risk and stop.
- Never suppress or hide a warning flag. Show everything.
- Unlimited MAX_UINT256 approvals always get minimum HIGH risk (score ≥ 70), regardless of
  how trusted the spender appears.
- Unverified contracts always get score contribution of +30.

## Output Format

Always present the result in this format:

```
🛡️ Pharos Transaction Guardrail
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Target:      <address>
Function:    <decoded name or selector>
Value:       <amount> <native token> (~$<usd>)
Network:     <network name>

Risk Score:  <N>/100 — <LEVEL> <emoji>
Action:      <PROCEED | WARN AND CONFIRM | BLOCK>

Checks:
  <✅/⚠️/🔴/🚨> <check_name>: <finding>
  ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
<PROCEED / WARN message / BLOCK message>
```

## Error Handling

| Error | Handling |
|-------|----------|
| `cast` not installed | Stop, show installation instructions from Prerequisites |
| RPC unreachable / timeout | Mark simulation check as FAILED, add +20 score, continue remaining static checks |
| Explorer API unavailable | Mark verification check as UNKNOWN, add +10 score, continue |
| `cast 4byte` returns no match | Report selector as unknown (not a risk in itself) |
| `cast estimate` fails | Report gas unknown, add +5 score |
| Invalid address format | Stop immediately, prompt user to fix address |
