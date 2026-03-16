# Execrow Agent Runtime

> The execution engine for autonomous AI agents in the Execrow protocol — wallet provisioning, permission enforcement, task execution, and multi-agent negotiation.

This repository contains the agent runtime — the software that runs inside or alongside an AI agent and gives it the ability to interact with the Execrow protocol. It handles wallet management, enforces the permission scope set by the human operator, executes the negotiation protocol when agreeing on task terms with counterparties, and manages the full lifecycle of a task from negotiation to payment.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Modules](#modules)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Integrating With Your Agent](#integrating-with-your-agent)
- [Development](#development)
- [Testing](#testing)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

The agent runtime is the bridge between an AI agent's decision-making and the Execrow protocol. Any AI agent — whether built with LangChain, AutoGen, CrewAI, or a fully custom framework — can integrate the Execrow agent runtime to gain financial autonomy.

The runtime handles:

- **Wallet Management** — creating, loading, and managing a Stellar wallet for the agent with hardware-level key security
- **Permission Enforcement** — enforcing the operator-defined permission scope locally before any on-chain action is attempted, preventing agents from exceeding their mandate
- **Negotiation Protocol** — a structured protocol for agents to negotiate task terms, payment amounts, deadlines, and proof requirements with counterparties
- **Task Lifecycle Manager** — tracks the full state of every active task from negotiation through escrow creation, execution, proof submission, and payment
- **Multi-Agent Coordination** — enables orchestrator agents to spawn subagents, delegate tasks, and manage payment flows across agent hierarchies
- **Human Override Handler** — listens for emergency pause signals from the operator and halts all financial activity immediately

---

## Architecture
```

┌──────────────────────────────────────────────────────────┐
│                    Your AI Agent                         │
│              (LangChain / AutoGen / Custom)               │
└─────────────────────┬────────────────────────────────────┘
                      │ Execrow SDK
                      ▼
┌──────────────────────────────────────────────────────────┐
│                  Agent Runtime                           │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │   Wallet     │  │  Permission  │  │  Negotiation  │  │
│  │  Manager     │  │   Enforcer   │  │   Protocol    │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │    Task      │  │  Multi-Agent │  │   Override    │  │
│  │  Lifecycle   │  │ Coordinator  │  │   Handler     │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
│                                                          │
└─────────────────────┬────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────┐
│              Execrow Contracts (Stellar)                 │
│                                                          │
│  Agent Registry • Escrow Engine • Reputation Ledger      │
└──────────────────────────────────────────────────────────┘
```

---

## Modules

### `wallet`
Manages the agent's Stellar wallet. Handles key generation and secure storage, balance queries, transaction signing, and interaction with the agent registry contract to provision and update the on-chain wallet. Enforces that all outgoing transactions stay within the operator-defined budget ceiling.

### `permissions`
Enforces the permission scope defined by the human operator. Before any financial action is executed, the permissions module checks it against the current scope. If the action exceeds the scope — wrong counterparty, too large an amount, unauthorized operation type — it is rejected locally before hitting the chain.

### `negotiation`
Implements the Execrow negotiation protocol. When an agent needs to agree on task terms with a counterparty, the negotiation module handles the structured back-and-forth — proposal, counter-proposal, acceptance, and rejection — until both parties agree on deliverables, payment, deadline, and proof requirements, or the negotiation fails.

### `task`
Manages the full lifecycle of a task from initial negotiation to final payment. Tracks task state (negotiating, escrowed, in-progress, proof-submitted, completed, disputed, cancelled), persists task history, and coordinates between the negotiation, wallet, and proof modules at each state transition.

### `multi_agent`
Enables orchestrator agents to spawn subagents, delegate tasks to them, and manage hierarchical payment flows. An orchestrator can lock a budget for a subagent, monitor its progress, and release or claw back funds based on outcomes.

### `override_handler`
Listens continuously for emergency pause signals from the human operator. When a pause signal is detected — either via a signed on-chain transaction or an authenticated API call — all pending financial operations are halted immediately, no new escrows are created, and the operator is notified of current open positions.

---

## Repository Structure
```

execrow-agent-runtime/
├── src/
│   ├── lib.rs
│   ├── wallet/
│   │   ├── mod.rs
│   │   ├── manager.rs
│   │   ├── keygen.rs
│   │   ├── signer.rs
│   │   └── balance.rs
│   ├── permissions/
│   │   ├── mod.rs
│   │   ├── enforcer.rs
│   │   ├── scope.rs
│   │   └── errors.rs
│   ├── negotiation/
│   │   ├── mod.rs
│   │   ├── protocol.rs
│   │   ├── proposal.rs
│   │   ├── session.rs
│   │   └── errors.rs
│   ├── task/
│   │   ├── mod.rs
│   │   ├── lifecycle.rs
│   │   ├── state.rs
│   │   ├── history.rs
│   │   └── errors.rs
│   ├── multi_agent/
│   │   ├── mod.rs
│   │   ├── orchestrator.rs
│   │   ├── subagent.rs
│   │   └── hierarchy.rs
│   └── override_handler/
│       ├── mod.rs
│       ├── listener.rs
│       ├── pause.rs
│       └── notify.rs
├── tests/
│   ├── wallet_test.rs
│   ├── permissions_test.rs
│   ├── negotiation_test.rs
│   ├── task_test.rs
│   └── integration/
│       ├── full-task-flow-test.rs
│       └── multi-agent-flow-test.rs
├── examples/
│   ├── basic_agent.rs
│   ├── multi_agent_orchestrator.rs
│   └── langchain_integration.rs
├── Cargo.toml
├── .gitignore
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

---

## Prerequisites

- [Rust](https://www.rust-lang.org/tools/install) `>=1.74.0`
- [Node.js](https://nodejs.org) `>=20.0.0` for TypeScript bindings
- Stellar testnet account

---

## Getting Started
```bash
git clone https://github.com/Execrow/execrow-agent-runtime.git
cd execrow-agent-runtime
cargo build
cargo test
```

---

## Integrating With Your Agent
```rust
use execrow_agent_runtime::{AgentRuntime, RuntimeConfig, PermissionScope};

#[tokio::main]
async fn main() {
    let config = RuntimeConfig {
        budget_ceiling: 100_000_000, // in stroops
        allowed_counterparties: vec![],
        operator_override_key: "YOUR_OPERATOR_PUBLIC_KEY".to_string(),
        network: Network::Testnet,
    };

    let runtime = AgentRuntime::new(config).await.unwrap();
    let wallet = runtime.provision_wallet().await.unwrap();

    println!("Agent wallet: {}", wallet.public_key());
}
```

---

## Contributing

We especially welcome contributors with experience in AI agent frameworks, Rust async programming, and cryptographic key management. Browse [`good first issue`](https://github.com/Execrow/execrow-agent-runtime/issues?q=label%3A%22good+first+issue%22) to get started.

---

## License

[Apache 2.0](LICENSE) — Execrow Organization, 2025

---

<div align="center">

Part of the [Execrow](https://github.com/Execrow) open-source organization.

**The payment infrastructure layer for autonomous AI agents.**

</div>
