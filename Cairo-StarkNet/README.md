# Cairo/StarkNet Smart Contract Audit Framework

> A comprehensive audit framework for Cairo smart contracts on StarkNet L2.

## Overview

This framework provides structured methodology for auditing Cairo smart contracts, with special attention to:
- **L1↔L2 Bridge Security**: Cross-layer message validation and fund safety
- **felt252 Arithmetic**: Field element overflow/underflow vulnerabilities
- **Signature Replay**: Nonce management and domain separation
- **L1 Handler Validation**: Authorized sender verification

## Framework Structure

```
Cairo-StarkNet/
├── README.md                              <- This file
├── Audit_Assistant_Playbook_Cairo.md      <- Conversation structure & prompts
├── CommandInstruction-Cairo.md            <- Cairo system prompt
└── Cairo-Audit-Methodology.md             <- Full audit methodology
```

## Quick Start

1. **Read the Playbook**: [Audit_Assistant_Playbook_Cairo.md](./Audit_Assistant_Playbook_Cairo.md)
2. **Use System Prompt**: [CommandInstruction-Cairo.md](./CommandInstruction-Cairo.md)
3. **Follow Methodology**: [Cairo-Audit-Methodology.md](./Cairo-Audit-Methodology.md)

## Key Concepts

### Cairo-Specific Considerations
| Aspect | Cairo/StarkNet Specifics |
|--------|-------------------------|
| **Types** | `felt252` (field element), `u128`, `u256`, `ContractAddress` |
| **Arithmetic** | Field elements wrap at prime P (~2^251) |
| **Storage** | `#[storage]` structs with `LegacyMap` |
| **Entry Points** | `#[external(v0)]`, `#[l1_handler]`, `#[constructor]` |
| **L1↔L2** | `send_message_to_l1_syscall`, L1 handler pattern |

### Critical Attack Vectors
1. **felt252 Overflow/Underflow** - Arithmetic on field elements without bounds
2. **L1→L2 Address Conversion** - Addresses > STARKNET_PRIME map to zero
3. **L1→L2 Message Failure** - Locked funds without cancellation mechanism
4. **Unchecked from_address** - L1 handlers without sender validation
5. **Signature Replay** - Missing nonce/domain separation

### Tool References
- **Caracal**: Static analyzer for Cairo
  - `unchecked-felt252-arithmetic`
  - `unchecked-l1-handler-from`
  - `missing-nonce-validation`
- **Starknet Foundry**: Testing framework
- **Cairo-lint**: Linting tool

## StarkNet Architecture Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                        ETHEREUM L1                               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │  L1 Bridge   │───▶│ StarkNet Core│───▶│  L1 Verifier │       │
│  │  Contract    │    │   Contract   │    │   Contract   │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│         │                   │                                    │
│         │ sendMessageToL2() │ Merkle Root                       │
│         ▼                   ▼                                    │
├─────────────────────────────────────────────────────────────────┤
│                        STARKNET L2                               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Sequencer  │───▶│  L2 Contract │───▶│    Prover    │       │
│  │              │    │ #[l1_handler]│    │   (STARK)    │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
└─────────────────────────────────────────────────────────────────┘

L1→L2 Message Flow:
1. L1 contract calls sendMessageToL2()
2. Message enters L1→L2 message queue
3. Sequencer includes message in L2 block
4. #[l1_handler] processes message

L2→L1 Message Flow:
1. L2 contract calls send_message_to_l1_syscall()
2. Message included in state update proof
3. L1 contract can consume message after proof verification
```

## 📚 Known Exploit Database

### StarkNet/Cairo Exploits
- [ ] **ZKLend (2024)**: L1 message validation bypass
- [ ] **JediSwap**: Price oracle manipulation
- [ ] **MySwap**: felt252 arithmetic issues

### L1↔L2 Bridge Exploits (Cross-Reference)
- [ ] **Wormhole Pattern**: Signature validation bypass on bridge
- [ ] **Ronin Pattern**: Validator compromise in bridge
- [ ] **Nomad Pattern**: Merkle proof validation failure

### General Cairo Vulnerabilities
- [ ] felt252 overflow in balance calculations
- [ ] Missing from_address check in L1 handlers
- [ ] Signature replay due to missing nonce
- [ ] L1→L2 address truncation to zero
- [ ] Locked funds from failed L1→L2 messages

## Integration with Other Frameworks

| If Auditing... | Also Reference... |
|----------------|-------------------|
| Cairo + Solidity L1 Bridge | `../Solidity-EVM/` |
| Cairo + CosmWasm IBC | `../RustBaseSmartContract/` |
| Cairo with Python tooling | `../Algorand-PyTeal/` |

## Learning Resources

1. **Cairo Book**: https://book.cairo-lang.org/
2. **StarkNet Documentation**: https://docs.starknet.io/
3. **OpenZeppelin Cairo**: https://github.com/OpenZeppelin/cairo-contracts
4. **Trail of Bits Cairo Patterns**: building-secure-contracts/not-so-smart-contracts/cairo/
5. **Caracal Analyzer**: https://github.com/crytic/caracal
