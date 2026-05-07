---
title: TVM (TON) Smart Contacts
description:
---

# Bug Bounty Report Guide

The report template below covers vulnerabilities in TAC-related FunC / Tact contracts deployed on TON.
Tools: Blueprint, toncli, tonutils-go, @ton/ton (JS SDK).

## Report Template

```
SECURITY VULNERABILITY REPORT
<Protocol> — <Severity> <Class> in <ContractName>
Date: YYYY-MM-DD
Status: <THEORETICAL | TESTNET CONFIRMED | MAINNET CONFIRMED>


===============================
EXECUTIVE SUMMARY
===============================

<op handler broken, supply/TON at risk, repeatability>


===============================
VULNERABILITY DETAILS
===============================

Contract:     <file.fc / .tact>, <op handler>, line <X>
Deployed:     EQ<address>
Class:        <category>

Root cause:
  <one sentence — missing check / wrong branch condition>

Vulnerable snippet:
  <code>

Fixed version:
  <code>

TON-specific notes:
  <bounce handling, gas forwarding, replay guard — if relevant>

Prerequisites:
  <conditions>


===============================
EXPLOITATION STEPS
===============================

Message body:
  op:       <hex>
  payload:  <fields>

1. <step>
2. <step>
...

Blueprint test / script:
  <abbreviated code>


===============================
POC RESULTS
===============================

Contract:    EQ<address>
Exploit TX:  <tonviewer link>

Balance table:
  <address>   before → after   delta

Scaling:
  <calls → minted / drained → gas>


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
Long-term:   <architectural / tooling>
Audit:       <adjacent ops / handlers>


===============================
PROOF-OF-CONCEPT CODE
===============================

<full Blueprint test or @ton/ton script>


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

Useful classes:
`Unprotected Internal Message`, `Bounce Exploit`, `Storage Drain`,
`Replay Attack (no op-guard)`, `Incorrect Fees / Forward Gas`,
`Cell Overflow`, `Tact Inheritance Access Control`.

### 2. Executive Summary

Same four questions:

1. **What is broken?** — which op-code handler / recv_internal branch.
2. **What does the attacker gain?** — TON drained / token minted / ownership.
3. **How much?** — contract balance or jetton supply at risk.
4. **Is it repeatable?** — per-message / once / needs timing.

**Example:**

```
The jetton minter's recv_internal handler does not verify the sender against
the admin address before processing op::mint. Any wallet can send a mint
message and create arbitrary tokens, inflating the supply without limit.

  Contract balance at risk:  0 TON (no drain)
  Token supply at risk:      unlimited inflation
  Attack cost:               ~0.05 TON (gas)
```

### 3. Vulnerability Details

#### 3.1 Affected Contract

```
Contract:   contracts/JettonMinter.fc  (or .tact)
Handler:    recv_internal — op::mint branch (line ~47)
Deployed:   EQ<address>  (mainnet / testnet)
```

Provide a Tonviewer / Tonscan link if the contract is verified.

#### 3.2 Root Cause

> The `op::mint` branch reads `sender_address` from the message slice but
> never calls `throw_unless(73, equal_slices(sender_address, storage::admin))`
> before executing the mint logic.

#### 3.3 Vulnerable Code Snippet

```func
;; JettonMinter.fc ~line 47 — VULNERABLE
if (op == op::mint) {
    ;; ← missing: throw_unless(73, equal_slices(sender, admin));
    slice to_address = in_msg_body~load_msg_addr();
    int  amount      = in_msg_body~load_coins();
    mint_tokens(to_address, amount);
    ...
}
```

#### 3.4 Fixed Version

```func
if (op == op::mint) {
    throw_unless(73, equal_slices(sender, admin)); ;; ← added
    slice to_address = in_msg_body~load_msg_addr();
    int  amount      = in_msg_body~load_coins();
    mint_tokens(to_address, amount);
    ...
}
```

#### 3.5 TON-specific Considerations

- **Bounce messages:** does the contract handle `op::excesses` / bounced
  messages safely? A missing bounce handler can leave funds locked.
- **Forward gas:** does the contract forward enough gas for sub-messages?
  Under-forwarding silently fails without reverting the parent tx.
- **Storage fees:** contracts with no incoming messages will be frozen and
  then deleted. Does the attacker benefit from this?
- **Replay:** TON has no global nonce — contracts must implement their own
  seqno or use the `valid_until` trick.

### 4. Exploitation Steps

For each step, include the message body layout (op + payload):

```
1. Build a mint message:
     op:         0x642b7d07  (op::mint)
     to_address: attacker_wallet
     amount:     1_000_000_000_000 (1 000 000 jettons, 6 decimals)

2. Send from any wallet with 0.1 TON attached (covers gas + forward).

3. Check attacker's jetton wallet balance — should increase by 1 000 000.

4. Repeat for unlimited supply inflation.
```

Blueprint test skeleton:

```typescript
it("mints without admin auth", async () => {
    const attacker = await blockchain.treasury("attacker");
    await minter.sendMint(attacker.getSender(), {
        toAddress: attacker.address,
        amount:    toNano("1000000"),
        value:     toNano("0.1"),
    });
    const balance = await getJettonBalance(attacker.address);
    expect(balance).toEqual(toNano("1000000")); // should FAIL if patched
});
```

### 5. PoC Results

Required:
- Contract address (`EQ...`).
- Transaction hash (lt + hash, or Tonviewer link).
- Message trace (Tonviewer shows the full message tree).
- Jetton balance / TON balance before and after.

**Example:**

```
CONTRACT              BALANCE BEFORE    BALANCE AFTER    DELTA
--------              --------------    -------------    -----
JettonMinter          100 000 supply    1 100 000        +1 000 000 minted
Attacker jetton wlt   0                 1 000 000        +1 000 000

Exploit TX: https://tonviewer.com/transaction/<hash>
Gas spent:  0.048 TON
```

#### 5.1 Scaling Economics

```
Calls    Amount per call    Total minted    Gas cost
-----    ---------------    ------------    --------
1        1 000 000          1 000 000       0.05 TON
100      1 000 000          100 000 000     5.0 TON
```


### 6. Impact Assessment

```
Severity: CRITICAL

  1. Unlimited token inflation — total supply can be minted to any address.
  2. No admin role required — any wallet can exploit.
  3. Per-message — repeatable indefinitely.
  4. DEX liquidity pools backed by this jetton become worthless.
```

TON-specific severity boosters:
- contract holds significant TON balance (storage drain possible),
- contract is a bridge or DEX vault,
- no upgrade / pause mechanism exists.

### 7. Recommended Fix

#### Immediate

```
Add sender verification as the first statement in the op::mint branch:
  throw_unless(73, equal_slices(sender, storage::admin));
```

#### Long-term

```
Use Tact's @internal access modifier or a dedicated access-control library.
Add Blueprint unit tests that assert every privileged op reverts for non-admin.
```

#### Adjacent Audit

```
Review all other privileged ops in recv_internal:
  op::change_admin, op::burn_notification, op::set_content.
Verify bounce handler is implemented for all outgoing messages carrying value.
```

### 8. Proof-of-Concept Code

Provide a **Blueprint** test or a minimal **@ton/ton** script:

```typescript
// test/JettonMinterExploit.spec.ts
import { Blockchain, SandboxContract } from "@ton/sandbox";
import { JettonMinter } from "../wrappers/JettonMinter";
import { toNano } from "@ton/ton";

describe("Exploit: unrestricted mint", () => {
    let blockchain: Blockchain;
    let minter: SandboxContract<JettonMinter>;

    beforeEach(async () => {
        blockchain = await Blockchain.create();
        // deploy with 1 TON initial balance
        minter = ...; // standard deploy
    });

    it("mints 1M tokens without admin", async () => {
        const attacker = await blockchain.treasury("attacker");
        const result = await minter.sendMint(attacker.getSender(), {
            toAddress: attacker.address,
            amount: toNano("1000000"),
            value: toNano("0.1"),
        });
        expect(result.transactions).toHaveTransaction({ success: true });
        // if this passes — the bug is confirmed
    });
});
```
