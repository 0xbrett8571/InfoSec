# Comprehensive Algorand/PyTeal Smart Contract Audit Methodology

> **Integration Note:** This file contains the Algorand-specific audit methodology.
> For LLM conversation structure, see `Audit_Assistant_Playbook_Algorand.md`.
> For the system prompt, see `CommandInstruction-Algorand.md`.

---

## **Phase 1: Entry Point Identification & Scope Definition**

### **Step 1.0: Time-Boxing Strategy**
Prevent analysis paralysis with structured time allocation:

```markdown
**Round 1 (40% of time): Quick Triage ALL Code Paths**
- 5 minutes max per handler/branch
- Identify transaction field validations
- Note group transaction usage
- Goal: Map the attack surface

**Round 2 (40% of time): Deep Dive TOP 5 Priority Areas**
- Transaction field validation completeness
- Group transaction security
- Access control on Update/Delete
- Goal: Find critical vulnerabilities

**Round 3 (20% of time): Cross-Contract & Edge Cases**
- Inner transaction flows
- Asset opt-in scenarios
- Clear state program impact
- Goal: Catch composition bugs
```

---

### **Step 1.1: Identify Algorand Contract Entry Points**

**For Applications (Approval Program):**
- [ ] Creation handler (`Txn.application_id() == Int(0)`)
- [ ] NoOp handlers (main functions)
- [ ] OptIn handler
- [ ] CloseOut handler
- [ ] UpdateApplication handler ← **CRITICAL: Check access control**
- [ ] DeleteApplication handler ← **CRITICAL: Check access control**

**For Clear State Program:**
- [ ] Always succeeds - cannot reject
- [ ] Minimize state cleanup impact

**For Smart Signatures:**
- [ ] All approval conditions
- [ ] Transaction types accepted

**Commands to identify entry points:**
```bash
# Find OnComplete handlers
grep -E "OnComplete\." --include="*.py" .
grep -E "on_completion" --include="*.teal" .

# Find transaction type checks
grep -E "type_enum" --include="*.py" .
grep -E "TypeEnum" --include="*.teal" .

# Find group transaction access
grep -E "Gtxn\[" --include="*.py" .
```

### **Step 1.2: Quick Protocol Understanding**
```markdown
## Algorand-Specific Protocol Context

**Contract Type**: [Application / Smart Signature / Both]
**Language**: [PyTeal / TEAL / Beaker / ARC4]
**TEAL Version**: [v8 / v9 / v10]

**State Schema:**
- Global Bytes: [count]
- Global Ints: [count]
- Local Bytes: [count]
- Local Ints: [count]

**External Dependencies:**
- Other Apps: [App IDs if any]
- ASAs: [Asset IDs if any]
- Oracles: [if any]

**Inner Transactions:**
- Types used: [Payment / AssetTransfer / AppCall / etc.]
- Fee strategy: [Fee pooling / App pays]
```

### **Step 1.3: Prioritization Matrix (Algorand Edition)**
```markdown
## Priority 1 (Attack Immediately)
- [ ] ALL transaction field validations (RekeyTo, Close, Fee)
- [ ] UpdateApplication and DeleteApplication access control
- [ ] Group transaction size validation
- [ ] Inner transaction fee settings
- [ ] Asset ID verification in swaps/transfers

## Priority 2 (Attack After)
- [ ] OptIn/CloseOut handlers
- [ ] Local state manipulation
- [ ] Lease field for replay protection
- [ ] Clear state program impact

## Priority 3 (Check Later)
- [ ] Opcode budget optimization
- [ ] State schema efficiency
- [ ] Code style and readability
```

### **Step 1.4: Mandatory Validation Checks**
| Check | Question | Algorand Considerations |
|-------|----------|------------------------|
| **Reachability** | Can this code path execute? | Valid OnComplete, correct group position |
| **Transaction Context** | Valid transaction structure? | Group size, indices, types |
| **Economic Realism** | Fees and balances feasible? | Min balance, fee pooling |
| **State Requirements** | Opt-in required? | Local state access needs opt-in |

---

## **Phase 2: Transaction Field Security**

### **Step 2.1: Critical Fields Checklist**

Every contract MUST validate these fields to prevent account takeover and fund drain:

| Field | Risk | Required Validation |
|-------|------|---------------------|
| **RekeyTo** | Account takeover | `Txn.rekey_to() == Global.zero_address()` |
| **CloseRemainderTo** | Full ALGO drain | `Txn.close_remainder_to() == Global.zero_address()` |
| **AssetCloseTo** | Full ASA drain | `Txn.asset_close_to() == Global.zero_address()` |
| **Fee** (Smart Sig) | Balance drain | `Txn.fee() == Global.min_txn_fee()` or `Int(0)` |

### **Step 2.2: Transaction Field Validation Patterns**

```python
# VULNERABLE: Missing all critical checks
def vulnerable_escrow():
    return And(
        Txn.type_enum() == TxnType.Payment,
        Txn.amount() <= Int(1000000),
        Txn.receiver() == authorized_receiver,
    )

# SECURE: Complete validation
def secure_escrow():
    return And(
        # Transaction type
        Txn.type_enum() == TxnType.Payment,
        
        # Business logic
        Txn.amount() <= Int(1000000),
        Txn.receiver() == authorized_receiver,
        
        # CRITICAL SECURITY CHECKS
        Txn.rekey_to() == Global.zero_address(),        # Prevent rekey
        Txn.close_remainder_to() == Global.zero_address(), # Prevent drain
        Txn.fee() == Global.min_txn_fee(),              # Prevent fee drain
    )
```

### **Step 2.3: Application Field Validation**

```python
# For applications, add these checks to approval program
def approval_program():
    # Security checks that apply to ALL handlers
    security_checks = And(
        Txn.rekey_to() == Global.zero_address(),
        Txn.close_remainder_to() == Global.zero_address(),
    )
    
    # Main program logic
    program = Cond(
        [Txn.application_id() == Int(0), on_creation],
        [Txn.on_completion() == OnComplete.NoOp, on_call],
        # ... other handlers
    )
    
    return And(security_checks, program)
```

---

## **Phase 3: Group Transaction Security**

### **Step 3.1: Group Size Validation**

```python
# VULNERABLE: No group size check
def vulnerable_swap():
    return And(
        Gtxn[0].type_enum() == TxnType.Payment,
        Gtxn[1].type_enum() == TxnType.AssetTransfer,
        # Attacker can add Gtxn[2], Gtxn[3], etc.!
    )

# SECURE: Explicit group size
def secure_swap():
    return And(
        Global.group_size() == Int(2),  # CRITICAL: Exact size
        Gtxn[0].type_enum() == TxnType.Payment,
        Gtxn[1].type_enum() == TxnType.AssetTransfer,
    )
```

### **Step 3.2: OnComplete Validation**

```python
# VULNERABLE: Only checks transaction type
def vulnerable_group():
    return And(
        Gtxn[0].type_enum() == TxnType.ApplicationCall,
        # Missing: Gtxn[0].on_completion() check!
        # Attacker can use ClearState instead of NoOp
    )

# SECURE: Check both type AND OnComplete
def secure_group():
    return And(
        Gtxn[0].type_enum() == TxnType.ApplicationCall,
        Gtxn[0].on_completion() == OnComplete.NoOp,  # Explicit check
    )
```

### **Step 3.3: Asset ID Verification**

```python
# VULNERABLE: No asset ID check
def vulnerable_purchase():
    return And(
        Gtxn[0].type_enum() == TxnType.AssetTransfer,
        Gtxn[0].asset_amount() >= required_amount,
        # Missing: Which asset? Attacker sends worthless token!
    )

# SECURE: Validate specific asset
USDC_ID = Int(12345678)  # Or from global state

def secure_purchase():
    return And(
        Gtxn[0].type_enum() == TxnType.AssetTransfer,
        Gtxn[0].xfer_asset() == USDC_ID,  # CRITICAL: Verify asset
        Gtxn[0].asset_amount() >= required_amount,
    )
```

---

## **Phase 4: Access Control Security**

### **Step 4.1: Update/Delete Application Protection**

```python
# VULNERABLE: Anyone can update or delete
def vulnerable_approval():
    return Cond(
        [Txn.application_id() == Int(0), on_creation],
        [Txn.on_completion() == OnComplete.UpdateApplication, Return(Int(1))],  # WRONG!
        [Txn.on_completion() == OnComplete.DeleteApplication, Return(Int(1))],  # WRONG!
        [Txn.on_completion() == OnComplete.NoOp, on_call],
    )

# SECURE: Only creator/admin can update or delete
def secure_approval():
    is_creator = Txn.sender() == Global.creator_address()
    
    return Cond(
        [Txn.application_id() == Int(0), on_creation],
        [Txn.on_completion() == OnComplete.UpdateApplication, is_creator],
        [Txn.on_completion() == OnComplete.DeleteApplication, is_creator],
        [Txn.on_completion() == OnComplete.NoOp, on_call],
    )

# BETTER: Disable updates entirely if not needed
def immutable_approval():
    return Cond(
        [Txn.application_id() == Int(0), on_creation],
        [Txn.on_completion() == OnComplete.UpdateApplication, Return(Int(0))],  # Reject
        [Txn.on_completion() == OnComplete.DeleteApplication, Return(Int(0))],  # Reject
        [Txn.on_completion() == OnComplete.NoOp, on_call],
    )
```

### **Step 4.2: Admin Pattern**

```python
# Flexible admin pattern with transfer capability
ADMIN_KEY = Bytes("admin")

def on_creation():
    return Seq([
        # Set creator as initial admin
        App.globalPut(ADMIN_KEY, Txn.sender()),
        Return(Int(1)),
    ])

def is_admin():
    return Txn.sender() == App.globalGet(ADMIN_KEY)

def transfer_admin():
    new_admin = Txn.application_args[1]
    return Seq([
        Assert(is_admin()),
        Assert(Len(new_admin) == Int(32)),  # Valid address length
        App.globalPut(ADMIN_KEY, new_admin),
        Return(Int(1)),
    ])
```

---

## **Phase 5: Inner Transaction Security**

### **Step 5.1: Fee Field in Inner Transactions**

```python
# VULNERABLE: Missing fee field
def vulnerable_inner_payment():
    return Seq([
        InnerTxnBuilder.Begin(),
        InnerTxnBuilder.SetFields({
            TxnField.type_enum: TxnType.Payment,
            TxnField.receiver: recipient,
            TxnField.amount: amount,
            # Missing: TxnField.fee: Int(0)
            # Each call drains app balance!
        }),
        InnerTxnBuilder.Submit(),
    ])

# SECURE: Explicit zero fee
def secure_inner_payment():
    return Seq([
        InnerTxnBuilder.Begin(),
        InnerTxnBuilder.SetFields({
            TxnField.type_enum: TxnType.Payment,
            TxnField.receiver: recipient,
            TxnField.amount: amount,
            TxnField.fee: Int(0),  # CRITICAL: Use fee pooling
        }),
        InnerTxnBuilder.Submit(),
    ])
```

### **Step 5.2: Inner Transaction Close Fields**

```python
# VULNERABLE: CloseRemainderTo not set (could default unexpectedly)
InnerTxnBuilder.SetFields({
    TxnField.type_enum: TxnType.Payment,
    TxnField.receiver: recipient,
    TxnField.amount: amount,
    TxnField.fee: Int(0),
    # Should explicitly set close fields to zero
})

# SECURE: Explicitly set close fields
InnerTxnBuilder.SetFields({
    TxnField.type_enum: TxnType.Payment,
    TxnField.receiver: recipient,
    TxnField.amount: amount,
    TxnField.fee: Int(0),
    TxnField.close_remainder_to: Global.zero_address(),  # Explicit
})
```

### **Step 5.3: Asset Opt-In Considerations**

```python
# VULNERABLE: Push pattern - fails if not opted in
def vulnerable_distribute():
    # If ANY recipient not opted in, entire batch fails
    return For(i, Int(0), num_recipients, Int(1)).Do(
        Seq([
            InnerTxnBuilder.Begin(),
            InnerTxnBuilder.SetFields({
                TxnField.type_enum: TxnType.AssetTransfer,
                TxnField.xfer_asset: asset_id,
                TxnField.asset_receiver: recipients[i],  # May not be opted in!
                TxnField.asset_amount: amounts[i],
                TxnField.fee: Int(0),
            }),
            InnerTxnBuilder.Submit(),
        ])
    )

# SECURE: Pull pattern - users claim
def secure_claim():
    # User initiates, must be opted in
    claimable = App.localGet(Txn.sender(), Bytes("claimable"))
    return Seq([
        Assert(claimable > Int(0)),
        InnerTxnBuilder.Begin(),
        InnerTxnBuilder.SetFields({
            TxnField.type_enum: TxnType.AssetTransfer,
            TxnField.xfer_asset: asset_id,
            TxnField.asset_receiver: Txn.sender(),  # User is caller, must be opted in
            TxnField.asset_amount: claimable,
            TxnField.fee: Int(0),
        }),
        InnerTxnBuilder.Submit(),
        App.localPut(Txn.sender(), Bytes("claimable"), Int(0)),
    ])
```

---

## **Phase 6: Attack Simulation**

### **Step 6.1: Known Algorand Exploit Patterns**

```markdown
## Historical Exploit Database - Algorand

**Transaction Field Exploits:**
- [ ] **Rekeying Attack**: Missing rekey_to validation → account takeover
- [ ] **CloseRemainderTo Drain**: Missing close validation → full ALGO drain
- [ ] **AssetCloseTo Drain**: Missing asset close validation → full ASA drain
- [ ] **Fee Drain**: Smart signature without fee check → balance drain

**Group Transaction Exploits:**
- [ ] **Group Size Manipulation**: Missing size check → repeated execution
- [ ] **OnComplete Bypass**: Type check without OnComplete → ClearState execution
- [ ] **Asset ID Confusion**: Missing asset verification → worthless token accepted

**Application Exploits:**
- [ ] **Unprotected Update**: Anyone can change application code
- [ ] **Unprotected Delete**: Anyone can destroy application
- [ ] **Inner Tx Fee Drain**: Missing fee:0 → app balance drained

**Timing/Replay Exploits:**
- [ ] **Time-Based Replay**: Missing Lease field → repeated execution
```

### **Step 6.2: Attack Scenario Templates**

#### Rekeying Attack
```markdown
1. Attacker constructs transaction with:
   - Valid business logic (amount, receiver, etc.)
   - RekeyTo = attacker's address

2. If contract doesn't check rekey_to:
   - Transaction executes successfully
   - Account is now controlled by attacker

3. Result: Complete account takeover
```

#### Group Size Manipulation
```markdown
1. Contract expects 2 transactions in group
2. Attacker creates group with 10 transactions:
   - Txn[0]: Valid payment
   - Txn[1]: Valid app call
   - Txn[2-9]: 8 additional app calls

3. If contract doesn't check group_size:
   - All 9 app calls execute
   - Operation performed 9x instead of 1x

4. Result: Repeated execution of privileged operation
```

### **Step 6.3: Edge Case Testing**

```python
# Edge cases to test

# 1. Zero amounts
Assert(Txn.amount() > Int(0))  # Reject zero?

# 2. Maximum values
MAX_UINT64 = Int(2**64 - 1)

# 3. Empty accounts (no ASA opt-in)
# Test push pattern with non-opted-in recipients

# 4. Group size limits (max 16)
Assert(Global.group_size() <= Int(16))

# 5. Lease field for replay protection
Assert(Txn.lease() == expected_lease)

# 6. First/Last valid bounds
Assert(Txn.first_valid() >= Global.round())
```

---

## **Phase 7: Enhanced Detection Patterns (ClaudeSkills Integration)**

> These patterns are sourced from Trail of Bits' building-secure-contracts repository

### **Pattern A1: Rekeying Attack ⚠️ CRITICAL**
**Description**: Missing RekeyTo validation allows account authorization transfer.

```python
# VULNERABLE: No RekeyTo check
def escrow():
    return And(
        Txn.type_enum() == TxnType.Payment,
        Txn.amount() <= max_amount,
    )

# SECURE: Validate RekeyTo
def escrow():
    return And(
        Txn.type_enum() == TxnType.Payment,
        Txn.amount() <= max_amount,
        Txn.rekey_to() == Global.zero_address(),  # CRITICAL
    )
```
**Tool Detection**: Tealer `unprotected-rekey`

### **Pattern A2: Unchecked Transaction Fee ⚠️ HIGH**
**Description**: Smart signatures without fee bounds allow balance drain.

```python
# VULNERABLE: No fee validation
def smart_sig():
    return And(
        Txn.type_enum() == TxnType.Payment,
        Txn.receiver() == authorized,
    )
    # Attacker sets fee = account_balance - amount!

# SECURE: Force minimum fee
def smart_sig():
    return And(
        Txn.type_enum() == TxnType.Payment,
        Txn.receiver() == authorized,
        Txn.fee() == Global.min_txn_fee(),  # Exact fee
    )

# OR: Zero fee with fee pooling
def smart_sig():
    return And(
        Txn.type_enum() == TxnType.Payment,
        Txn.receiver() == authorized,
        Txn.fee() == Int(0),  # Another tx pays via pooling
    )
```

### **Pattern A3: Closing Account ⚠️ CRITICAL**
**Description**: Missing CloseRemainderTo validation allows full ALGO drain.

```python
# VULNERABLE: No close check
def payment_escrow():
    return And(
        Txn.type_enum() == TxnType.Payment,
        Txn.amount() <= max_amount,
    )
    # Attacker sets close_remainder_to = attacker_address
    # Gets ALL remaining ALGO!

# SECURE: Validate close field
def payment_escrow():
    return And(
        Txn.type_enum() == TxnType.Payment,
        Txn.amount() <= max_amount,
        Txn.close_remainder_to() == Global.zero_address(),  # CRITICAL
    )
```

### **Pattern A4: Closing Asset ⚠️ CRITICAL**
**Description**: Missing AssetCloseTo validation allows full ASA drain.

```python
# VULNERABLE: No asset close check
def asset_escrow():
    return And(
        Txn.type_enum() == TxnType.AssetTransfer,
        Txn.asset_amount() <= max_amount,
    )
    # Attacker sets asset_close_to = attacker_address
    # Gets ALL remaining ASA balance!

# SECURE: Validate asset close field
def asset_escrow():
    return And(
        Txn.type_enum() == TxnType.AssetTransfer,
        Txn.asset_amount() <= max_amount,
        Txn.asset_close_to() == Global.zero_address(),  # CRITICAL
    )
```

### **Pattern A5: Group Size Check ⚠️ HIGH**
**Description**: Missing group size validation allows operation repetition.

```python
# VULNERABLE: No group size check
def swap():
    return And(
        Gtxn[0].type_enum() == TxnType.Payment,
        Gtxn[1].type_enum() == TxnType.ApplicationCall,
    )
    # Attacker adds 8 more app calls in group!

# SECURE: Validate exact group size
def swap():
    return And(
        Global.group_size() == Int(2),  # CRITICAL: Exact size
        Gtxn[0].type_enum() == TxnType.Payment,
        Gtxn[1].type_enum() == TxnType.ApplicationCall,
    )
```
**Tool Detection**: Tealer `group-size-check`

### **Pattern A6: Access Controls ⚠️ CRITICAL**
**Description**: Unprotected Update/Delete allows contract takeover.

```python
# VULNERABLE: Anyone can update
program = Cond(
    [Txn.on_completion() == OnComplete.UpdateApplication, Return(Int(1))],
)

# SECURE: Only creator can update
is_creator = Txn.sender() == Global.creator_address()
program = Cond(
    [Txn.on_completion() == OnComplete.UpdateApplication, is_creator],
)

# BEST: Disable updates entirely
program = Cond(
    [Txn.on_completion() == OnComplete.UpdateApplication, Return(Int(0))],
)
```
**Tool Detection**: Tealer `update-application-check`

### **Pattern A7: Asset ID Verification ⚠️ HIGH**
**Description**: Missing asset ID check allows worthless token substitution.

```python
# VULNERABLE: No asset ID check
def purchase():
    return And(
        Gtxn[0].type_enum() == TxnType.AssetTransfer,
        Gtxn[0].asset_amount() >= price,
    )
    # Attacker sends worthless token instead of USDC!

# SECURE: Verify specific asset
USDC_ID = Int(12345678)
def purchase():
    return And(
        Gtxn[0].type_enum() == TxnType.AssetTransfer,
        Gtxn[0].xfer_asset() == USDC_ID,  # CRITICAL
        Gtxn[0].asset_amount() >= price,
    )
```

### **Pattern A8: Inner Transaction Fee ⚠️ MEDIUM**
**Description**: Missing fee:0 in inner transactions drains app balance.

```python
# VULNERABLE: No fee specified
InnerTxnBuilder.SetFields({
    TxnField.type_enum: TxnType.Payment,
    TxnField.receiver: recipient,
    TxnField.amount: amount,
    # fee defaults to min_txn_fee, draining app!
})

# SECURE: Explicit zero fee
InnerTxnBuilder.SetFields({
    TxnField.type_enum: TxnType.Payment,
    TxnField.receiver: recipient,
    TxnField.amount: amount,
    TxnField.fee: Int(0),  # CRITICAL: Use fee pooling
})
```

### **Pattern A9: Clear State Transaction ⚠️ HIGH**
**Description**: Checking type without OnComplete allows ClearState bypass.

```python
# VULNERABLE: Only checks type
def validate_group():
    return And(
        Gtxn[1].type_enum() == TxnType.ApplicationCall,
        # Missing OnComplete check!
    )
    # Attacker uses ClearState instead of NoOp!

# SECURE: Check both type AND OnComplete
def validate_group():
    return And(
        Gtxn[1].type_enum() == TxnType.ApplicationCall,
        Gtxn[1].on_completion() == OnComplete.NoOp,  # CRITICAL
    )
```

---

## **Universal Red Flags (Algorand Edition)**

```markdown
## Immediate Red Flags - Stop and Investigate

### Transaction Fields
- [ ] ANY code path missing Txn.rekey_to() check
- [ ] ANY payment missing Txn.close_remainder_to() check
- [ ] ANY asset transfer missing Txn.asset_close_to() check
- [ ] Smart signature without Txn.fee() validation

### Group Transactions
- [ ] Gtxn[] access without Global.group_size() check
- [ ] Type check without OnComplete validation
- [ ] Missing Txn.xfer_asset() verification in swaps

### Access Control
- [ ] UpdateApplication returns Int(1) without sender check
- [ ] DeleteApplication returns Int(1) without sender check
- [ ] Admin address can be changed by non-admin

### Inner Transactions
- [ ] Missing TxnField.fee: Int(0)
- [ ] Push pattern for asset distribution (opt-in DoS)
- [ ] Missing close field settings
```

---

## **Severity Classification (Algorand Edition)**

```markdown
## CRITICAL (Immediate account/fund takeover)
- Missing RekeyTo validation (account takeover)
- Missing CloseRemainderTo validation (full ALGO drain)
- Missing AssetCloseTo validation (full ASA drain)
- Unprotected UpdateApplication (code takeover)

## HIGH (Significant fund loss or manipulation)
- Unchecked transaction fee (balance drain)
- Missing group size check (repeated execution)
- Missing asset ID verification (worthless token accepted)
- Unprotected DeleteApplication

## MEDIUM (Limited loss or DoS)
- Inner transaction fee drain (app balance)
- Clear state transaction bypass
- Time-based replay attacks
- Asset opt-in DoS (push pattern)

## LOW (Minor issues)
- Opcode budget inefficiency
- State schema inefficiency
- Code style issues

## INFO (Suggestions)
- Consider ARC4 for better ABI
- Add event logging
- Documentation improvements
```

---

## **Quick Reference Card**

```
┌─────────────────────────────────────────────────────────────────┐
│               ALGORAND AUDIT QUICK REFERENCE                     │
├─────────────────────────────────────────────────────────────────┤
│ 1. TRANSACTION FIELD CHECKLIST (ALL paths):                     │
│    ✓ Txn.rekey_to() == Global.zero_address()                    │
│    ✓ Txn.close_remainder_to() == Global.zero_address()          │
│    ✓ Txn.asset_close_to() == Global.zero_address()              │
│    ✓ Txn.fee() == Global.min_txn_fee() (smart sigs)             │
│                                                                 │
│ 2. GROUP TRANSACTION CHECKLIST:                                 │
│    ✓ Global.group_size() == expected_size                       │
│    ✓ Gtxn[i].on_completion() validated (not just type)          │
│    ✓ Gtxn[i].xfer_asset() == expected_asset                     │
│                                                                 │
│ 3. ACCESS CONTROL CHECKLIST:                                    │
│    ✓ UpdateApplication → sender == creator/admin                │
│    ✓ DeleteApplication → sender == creator/admin                │
│    ✓ OR: Return Int(0) to disable                               │
│                                                                 │
│ 4. INNER TRANSACTION CHECKLIST:                                 │
│    ✓ TxnField.fee: Int(0) (always!)                             │
│    ✓ Consider pull pattern for assets                           │
│    ✓ Set close fields explicitly                                │
│                                                                 │
│ 5. TOOLS:                                                       │
│    • Tealer: tealer detect --program approval.teal              │
│    • Sandbox: ./sandbox up dev                                  │
│    • Detectors: unprotected-rekey, group-size-check             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📚 **Learning Resources**

### Essential Reading
1. **Algorand Developer Portal**: https://developer.algorand.org/
2. **PyTeal Documentation**: https://pyteal.readthedocs.io/
3. **Tealer Analyzer**: https://github.com/crytic/tealer
4. **Trail of Bits Algorand Patterns**: building-secure-contracts/not-so-smart-contracts/algorand/
5. **ARC Standards**: https://arc.algorand.foundation/

### Tools
1. **Tealer**: https://github.com/crytic/tealer
2. **Algorand Sandbox**: https://github.com/algorand/sandbox
3. **Beaker**: https://github.com/algorand-devrel/beaker (high-level framework)

### Practice
1. **Algorand Developer Tutorials**: https://developer.algorand.org/tutorials/
2. **PyTeal Examples**: https://github.com/algorand/pyteal/tree/master/examples
3. **Past Audit Reports**: Search for Algorand protocol audits
