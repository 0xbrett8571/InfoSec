# Audit Assistant Playbook

A cognitive playbook for experienced smart contract auditors.

This is NOT an automated audit tool.
It does NOT replace auditor judgment.
It structures how auditors think, explore, validate, and report findings.

## Repository Structure

```
audit-assistant-playbook/
├── Audit_Assistant_Playbook.md           <- Solidity conversation structure & prompts
├── VULNERABILITY_PATTERNS_INTEGRATION.md <- ClaudeSkills pattern integration analysis
├── InfoSec/
│   ├── audit-workflow1.md                <- Solidity manual audit methodology
│   ├── audit-workflow2.md                <- Semantic phase analysis (SNAPSHOT→COMMIT)
│   ├── CommandInstruction.md             <- Solidity system prompt
│   └── report-writing.md                 <- Finding report templates
├── RustBaseSmartContract/
│   ├── README.md                         <- Rust framework documentation
│   ├── Audit_Assistant_Playbook_Rust.md  <- Rust conversation structure & prompts
│   ├── CommandInstruction-Rust.md        <- Rust system prompt
│   └── Rust-Smartcontract-workflow.md    <- Rust audit methodology (w/ ClaudeSkills patterns)
├── Go-SmartContract/
│   ├── README.md                         <- Go framework documentation
│   ├── Audit_Assistant_Playbook_Go.md    <- Go conversation structure & prompts
│   ├── CommandInstruction-Go.md          <- Go system prompt
│   └── Go-Smart-Contract-Audit-Methodology.md <- Go audit methodology (w/ ClaudeSkills patterns)
├── Cairo-StarkNet/
│   ├── README.md                         <- Cairo framework documentation
│   ├── Audit_Assistant_Playbook_Cairo.md <- Cairo conversation structure & prompts
│   ├── CommandInstruction-Cairo.md       <- Cairo system prompt
│   └── Cairo-Audit-Methodology.md        <- Cairo audit methodology (w/ ClaudeSkills patterns)
├── Algorand-PyTeal/
│   ├── README.md                         <- Algorand framework documentation
│   ├── Audit_Assistant_Playbook_Algorand.md <- Algorand conversation structure & prompts
│   ├── CommandInstruction-Algorand.md    <- Algorand system prompt
│   └── Algorand-Audit-Methodology.md     <- Algorand audit methodology (w/ ClaudeSkills patterns)
├── ClaudeSkills/                         <- Vulnerability pattern resources (submodule)
└── README.md
```

## Framework Overview

### Solidity Framework (`InfoSec/`)
For EVM-based smart contracts (Ethereum, Arbitrum, Optimism, etc.)

| Component | Purpose |
|-----------|---------|
| **Playbook** | Conversation structure: roles, prompts, chat lifecycle |
| **Workflow 1** | Manual audit phases, time-boxing, attack vectors, finding templates |
| **Workflow 2** | Semantic phase classification, inheritance tracing, cross-phase attacks |
| **CommandInstruction** | System prompt enforcing methodology compliance |

### Rust Framework (`RustBaseSmartContract/`)
For Rust-based smart contracts (CosmWasm, Solana, Substrate)

| Component | Purpose |
|-----------|---------|
| **Playbook Rust** | Rust-specific conversation structure & prompts |
| **Workflow** | Ownership analysis, panic safety, arithmetic checks |
| **CommandInstruction Rust** | Rust system prompt with framework-specific guidance |

### Go Framework (`Go-SmartContract/`)
For Go-based blockchain applications (Cosmos SDK, Tendermint, IBC)

| Component | Purpose |
|-----------|---------|
| **Playbook Go** | Go-specific conversation structure & prompts |
| **Methodology** | Pointer safety, error handling, zero-value analysis |
| **CommandInstruction Go** | Go system prompt with Cosmos/Tendermint guidance |

### Cairo Framework (`Cairo-StarkNet/`)
For Cairo-based smart contracts (StarkNet L2)

| Component | Purpose |
|-----------|---------|
| **Playbook Cairo** | Cairo-specific conversation structure & prompts |
| **Methodology** | felt252 arithmetic, L1↔L2 messaging, signature replay |
| **CommandInstruction Cairo** | Cairo system prompt with StarkNet bridge guidance |

### Algorand Framework (`Algorand-PyTeal/`)
For Algorand smart contracts and smart signatures (PyTeal/TEAL)

| Component | Purpose |
|-----------|---------|
| **Playbook Algorand** | Algorand-specific conversation structure & prompts |
| **Methodology** | Transaction field validation, group security, inner transactions |
| **CommandInstruction Algorand** | Algorand system prompt with Tealer guidance |

## Key Concepts

### Universal (All Frameworks)
- **Semantic Phases**: SNAPSHOT → ACCOUNTING → VALIDATION → MUTATION → COMMIT
- **Validation Checks**: Reachability, State Freshness, Execution Closure, Economic Realism
- **Time-Boxing**: 40/40/20 rule (Triage → Deep Dive → Cross-Function)
- **Known Exploit Patterns**: Historical database per ecosystem

### Solidity-Specific
- Storage layout verification (upgradeable contracts)
- Flash loan attack spine
- Reentrancy via cross-function state

### Rust-Specific
- Ownership & borrowing analysis
- Panic safety (`.unwrap()`, `.expect()`)
- Arithmetic safety (`checked_*`, `saturating_*`)
- Framework concerns (CosmWasm IBC, Solana PDAs, Substrate weights)

### Go-Specific
- Pointer vs value receiver analysis
- Error handling (ignored errors, panic safety)
- Zero-value struct exploitation
- Framework concerns (Cosmos SDK keepers, IBC callbacks, ABCI)

### Cairo-Specific
- `felt252` arithmetic (field element modulo prime)
- L1↔L2 message validation (`from_address` spoofing)
- Address conversion (`ContractAddress` ↔ `felt252`)
- Signature replay protection (nonce/timestamp checks)
- L1 handler access control (`#[l1_handler]`)

### Algorand-Specific
- Transaction field validation (RekeyTo, CloseRemainderTo, AssetCloseTo)
- Group transaction size and OnComplete validation
- Inner transaction fee (always `fee: Int(0)`)
- Smart signature fee drain prevention
- Asset ID verification in swaps

## Quick Start

### For Solidity Audits
1. Read [Audit_Assistant_Playbook.md](./Audit_Assistant_Playbook.md)
2. Use [CommandInstruction.md](./InfoSec/CommandInstruction.md) as system prompt
3. Follow methodology in [audit-workflow1.md](./InfoSec/audit-workflow1.md)

### For Rust Audits
1. Read [Audit_Assistant_Playbook_Rust.md](./RustBaseSmartContract/Audit_Assistant_Playbook_Rust.md)
2. Use [CommandInstruction-Rust.md](./RustBaseSmartContract/CommandInstruction-Rust.md) as system prompt
3. Follow methodology in [Rust-Smartcontract-workflow.md](./RustBaseSmartContract/Rust-Smartcontract-workflow.md)

### For Go Audits
1. Read [Audit_Assistant_Playbook_Go.md](./Go-SmartContract/Audit_Assistant_Playbook_Go.md)
2. Use [CommandInstruction-Go.md](./Go-SmartContract/CommandInstruction-Go.md) as system prompt
3. Follow methodology in [Go-Smart-Contract-Audit-Methodology.md](./Go-SmartContract/Go-Smart-Contract-Audit-Methodology.md)

### For Cairo/StarkNet Audits
1. Read [Audit_Assistant_Playbook_Cairo.md](./Cairo-StarkNet/Audit_Assistant_Playbook_Cairo.md)
2. Use [CommandInstruction-Cairo.md](./Cairo-StarkNet/CommandInstruction-Cairo.md) as system prompt
3. Follow methodology in [Cairo-Audit-Methodology.md](./Cairo-StarkNet/Cairo-Audit-Methodology.md)

### For Algorand/PyTeal Audits
1. Read [Audit_Assistant_Playbook_Algorand.md](./Algorand-PyTeal/Audit_Assistant_Playbook_Algorand.md)
2. Use [CommandInstruction-Algorand.md](./Algorand-PyTeal/CommandInstruction-Algorand.md) as system prompt
3. Follow methodology in [Algorand-Audit-Methodology.md](./Algorand-PyTeal/Algorand-Audit-Methodology.md)

## Who This Is For
- Experienced smart contract auditors
- Security researchers doing manual audits
- Auditors working with LLM assistants

## Who This Is NOT For
- Beginners learning Solidity/Rust/Go
- Automated vulnerability scanning
- "Run once and get bugs" workflows

## License
MIT


