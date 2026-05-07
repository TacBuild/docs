---
title: EVM Node
description:
---

# Bug Bounty Report Guide

The report template below covers vulnerabilities in EVM-compatible [Cosmos TAC chain node](https://github.com/TacBuild/evm): precompiles,
EVM↔Cosmos state synchronisation, bank/staking/distribution module bugs.
Tools: Hardhat (EVM side), Cosmos REST API, Go node source.

## Report Template

```
SECURITY VULNERABILITY REPORT
<Network> — <Severity> <Class> in <Component>
Date: YYYY-MM-DD
Status: <THEORETICAL | LOCAL REPRODUCED | TESTNET CONFIRMED | MAINNET CONFIRMED>


===============================
EXECUTIVE SUMMARY
===============================

<5–10 lines: what is broken, what the attacker gains, specific numbers>


===============================
VULNERABILITY DETAILS
===============================

Affected component:
  <file>:<lines>, <function name>

Root cause:
  <one or two sentences at the line level>

How it works:
  <EVM↔Cosmos mechanism, 1–3 paragraphs>

Technical prerequisite:
  <conditions for exploitation, if any>

Comparison with correct code:
  <adjacent correct function, if applicable>


===============================
EXPLOITATION STEPS
===============================

<numbered list with specific arguments and before/after state>


===============================
PROOF-OF-CONCEPT RESULTS
===============================

Test: <name> (<scale>)
  Contract:   <addr>
  Exploit TX: <hash>
  Block:      <number>
  <money-flow table>
  Status: CONFIRMED

Scaling economics:
  <fund → phantom → derivative → gas → profit>


===============================
IMPACT ASSESSMENT
===============================

Severity: <CRITICAL | HIGH | MEDIUM | LOW>

  1. <impact point>
  2. <impact point>
  ...


===============================
RECOMMENDED FIX
===============================

Immediate:   <Go diff / pseudo-diff>
Long-term:   <architectural proposal>
Audit:       <adjacent functions to review>


===============================
PROOF-OF-CONCEPT CODE
===============================

<full self-contained PoC>


===============================
DISCLOSURE TIMELINE
===============================

YYYY-MM-DD   <event>
YYYY-MM-DD   <event>
```

## Report Template Commentary

## 1. Title and Metadata

```
SECURITY VULNERABILITY REPORT
<Network> — <Severity> <Class> in <Component>
Date: YYYY-MM-DD
Status: <THEORETICAL | LOCAL REPRODUCED | TESTNET CONFIRMED | MAINNET CONFIRMED>
```

- One line: **network + vulnerability class + affected component**.
- `Status` must state the confirmation level:
  `THEORETICAL` / `LOCAL REPRODUCED` / `TESTNET CONFIRMED` / `MAINNET CONFIRMED`.

Useful classes:
`Balance Desync`, `State Collision`, `Precompile Missing Invariant`,
`Bank Module Bypass`, `IBC Replay`, `Consensus Equivocation`.

### 2. Executive Summary

Triage reads this in 30 seconds. Must answer four questions:

1. **What is broken?** — one sentence on the root cause.
2. **What does the attacker gain?** — tokens / control / data.
3. **How much?** — specific numbers.
4. **Is it repeatable?** — yes/no.

**Example:**

```
An attacker who starts with X native tokens ends up with X + profit
after a single exploit cycle. The profit scales linearly with the amount
invested. Gas cost is fixed regardless of exploit scale.

  Invested:            10 <token>
  Phantom balance:      5 <token>   (should be 0 after staking)
  Derivative received: 9.72 <lsd-token>
  Net profit:          +2.01 <token> equivalent

The attack is repeatable and can be looped for unlimited profit.
```

### 3. Vulnerability Details

#### 3.1 Affected Component

State the **exact** location in the node source:

```
Affected component:
  precompiles/<module>/tx.go — function <FunctionName> (lines X–Y)
```

Permalink to a specific commit if the repo is public:

```
https://github.com/<org>/<repo>/blob/<commit>/precompiles/<module>/tx.go#LX-LY
```

#### 3.2 Root Cause

One or two sentences — **at the line-of-code level**, not "somewhere in the module":

> `<FunctionName>` has zero calls to `<MissingFunction>()`.
> Every adjacent precompile that performs the same operation includes this call —
> `<FunctionName>` was missed.

#### 3.3 How It Works

1. **Context:** how the EVM↔Cosmos state bridge normally works.
2. **What the vulnerable code does:** step by step.
3. **Where the logic breaks:** the specific missing call / condition.
4. **Observable effect:** what state inconsistency is produced.

#### 3.4 Technical Prerequisite

Any conditions required for exploitation (directly affects severity):

```
Example: the exploit contract must perform at least one storage write (SSTORE)
before calling the vulnerable precompile, so the EVM stateObject is marked
"dirty" in the StateDB journal and the deduction is overwritten at commit.
```

#### 3.5 Comparison with Correct Code

If an **adjacent function is implemented correctly** — always show the diff.
This is the strongest evidence that it's a missed piece, not an architectural flaw:

```go
// <AdjacentFunction> (correct):
if <condition> {
    p.SetBalanceChangeEntries(
        cmn.NewBalanceChangeEntry(callerAddr, amount, cmn.Sub),
    )
}

// <VulnerableFunction> — this block is missing entirely.
```

### 4. Exploitation Steps

One transaction = one step. Include **specific arguments** and **before/after state**.

```
TX N — <stepName>():
  <describe what the tx does and why it is necessary>
  Calls:
    PRECOMPILE.<method>(
        arg1,   // description
        arg2,   // description
    )
  Expected (correct): balance decreases by X.
  Actual (buggy):     balance unchanged — phantom balance created.
```

### 5. PoC Results

The most valuable part for triage: **real on-chain data**.

Required:
- Deployment address of the PoC contract.
- TX hashes for every step.
- Block numbers.
- Money-flow table.
- Final profit calculation.

**Example table:**

```
STEP                 CONTRACT BALANCE    WALLET CHANGE        NOTES
----                 ----------------    -------------        -----
Start                0                                        wallet: NNN <token>
Fund contract        X                   -X
step1_setup()        X/2                 -gas                 X/2 locked
step2_exploit()      X/2  (PHANTOM)      -gas                 BUG: should be 0
step3_withdraw()     0                   +X/2                 phantom funds to wallet
step4_profit()       0                   +Y <lsd-token>       derivative to wallet
```

#### 5.1 Scaling Economics

```
Fund amount    Phantom balance    Derivative received    Gas cost    Net profit
-----------    ---------------    -------------------    --------    ----------
10  <token>    5   <token>        ~9.7  <lsd>            ~C <token>  +P1 <token>
100 <token>    50  <token>        ~97   <lsd>            ~C <token>  +P2 <token>
1K  <token>    500 <token>        ~970  <lsd>            ~C <token>  +P3 <token>
```

### 6. Impact Assessment

```
Severity: CRITICAL

  1. Unlimited native token inflation — balance deduction is silently skipped.
  2. Direct profit extraction — no social engineering required.
  3. Repeatable and linearly scalable — gas is the only cost.
  4. Minimal prerequisites — <describe any required conditions>.
  5. Core module accounting corrupted — staking + bank state diverge.
```

CVSS v3.1 vector if the program requires it. Factors that raise severity:
native token inflation, no privilege requirements, breakage of core modules
(bank, staking, IBC).

### 7. Recommended Fix

#### Immediate

Minimal change — a specific diff or pseudo-diff in Go:

```go
// Add after the Cosmos-side operation call:
if err := stateDB.SetBalanceChangeEntries(
    cmn.NewBalanceChangeEntry(callerAddr, amount, cmn.Sub),
); err != nil {
    return nil, err
}
```

#### Long-term

Architectural fix so the class cannot recur:

```
Adopt the upstream automatic BalanceHandler pattern (cosmos/evm ≥ v0.3.2).
It detects balance changes from Cosmos SDK events automatically,
eliminating the "forgotten SetBalanceChangeEntries" class entirely.
```

#### Adjacent Audit

```
Audit ALL precompile functions that move native tokens for the same missing call.
Pay special attention to: <list adjacent functions worth reviewing>.
```

### 8. Proof-of-Concept Code

Attach the **fully working** PoC. Requirements:

- **Minimal** — no extra helpers or logging.
- **Self-contained** — compiles without external dependencies beyond
  hardhat / forge / cosmjs.
- **Commented** — explain non-obvious preconditions (e.g. why a storage write
  before the precompile call is necessary to trigger the bug).
