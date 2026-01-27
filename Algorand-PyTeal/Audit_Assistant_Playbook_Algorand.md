# Audit Assistant Playbook – Algorand/PyTeal Edition

> A structured conversation framework for Algorand smart contract security audits.

---

## 1. Introduction

### Purpose
This playbook structures how security auditors interact with LLMs when auditing Algorand smart contracts. It provides:
- **Conversation templates** for each audit phase
- **Algorand-specific prompts** for transaction field validation
- **Quality gates** to ensure thorough analysis

### Related Documents
| Document | Purpose |
|----------|---------|
| [Algorand-Audit-Methodology.md](./Algorand-Audit-Methodology.md) | Technical methodology |
| [CommandInstruction-Algorand.md](./CommandInstruction-Algorand.md) | System prompt |

---

## 2. Conversation Structure

### Phase 1: Scope Definition
```markdown
## Audit Scope

**Contract Type:** [Application / Smart Signature / Both]
**Language:** [PyTeal / TEAL / Beaker / ARC4]
**TEAL Version:** [e.g., v8, v9, v10]

**Contract(s):**
- Approval Program: [filename]
- Clear State Program: [filename]
- Smart Signature(s): [if any]

**Focus Areas:**
- [ ] Transaction field validation (RekeyTo, Close, Fee)
- [ ] Group transaction security
- [ ] Access control (Update/Delete)
- [ ] Inner transaction safety
- [ ] Asset handling (opt-in, transfers)

**Out of Scope:**
- [ ] [List any exclusions]

**External Integrations:**
- Oracles: [if any]
- Other Applications: [App IDs]
- ASAs: [Asset IDs]
```

### Phase 2: Entry Point Mapping
```markdown
## Entry Point Analysis Request

Please analyze this Algorand contract and identify ALL code paths:

**For Applications:**
1. NoOp handlers (main functions)
2. OptIn handler
3. CloseOut handler
4. UpdateApplication handler ← Check access control!
5. DeleteApplication handler ← Check access control!
6. Clear State program

**For Smart Signatures:**
1. All approval conditions
2. Transaction types handled

For each path, identify:
- Transaction field validations (or lack thereof)
- Group transaction assumptions
- State changes (global/local)
- Inner transactions

**Contract Code:**
[Paste contract here]
```

### Phase 3: Transaction Field Analysis
```markdown
## Transaction Field Security Analysis

For ALL transaction handling code:

**Critical Fields to Validate:**
1. **RekeyTo**: Must equal Global.zero_address()
2. **CloseRemainderTo**: Must equal Global.zero_address()
3. **AssetCloseTo**: Must equal Global.zero_address()
4. **Fee** (Smart Sigs): Must equal Global.min_txn_fee() or Int(0)

**For Each Field:**
- Is validation present?
- Is validation correct?
- Can it be bypassed?

**Contract Code:**
[Paste contract here]
```

### Phase 4: Group Transaction Analysis
```markdown
## Group Transaction Security Analysis

For code using Gtxn[] (group transactions):

1. **Group Size Validation:**
   - Is Global.group_size() checked?
   - What's the expected size?
   - Can attacker add extra transactions?

2. **Transaction Index Usage:**
   - Are indices hardcoded or dynamic?
   - Could indices be manipulated?

3. **OnComplete Validation:**
   - Is Gtxn[i].on_completion() checked?
   - Could ClearState be substituted for NoOp?

4. **Asset/App ID Validation:**
   - Is Gtxn[i].xfer_asset() validated?
   - Is Gtxn[i].application_id() validated?

**Contract Code:**
[Paste contract here]
```

### Phase 5: Deep Dive Prompts

#### Access Control Analysis
```markdown
## Access Control Analysis

For UpdateApplication and DeleteApplication handlers:

1. **Update Protection:**
   - Is sender validated against creator/admin?
   - Can anyone call UpdateApplication?

2. **Delete Protection:**
   - Is sender validated against creator/admin?
   - Can anyone delete the application?

3. **Admin Pattern:**
   - Is admin address stored in global state?
   - Can admin be changed? By whom?

**Look for patterns like:**
```python
# VULNERABLE
Cond(
    [Txn.on_completion() == OnComplete.UpdateApplication, Return(Int(1))],
)

# SECURE
Cond(
    [Txn.on_completion() == OnComplete.UpdateApplication, 
     Return(Txn.sender() == Global.creator_address())],
)
```
```

#### Inner Transaction Analysis
```markdown
## Inner Transaction Security Analysis

For all InnerTxnBuilder usage:

1. **Fee Field:**
   - Is TxnField.fee explicitly set to Int(0)?
   - If not, each inner tx drains app balance!

2. **Close Fields:**
   - Is CloseRemainderTo set or validated?
   - Is AssetCloseTo set or validated?

3. **Asset Opt-In Considerations:**
   - Using push or pull pattern?
   - What if recipient not opted in?

**Contract Code:**
[Paste contract here]
```

---

## 3. Finding Documentation

### Finding Template
```markdown
## [SEVERITY] Finding Title

### Summary
[One paragraph describing the issue]

### Vulnerability Details

**Location:** `contract.py` (lines X-Y)
**Handler:** [NoOp / OptIn / UpdateApplication / etc.]

**Root Cause:**
[Technical explanation]

**Vulnerable Code:**
```python
# PyTeal code showing vulnerability
```

### Impact
[What can an attacker achieve?]

### Attack Scenario
1. Attacker constructs transaction group with:
   - [Transaction details]
2. Missing validation allows:
   - [Exploit step]
3. Result: [Impact]

### Proof of Concept
```python
# Transaction construction showing exploit
from algosdk.future import transaction

atc = AtomicTransactionComposer()
# Exploit transactions...
```

### Recommended Mitigation
```python
# Secure PyTeal code
```

### References
- Tealer Detector: [detector name]
- Trail of Bits Pattern: [reference]
```

### Severity Guidelines

| Severity | Criteria | Examples |
|----------|----------|----------|
| **CRITICAL** | Account takeover, full drain | Missing RekeyTo check, unvalidated CloseRemainderTo |
| **HIGH** | Significant fund loss, app takeover | Unprotected Update/Delete, asset drain |
| **MEDIUM** | Limited loss, DoS | Group size manipulation, fee drain |
| **LOW** | Minor issues | Gas inefficiency, style issues |
| **INFO** | Suggestions | Documentation, best practices |

---

## 4. Quality Gates

### Before Reporting a Finding
- [ ] **Reachable**: Code path can be executed
- [ ] **Triggerable**: Can construct valid transaction
- [ ] **Impactful**: Clear negative outcome
- [ ] **Not Mitigated**: No existing protection
- [ ] **Reproducible**: Attack steps are clear

### Transaction Field Specific Gates
- [ ] **RekeyTo**: Checked for ALL transaction acceptance
- [ ] **CloseRemainderTo**: Checked for payment handling
- [ ] **AssetCloseTo**: Checked for asset transfer handling
- [ ] **Fee**: Checked for smart signatures

---

## 5. Tool Integration

### Tealer Commands
```bash
# Full analysis
tealer detect --program approval.teal

# Specific detector
tealer detect --program approval.teal --detectors unprotected-rekey

# Compile PyTeal first
python contract.py > approval.teal
tealer detect --program approval.teal
```

### Algorand Sandbox
```bash
# Start local network
./sandbox up dev

# Create application
goal app create \
    --creator $ADDR \
    --approval-prog approval.teal \
    --clear-prog clear.teal \
    --global-byteslices 2 \
    --global-ints 2 \
    --local-byteslices 1 \
    --local-ints 1

# Call application
goal app call \
    --app-id $APP_ID \
    --from $ADDR \
    --app-arg "str:function_name"
```

---

## 6. Example Conversations

### Example 1: Missing RekeyTo Check
```
User: Please review this smart signature for security issues:

def escrow():
    return And(
        Txn.type_enum() == TxnType.Payment,
        Txn.amount() <= Int(1000000),
        Txn.receiver() == Addr("RECIPIENT_ADDRESS"),
    )