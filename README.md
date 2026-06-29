# Solana DePIN Skill for Claude Code

A production-grade Claude Code / Codex skill for building Decentralized Physical Infrastructure Networks (DePIN) on Solana. Covers the full DePIN builder lifecycle: token reward mechanics, oracle integration, proof mechanisms, device identity, ZK compression at scale, operator economics, governance, and deployment patterns.

> **Extends**: [solana-dev-skill](https://github.com/solana-foundation/solana-dev-skill)

## Overview

Solana is the **#1 chain for DePIN** — Helium, Hivemapper, Render Network, Nosana, GEODNET, DIMO, and UpRock all ship on Solana. Yet builders face the same unsolved problems: how to design token rewards for thousands of operators, verify physical location without centralized trust, manage device identity at scale, and keep account costs under control with ZK compression.

This skill fills that gap. It packages the collective patterns from Solana's largest DePINs into a progressive, token-efficient skill that any Claude Code / Codex agent can load.

```
┌─────────────────────────────────────────────────────────────────┐
│                     solana-depin-skill (addon)                  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  DePIN Builder Skills                                    │  │
│  │  ├── Token Rewards & Inflation Models                    │  │
│  │  ├── Reward Distribution (Streamflow, Merkle, ZK)        │  │
│  │  ├── Proof Mechanisms (PoL, PoW, PoB, Consensus)         │  │
│  │  ├── Oracle Integration (Pyth, Switchboard, VRF)         │  │
│  │  ├── Device Identity & Anti-Sybil                        │  │
│  │  ├── ZK Compression for 1M+ Device State                 │  │
│  │  ├── Staking, Slashing & Governance                      │  │
│  │  └── Indexing, Webhooks & Operator Onboarding            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼ references                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  solana-dev-skill (core)                                  │  │
│  │  ├── Frontend (framework-kit, kit-web3-interop)           │  │
│  │  ├── Programs (Anchor, Pinocchio)                         │  │
│  │  ├── Testing (LiteSVM, Mollusk, Surfpool)                 │  │
│  │  └── Security (program + client checklists)               │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## The Problem

DePIN builders on Solana repeatedly rebuild the same infrastructure:

- **Token rewards** — designing inflation models, emission schedules, and reward formulas
- **Operator payouts** — streaming rewards to thousands of devices efficiently
- **Proof mechanisms** — verifying physical location, work, or bandwidth cryptographically
- **Oracle integration** — connecting real-world data (prices, weather, GPS) to on-chain logic
- **Device identity** — managing millions of device accounts without paying Solana rent for each one
- **Staking & governance** — operator staking, slashing conditions, DAO parameter control
- **Indexing** — monitoring device state changes in real-time at scale

This skill provides battle-tested patterns from Solana's largest DePINs so you don't have to invent them yourself.

## What's Included

### DePIN-Specific Skills (This Addon)

| Skill | Description |
|-------|-------------|
| [SKILL.md](skill/SKILL.md) | Entry point — routing hub with progressive disclosure |
| [overview.md](skill/overview.md) | DePIN landscape, key concepts, architectural patterns |
| [token-rewards.md](skill/token-rewards.md) | Token mechanics, inflation models (burn-and-mint, SaaS buyback), emission schedules |
| [reward-distribution.md](skill/reward-distribution.md) | Streamflow streaming/vesting, Merkle distributor, compressed airdrops |
| [proof-mechanisms.md](skill/proof-mechanisms.md) | Proof of Location, Work, Bandwidth, Data — patterns from Helium, Hivemapper, Nosana |
| [oracle-integration.md](skill/oracle-integration.md) | Pyth price feeds, Switchboard custom feeds, VRF for challenges |
| [device-identity.md](skill/device-identity.md) | Compressed PDAs, anti-sybil, hardware attestation, session keys |
| [zk-compression.md](skill/zk-compression.md) | ZK Compression (Helius/Light) for 100K+ device accounts |
| [staking-governance.md](skill/staking-governance.md) | Staking pools, Realms DAO, Squads multisig, parameter governance |
| [indexing-operator-onboarding.md](skill/indexing-operator-onboarding.md) | Helius webhooks, DAS API, embedded wallets for non-crypto operators |

### Core Skills (from solana-dev-skill)

| Skill | Description |
|-------|-------------|
| frontend-framework-kit.md | React hooks, wallet connection |
| kit-web3-interop.md | Kit ↔ web3.js boundary patterns |
| security.md | Security checklist (programs + clients) |
| programs-anchor.md | Anchor framework patterns |
| programs-pinocchio.md | High-performance Pinocchio |
| idl-codegen.md | IDL generation, client codegen |
| testing.md | LiteSVM, Mollusk, Surfpool |

## Installation

### Recommended: Custom Install

```bash
git clone https://github.com/titalabs/rw-usecase-skills
cd rw-usecase-skills
./install-custom.sh
```

The custom installer lets you:
- Choose install location (personal `~/.claude/skills/` or project `./.claude/skills/`)
- Skip core skill if you already have `solana-dev-skill`
- Choose where to place `CLAUDE.md`

### Standard Install (Automation)

```bash
./install.sh        # Interactive with defaults
./install.sh -y     # Non-interactive, all defaults
```

**Standard defaults:**
- Location: `~/.claude/skills/`
- Installs both `solana-dev` and `solana-depin` skills
- Copies `CLAUDE.md` to `~/.claude/`

### Install Comparison

| Feature | `install.sh` | `install-custom.sh` |
|---------|--------------|---------------------|
| Interactive prompts | Minimal (Y/n) | Full menu |
| Location choice | Default only | Personal/Project/Custom |
| Core skill handling | Always installs | Detects existing |
| CLAUDE.md placement | `~/.claude/` | Choose location |
| Best for | Automation, scripts | Manual setup |

### If You Already Have solana-dev-skill

Use `./install-custom.sh` — it detects existing installations and only installs the DePIN addon.

## Default Stack (June 2026)

### Token & Rewards
| Layer | Choice |
|-------|--------|
| Token Standard | SPL Token 2022 |
| Reward Streaming | Streamflow SDK v2 |
| Compressed Distribution | Helius ZK Compression API, @lightprotocol/stateless.js |
| Airdrops | Merkle distributor, compressed drops |

### Oracles
| Layer | Choice |
|-------|--------|
| Price Feeds | Pyth (push/pull, 400ms) |
| Custom Data | Switchboard (permissionless, Surge sub-100ms) |
| Randomness | Switchboard VRF (TEE commit-reveal) |

### Device Identity
| Layer | Choice |
|-------|--------|
| Registry | Compressed PDAs via Helius ZK Compression |
| Anti-Sybil | Trust scores, hardware attestation, stake gates |
| Wallets | Session keys (IoT), auto-generated (non-crypto operators) |

### Governance
| Layer | Choice |
|-------|--------|
| DAO | Realms / SPL Governance |
| Multisig | Squads v5 |
| Staking | Streamflow pools, custom Anchor programs |

### Indexing
| Layer | Choice |
|-------|--------|
| API | Helius DAS API |
| Webhooks | Helius real-time event hooks |
| Streaming | Helius WebSocket |

### Program Development
| Layer | Choice |
|-------|--------|
| Framework | Anchor (default), Pinocchio (CU-optimized) |
| Testing | LiteSVM, Mollusk, Surfpool |
| Security | solana-dev security checklists |

## Agents

| Agent | Model | Purpose |
|-------|-------|---------|
| **depin-architect** | opus | DePIN architecture, tokenomics, system design |
| **tech-docs-writer** | sonnet | Documentation |

## Commands

| Command | Purpose |
|---------|---------|
| **/design-depin-tokenomics** | Design token economics for a DePIN project |
| **/setup-reward-stream** | Configure Streamflow reward streaming for operators |
| **/generate-device-registry** | Generate compressed device registry Anchor program |

## Usage Examples

### Tokenomics Design
```
"Design tokenomics for a wireless DePIN with 50K hotspots"
"Create a burn-and-mint equilibrium model for my data marketplace"
"Design reward emission schedule with daily operator payouts"
```

### Proof Mechanisms & Oracles
```
"Set up Proof of Location using Switchboard oracles"
"Integrate Pyth price feeds for reward calculations"
"Design a challenge-response protocol for device verification"
```

### Device Management at Scale
```
"Design compressed device accounts for 1M sensors"
"Create a device registry Anchor program with ZK Compression"
"Set up Helius webhooks for real-time device monitoring"
```

### Governance & Operations
```
"Set up operator staking with slashing conditions"
"Create a Realms DAO for protocol parameter voting"
"Configure Squads multisig for program upgrade authority"
```

## Repository Structure

```
solana-depin-skill/
├── CLAUDE.md                    # Claude configuration
├── README.md                    # This file
├── install.sh                   # Standard installer (defaults)
├── install-custom.sh            # Custom installer (full options)
│
├── skill/                       # DePIN addon skills
│   ├── SKILL.md                 # Entry point (routing hub)
│   ├── overview.md              # DePIN landscape & patterns
│   ├── token-rewards.md         # Token mechanics & inflation models
│   ├── reward-distribution.md   # Streamflow, Merkle, compressed drops
│   ├── proof-mechanisms.md      # PoL, PoW, PoB, consensus patterns
│   ├── oracle-integration.md    # Pyth, Switchboard, VRF, custom feeds
│   ├── device-identity.md       # Device PDAs, anti-sybil, session keys
│   ├── zk-compression.md        # ZK Compression for large-scale state
│   ├── staking-governance.md    # Staking pools, DAOs, multisig
│   └── indexing-operator-onboarding.md  # Helius webhooks, embedded wallets
│
├── agents/                      # Specialized agents
│   ├── depin-architect.md       # DePIN architecture & tokenomics
│   └── tech-docs-writer.md      # Documentation
│
├── commands/                    # Workflow commands
│   ├── design-depin-tokenomics.md
│   ├── setup-reward-stream.md
│   └── generate-device-registry.md
│
└── rules/                       # Auto-loading code rules
    ├── anchor.md                # Anchor program patterns for DePIN
    ├── typescript.md            # TypeScript SDK patterns
    └── rust.md                  # Rust program patterns
```

## Development Workflow

### Two-Strike Rule

If a build or test fails twice on the same issue:
1. Claude will **STOP** immediately
2. Present error output and code change
3. Ask for user guidance

## Related

- [solana-dev-skill](https://github.com/solana-foundation/solana-dev-skill) — Core Solana development skill (required dependency)
- [solana-ai-kit](https://github.com/solanabr/solana-ai-kit) — The Solana AI Kit this skill plugs into

## Contributing

Contributions are welcome! Please ensure any updates reflect current Solana DePIN ecosystem best practices.

1. Fork the repository
2. Create a feature branch: `git checkout -b feat/my-feature-29-06-2026`
3. Make your changes
4. Submit a pull request

## License

MIT License — see [LICENSE](LICENSE) for details.

---

Maintained by [titalabs](https://github.com/titalabs)
