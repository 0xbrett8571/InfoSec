Write a **security vulnerability report** following the **guided-tour narrative** designed to pre-emptively answer triage and reviewer questions. This is exactly the right moment to **lock the PoC ↔ report pairing** so reviewers never have to “figure things out themselves”.

Below is **report template**, but **augmented** so that:

* Every **claim** is explicitly backed by a **specific PoC section** or **the source Contract Code**
* Reviewers can **jump back and forth** between report and code
* Severity and impact become **undeniable**, not interpretive

> If a claim cannot be pointed to a specific PoC line, it does not belong in the report.
---

## TITLE
**[Critical]: [Vulnerability Class] in [Contract::function] Allows Direct Asset Extraction**

---

## Triage Dashboard

_A summary with the following structure: {root cause} will cause [a/an] {impact} for {affected party} as {actor} will {attack path}_

| Category | Details |
| :--- | :--- |
| **Severity** | **Critical** |
| **Impact** | Direct Theft / Protocol Insolvency |
| **Confidence** | **Confirmed** (Executable Foundry PoC Attached) |
| **Vector** | [e.g., Logic Error / Missing Access Control / Accounting Bug] |
| **File** | `[src/Contract.sol]` (Line #XX) |
| **Semantic Phase** | [SNAPSHOT/ACCOUNTING/VALIDATION/MUTATION/COMMIT] |

> 🔗 **PoC Reference:** Environment setup and assumptions are enforced in `setUp()` of the Foundry test.

---

## Methodology Validation Checks

_Per [CommandInstruction.md] — ALL must pass before reporting_

| Check | Status | Evidence |
|-------|--------|----------|
| **Reachability** | ✅ | [Can this path execute on-chain without impossible assumptions?] |
| **State Freshness** | ✅ | [Works with current on-chain state, not stale/historical?] |
| **Execution Closure** | ✅ | [All external calls, callbacks, delegatecalls modeled?] |
| **Economic Realism** | ✅ | [Attacker cost, timing, and constraints are feasible?] |

---

## Executive Proof (30-Second Review)

_Write a detailed description of the root cause and impact(s) of this finding._

Example:-
_A malicious, unprivileged external attacker can extract assets from the protocol by abusing a flaw in [Contract::function].
The vulnerability allows the attacker to break a core accounting invariant, resulting in a net transfer of funds from the protocol to the attacker._

**If you read nothing else:**
1. **Attacker starts with:** 0.1 ETH (gas only) - PoC L#8
2. **Attacker does:** One call to `vulnerableFunction()` - PoC L#42
3. **Attacker ends with:** 1000 ETH profit - PoC L#55
4. **Protocol loses:** Exactly 1000 ETH - PoC L#58

**This is not:** A complex multi-step, multi-block, privileged attack.
**This is:** A single function call that breaks basic accounting.

---

## The One-Two Sentence Kill
> An unprivileged attacker can call `Contract::withdraw()` and receive more assets than deposited because the function fails to validate `shares ≤ balanceOf[msg.sender]`.

**Proof:** PoC line #42: `attackerReceives > attackerDeposits` assertion.

---

## Violated Core Invariant
> "Total assets in protocol ≥ Sum of user entitlements"

**Evidence:** In PoC, `assertEq(totalAssetsBefore - totalAssetsAfter, attackerProfit)` fails.

---

## Impact & Economics

_In case it's an attack path: The {affected party} suffers an approximate loss of {value}. [The attacker gains {gain} or loses {loss}]._

_Example: - The stakers suffer a 50% loss during staking. The attacker gains this 50% from stakers. - The protocol suffers a 0.0006 ETH minting fee. The attacker loses their portion of the fee and doesn't gain anything (griefing)._

_In case it's a vulnerability path: The {affected party} [suffers an approximate loss of {value} OR cannot {execute action}]._

_Example: - The users suffer an approximate loss of 0.01% due to precision loss. - The user cannot mint tokens._


### Damage Assessment
* **Direct Loss:** All funds reachable through `[Contract::function]` can be drained.
* **Likelihood:** **High** (No privileges required, fully on-chain, permissionless)

### Attack Economics
| Metric | Value | Proof |
|--------|-------|-------|
| **Attacker Cost** | ~0.05 ETH (gas only) | `gasUsed * gasPrice` in PoC |
| **Minimum Profit** | [X] ETH | `assertGt(attackerProfit, MIN_PROFIT)` |
| **ROI** | [X]% | Calculated in PoC post-exploit |
| **Repeatable?** | Yes, until contract empty | Loop demonstration in PoC |

### Severity Justification Matrix
| Criterion | Required for Critical | This Vulnerability | Proof |
|-----------|----------------------|-------------------|-------|
| Direct fund loss | Yes | $X extractable | L#67: `attackerGain = 1000 ETH` |
| Permissionless | Yes | No roles required | L#12: `attacker = makeAddr("attacker")` |
| Protocol insolvency | Yes | Can drain all assets | L#89: `while(victimBalance > 0)` loop |
| No special conditions | Yes | Always exploitable | L#5: `setUp()` shows normal state |

> 🔗 **PoC Reference:** Balance snapshots taken before/after enforce `victimLoss == attackerGain`

---

## Proof of Concept (Executable, Mainnet-Fork)

### **What This PoC Proves**
* The attacker starts **unprivileged**
* The protocol holds real funds
* After the exploit: Victim balance ↓, Attacker balance ↑, Amounts match

### **60-Second Verification**
1. **Check setup:** Lines #X-#Y → Real mainnet fork with real balances
2. **Skip to assertion:** Line #Z → `assert(victimLoss > 0)`
3. **Trace one transaction:** `vm.startPrank(ATTACKER)` → Single call exploits

```solidity
// [PASTE THE FINAL PoC HERE]
// Key anchors reviewers should notice:
// - setUp(): fork creation + invariant snapshots
// - testExploit(): attacker impersonation
// - Post-conditions: victimLoss == attackerGain
```

### PoC Assertions (Enforced)
| Assertion | Meaning | PoC Line |
|-----------|---------|----------|
| `victimBalanceAfter < victimBalanceBefore` | Protocol loses funds | #55 |
| `attackerBalanceAfter > attackerBalanceBefore` | Attacker profits | #56 |
| `victimLoss == attackerGain` | Net extraction proven | #58 |
| `totalAssetsBefore - totalAssetsAfter == attackerProfit` | Invariant violation | #60 |

> 🔗 **PoC Anchors:**
> - `testExploit()` L#24-31: Attacker setup with real funds
> - `testExploit()` L#42-45: Single exploit transaction
> - `testExploit()` L#51-55: Profit verification assertions

---

## Root Cause

_In case it’s a mistake in the code: In {link to code} the {root cause}

Example: - In stake.sol:551 there is a missing check on transfer function - In lp.sol:12 the fee calculation does division before multiplication which will revert the transaction on lp.sol:18

In case it’s a conceptual mistake: The choice to {design choice} is a mistake as {root cause}

Example: - The choice to use Uniswap as an oracle is a mistake as the price can be manipulated - The choice to depend on Protocol X for admin calls is a mistake as it will cause any call to revert_
### Root Cause Category
- [ ] Reentrancy
- [ ] Access Control
- [ ] Oracle Manipulation
- [ ] Integer Overflow/Underflow
- [ ] Logic Error
- [ ] Initialization
- [ ] Storage Collision
- [ ] Signature Replay
- [ ] Other: ___

### Semantic Phase Analysis
_Per [audit-workflow2.md, Step 2.2] — Where in the execution flow does the bug occur?_

| Phase | What Happens | Status |
|-------|--------------|--------|
| SNAPSHOT | [State reads] | ✅ / ❌ |
| ACCOUNTING | [Time/oracle calculations] | ✅ / ❌ |
| VALIDATION | [Checks/requires] | ✅ / ❌ |
| MUTATION | [Balance updates] | ✅ / ❌ |
| COMMIT | [Storage writes] | ✅ / ❌ |

**Bug Location:** [PHASE] — [Brief description]
### Vulnerable Code
`[src/Contract.sol]` (Line #XX)
```solidity
function vulnerableFunction(...) external {
    // [!] Missing invariant / access control / validation
    ...
}
```

### Behavior Comparison
| Expected (per spec) | Actual (PoC demonstrates) | PoC Proof |
|---------------------|---------------------------|-----------|
| `withdraw(100)` → User gets ≤100 worth | `withdraw(100)` → User gets 150 worth | L#42: `attackerReceives > amount` |
| Total assets decrease by ≤100 | Total assets decrease by 150 | L#60: `totalAssetsDelta > amount` |
| Other users' shares unaffected | Other users' shares diluted | L#65: `shareValueDecreases` |

### Proof of Defect
1. **Code contradiction:** Function `deposit()` enforces invariant at L#45, but `vulnerableFunction()` doesn't at L#78
2. **Documentation mismatch:** NatSpec @dev says "prevents overdrafts" but code doesn't
3. **Test suite gap:** Existing tests check this invariant for other functions but not this one

### Distinction from Related Vectors
| Possible Misclassification | Technical Rationale | PoC Evidence |
|----------------------------|----------------|--------------|
| "Frontrunning opportunity" | No race condition needed; works in isolation | Single-block exploit |
| "Oracle manipulation" | No external price feeds used | Pure internal accounting |
| "Governance attack" | No privileges required | Attacker is fresh EOA |

### Similar Historical Exploits
_Per [audit-workflow1.md, Step 5.1b] — Reference known exploits if pattern matches_

| Historical Exploit | Similarity | Difference |
|--------------------|------------|------------|
| [Exploit Name (Year)] | [What's similar] | [What's different] |
| [Exploit Name (Year)] | [What's similar] | [What's different] |

> **Note:** If this matches a known vulnerability class, explicitly state it. Triagers respect historical awareness.

---

## Recommended Mitigation

Enforce the missing invariant explicitly before allowing state transitions:
```diff
function vulnerableFunction(...) external {
-    // implicit assumption
+    require(userBalance >= amount, "Insufficient balance");
     ...
}
```

### **Defense-in-Depth**
* Add invariant checks after state updates
* Use internal accounting assertions
* Add regression tests mirroring the attached PoC

> 🔗 **PoC Reference:** The attached test can be converted into a permanent regression test to prevent re-introduction.

---

## Validation Checklist

### **PoC ↔ Report Pairing**
- [ ] Every claim references a PoC line or assertion
- [ ] No speculative language ("could", "might")
- [ ] All impact claims are asserted in code

### **Triage-Proofing**
- [ ] Explicit attacker profit or permanent value loss
- [ ] Clear invariant statement
- [ ] "Not a design choice" explanation
- [ ] Platform-specific severity mapping included

### **Reviewer Experience**
- [ ] Reviewer can read report + PoC side-by-side
- [ ] No mental math required
- [ ] No interpretation required

### **Technical Validation**
- [ ] **Single-transaction exploit:** PoC uses only `vm.startPrank()` once
- [ ] **No setup assumptions:** Attacker starts with only gas money
- [ ] **Real fork:** Uses `vm.createSelectFork(MAINNET, BLOCK)`
- [ ] **Failed fix test:** Added a test showing the suggested fix works

### **Timing Constraints**
| Constraint | Requirement | PoC Proof |
|------------|-------------|-----------|
| **Blocks required** | [Single block / Multiple] | [PoC line or explanation] |
| **Time window** | [Any time / Specific window] | [PoC line or explanation] |
| **Ordering dependency** | [None / Frontrun required] | [PoC line or explanation] |
| **External conditions** | [None / Oracle state / etc.] | [PoC line or explanation] |

---

## Severity Platform Mapping

**Immunefi:**
> Critical — Direct loss of user or protocol funds via permissionless exploit

**Code4rena:**
> High/Critical — Theft or permanent loss of funds due to broken accounting invariant

**Hats Finance:**
> Critical — Loss of funds without privilege escalation or external dependencies

---

## Appendix: Impact Type Templates

_Use the appropriate template based on impact type:_

### For Denial of Service (DoS)
```markdown
**Impact Type:** Denial of Service
**Victim cannot:** [Execute what action?]
**Duration:** [Permanent / Temporary (X blocks/hours)]
**Recovery:** [None possible / Admin intervention / Self-recovery after X]
**Cost to attacker:** [X ETH]
**Damage to victim:** [Lost opportunity / Stuck funds / Operational impact]
```

### For Griefing (Attacker Loses Too)
```markdown
**Impact Type:** Griefing
**Attacker cost:** [X ETH]
**Victim loss:** [Y ETH or impact]
**Attacker gain:** [None / Indirect benefit]
**Motivation:** [Competitor / Malice / Market manipulation]
```

### For Privilege Escalation
```markdown
**Impact Type:** Privilege Escalation
**Attacker gains:** [Role / Capability]
**Can then:** [List of new attack surfaces enabled]
**Leads to:** [Ultimate downstream impact]
```

### For Information Disclosure
```markdown
**Impact Type:** Information Disclosure
**Leaked data:** [What is exposed?]
**Who can access:** [Anyone / Specific conditions]
**Downstream risk:** [What can attacker do with this info?]
```

---
Save the report in a Markdown file.