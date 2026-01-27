# Audit Assistant Playbook – Cairo/StarkNet Edition

> A structured conversation framework for Cairo smart contract security audits.

---

## 1. Introduction

### Purpose
This playbook structures how security auditors interact with LLMs when auditing Cairo smart contracts on StarkNet. It provides:
- **Conversation templates** for each audit phase
- **Cairo-specific prompts** for L1↔L2 security
- **Quality gates** to ensure thorough analysis

### Related Documents
| Document | Purpose |
|----------|---------|
| [Cairo-Audit-Methodology.md](./Cairo-Audit-Methodology.md) | Technical methodology |
| [CommandInstruction-Cairo.md](./CommandInstruction-Cairo.md) | System prompt |
| [../Solidity-EVM/](../Solidity-EVM/) | For L1 bridge contracts |

---

## 2. Conversation Structure

### Phase 1: Scope Definition
```markdown
## Audit Scope

**Contract(s):** [List Cairo contracts]
**L1 Bridge (if any):** [Solidity contract address/repo]
**StarkNet Version:** [e.g., 0.13.x]
**Cairo Version:** [Cairo 1.x / Cairo 2.x]

**Focus Areas:**
- [ ] L1↔L2 message security
- [ ] felt252 arithmetic safety
- [ ] Access control and ownership
- [ ] Signature verification
- [ ] Storage and state management

**Out of Scope:**
- [ ] [List any exclusions]

**Known Integrations:**
- L1 Bridge: [Contract name]
- Oracles: [e.g., Pragma, Empiric]
- DEX: [e.g., JediSwap, 10KSwap]
```

### Phase 2: Entry Point Mapping
```markdown
## Entry Point Analysis Request

Please analyze the following Cairo contract and identify ALL entry points:

1. `#[external(v0)]` functions (state-changing)
2. `#[l1_handler]` functions (L1→L2 messages)
3. `#[constructor]` initialization
4. View functions (read-only)

For each entry point, provide:
- Function signature
- Access control (if any)
- State changes (storage writes)
- External interactions (L2→L1 messages, contract calls)
- Arithmetic operations on felt252

**Contract Code:**
[Paste contract here]
```

### Phase 3: L1↔L2 Security Analysis
```markdown
## L1↔L2 Security Analysis Request

For each `#[l1_handler]` function:

1. **from_address Validation:**
   - Is from_address compared to authorized L1 contract?
   - Where is the authorized address stored?

2. **Address Conversion:**
   - Are ContractAddress parameters validated for zero?
   - Is L1 side validating addresses < STARKNET_PRIME?

3. **Message Flow:**
   - Can messages fail silently?
   - Is there a cancellation/recovery mechanism?
   - What happens to funds if L2 processing fails?

4. **Symmetric Validation:**
   - Are access controls identical on L1 and L2?
   - Can funds be deposited but not withdrawn?

**L1 Contract (if available):**
[Paste Solidity bridge contract]

**L2 Contract:**
[Paste Cairo contract]
```

### Phase 4: Arithmetic Safety Analysis
```markdown
## felt252 Arithmetic Analysis Request

For each function with arithmetic operations:

1. **Identify felt252 Operations:**
   - Addition: `a + b`
   - Subtraction: `a - b`
   - Multiplication: `a * b`
   - Division: `a / b`

2. **Check Bounds:**
   - Is there validation before arithmetic?
   - What's the maximum expected value?
   - Could user input cause overflow?

3. **Recommend Safe Types:**
   - Should this use u128 or u256 instead?
   - Are there existing safe math utilities?

**Focus on:**
- Balance updates
- Fee calculations
- Reward distributions
- Price computations
```

### Phase 5: Deep Dive Prompts

#### Signature Verification Analysis
```markdown
## Signature Security Analysis

For signature verification logic:

1. **Nonce Management:**
   - Is there a per-signer nonce?
   - Is nonce incremented BEFORE execution?
   - Can the same signature be replayed?

2. **Domain Separation:**
   - Does the hash include chain_id?
   - Does the hash include contract_address?
   - Is EIP-712 style domain separator used?

3. **Implementation:**
   - Using OpenZeppelin Account?
   - Custom signature verification?
   - Which hash function (poseidon, pedersen)?

**Verify against:**
- Cross-chain replay (mainnet↔testnet)
- Same-chain replay (nonce reuse)
- Signature malleability
```

#### Storage Analysis
```markdown
## Storage Security Analysis

For storage operations:

1. **Storage Layout:**
   - `LegacyMap` usage and keys
   - Potential key collisions
   - Upgrade compatibility

2. **Access Patterns:**
   - Read-modify-write sequences
   - Potential reentrancy via callbacks
   - Storage slot predictability

3. **Initialization:**
   - Constructor sets all required state?
   - Can storage be re-initialized?
   - Zero-value handling
```

---

## 3. Finding Documentation

### Finding Template
```markdown
## [SEVERITY] Finding Title

### Summary
[One paragraph describing the issue]

### Vulnerability Details

**Location:** `contract.cairo::function_name` (lines X-Y)

**Root Cause:**
[Technical explanation of why this is vulnerable]

**Vulnerable Code:**
```cairo
// Code snippet showing the vulnerability
```

### Impact
[What can an attacker achieve? Quantify if possible]

### Attack Scenario
1. Attacker deploys malicious L1 contract (if L1↔L2 issue)
2. Attacker calls X with parameters Y
3. This causes Z
4. Result: [Fund loss / DoS / Access bypass]

### Proof of Concept
```cairo
#[test]
fn test_exploit() {
    // Test code demonstrating the vulnerability
}
```

### Recommended Mitigation
```cairo
// Secure implementation
```

### References
- Caracal Detector: [detector name]
- Similar Exploit: [historical reference]
- Documentation: [relevant docs]
```

### Severity Guidelines

| Severity | Criteria | Examples |
|----------|----------|----------|
| **CRITICAL** | Direct fund theft, L1 handler bypass | Unchecked from_address, signature forgery |
| **HIGH** | Significant fund loss, protocol insolvency | felt252 overflow in balances, L1↔L2 desync |
| **MEDIUM** | Limited loss, DoS, access control issues | Missing group validation, fee manipulation |
| **LOW** | Gas inefficiency, missing events | Suboptimal storage access patterns |
| **INFO** | Suggestions, best practices | Code style, documentation gaps |

---

## 4. Quality Gates

### Before Reporting a Finding
- [ ] **Reachable**: Function is #[external(v0)] or #[l1_handler]
- [ ] **Triggerable**: Can construct valid transaction/message
- [ ] **Impactful**: Clear negative outcome (fund loss, DoS, etc.)
- [ ] **Not Mitigated**: No existing protection in code
- [ ] **Reproducible**: PoC test passes or clear attack path

### L1↔L2 Specific Gates
- [ ] **L1 Side Audited**: Solidity contract reviewed
- [ ] **Message Flow Traced**: End-to-end path documented
- [ ] **Cancellation Tested**: Recovery mechanism verified
- [ ] **Address Conversion**: Edge cases checked

---

## 5. Tool Integration

### Caracal Commands
```bash
# Full analysis
caracal detect --target ./src

# Specific detector
caracal detect --target ./src --detectors unchecked-l1-handler-from

# JSON output for parsing
caracal detect --target ./src --output-format json
```

### Starknet Foundry
```bash
# Run all tests
snforge test

# Run specific test
snforge test test_exploit

# With gas reporting
snforge test --gas-report
```

### Manual Review Checklist Integration
```markdown
## Manual Review Checklist

### L1 Handler Security
- [ ] All #[l1_handler] validate from_address
- [ ] Authorized L1 addresses stored securely
- [ ] Cannot be updated without proper access control

### Arithmetic Safety
- [ ] No unprotected felt252 arithmetic
- [ ] Balance operations use u128/u256
- [ ] Explicit overflow checks where needed

### Signature Security
- [ ] Nonces incremented per signer
- [ ] Domain separator includes chain_id
- [ ] Using established libraries (OZ)

### Access Control
- [ ] Owner/admin pattern implemented correctly
- [ ] Upgrade functions protected
- [ ] Initialization cannot be repeated
```

---

## 6. Example Conversations

### Example 1: L1 Handler Vulnerability
```
User: Please review this L1 handler for security issues:

#[l1_handler]
fn handle_deposit(
    ref self: ContractState,
    from_address: felt252,
    user: ContractAddress,
    amount: u256
) {
    let balance = self.balances.read(user);
    self.balances.write(user, balance + amount);
}