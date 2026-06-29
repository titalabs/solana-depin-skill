---
name: solana-depin
description: "Solana DePIN (Decentralized Physical Infrastructure Networks) builder skill. Covers token reward mechanics, oracle integration, proof mechanisms (PoL/PoW/PoB), device identity, ZK compression at scale, operator economics, governance, and deployment. For Anchor program development and testing, delegates to the core solana-dev skill."
user-invocable: true
---

# Solana DePIN Builder Skill

> **Extends**: [solana-dev-skill](../solana-dev/SKILL.md) — Core Solana development (programs, frontend, testing, security)

## What This Skill Is For

Use this skill when the user asks for:

### Token Rewards & Economics
- Designing tokenomics for a DePIN project (emission schedules, inflation models)
- Burn-and-mint equilibrium, SaaS buyback, or fixed-supply models
- Operator reward formulas based on contribution (uptime, data, compute)

### Reward Distribution
- Streaming rewards to thousands of operators via Streamflow
- Merkle distributor contracts for batch payouts
- Compressed airdrops using ZK Compression (Helius/Light Protocol)
- Vesting schedules for operators and device manufacturers

### Proof Mechanisms
- Proof of Location (Helium-style challenge-response, GPS + WiFi triangulation)
- Proof of Work/Compute (Nosana GPU inference, Render rendering)
- Proof of Bandwidth/Data (UpRock, DIMO, Hivemapper patterns)
- Multi-device consensus for data verification

### Oracle Integration
- Pyth price feeds for reward calculations and staking
- Switchboard custom data feeds for geospatial, weather, or device data
- Switchboard VRF for randomness in challenge protocols
- Custom oracle design for DePIN-specific data

### Device Identity & Anti-Sybil
- Compressed PDA device registry design
- Anti-sybil mechanisms (hardware attestation, TEEs, stake gates)
- Session keys for IoT devices
- Compressed NFT-based device identity

### ZK Compression at Scale
- Compressed accounts for 100K+ device state (Helius ZK Compression)
- Batch commit patterns for device heartbeats
- Cost comparison: uncompressed vs compressed

### Staking, Slashing & Governance
- Operator staking pools with slashing conditions
- Realms/SPL Governance DAO for parameter votes
- Squads multisig for treasury and program upgrades

### Indexing & Operator Onboarding
- Helius webhooks for real-time device monitoring
- DAS API for querying device state
- Embedded wallet UX for non-crypto-native operators

### Program Development (Delegate to Core Skill)
- For Anchor programs → [programs-anchor.md](../solana-dev/programs-anchor.md)
- For Pinocchio programs → [programs-pinocchio.md](../solana-dev/programs-pinocchio.md)
- For IDL/codegen → [idl-codegen.md](../solana-dev/idl-codegen.md)
- For program testing → [testing.md](../solana-dev/testing.md)

## Default Stack Decisions (Opinionated)

### 1) Token & Rewards: SPL Token 2022 + Streamflow
- SPL Token 2022 for advanced token features (transfer hooks, confidential transfers)
- Streamflow SDK v2 for streaming/vesting/staking
- Merkle distributor for batch payouts
- Helius ZK Compression for compressed drops

### 2) Oracles: Pyth + Switchboard
- Pyth for price feeds (push/pull model, 400ms updates)
- Switchboard for custom data feeds and VRF
- Switchboard Surge for sub-100ms streaming data

### 3) Device Identity: Compressed PDAs
- Helius ZK Compression APIs for compressed device accounts
- `@lightprotocol/stateless.js` SDK for compression operations
- Gradual trust scores + hardware attestation for anti-sybil

### 4) Governance: Realms + Squads
- Realms/SPL Governance for DAO parameter voting
- Squads v5 for program upgrades and treasury management
- Streamflow staking pools for operator incentives

### 5) Indexing: Helius
- DAS API for querying device state
- Webhooks for real-time state change notifications
- WebSocket for streaming account updates

### 6) Testing
- Programs: LiteSVM, Mollusk, Surfpool (see core skill)
- SDK integration: Vitest + Jest

## Operating Procedure

### 1. Classify the Task Layer

| Layer | Examples | Skill File(s) |
|-------|----------|---------------|
| DePIN Landscape | "What is DePIN?", "How does Helium work?" | [overview.md](overview.md) |
| Tokenomics | Design rewards, inflation, emissions | [token-rewards.md](token-rewards.md) |
| Operator Payouts | Stream rewards, vest, airdrop | [reward-distribution.md](reward-distribution.md) |
| Proof Systems | PoL, PoW, PoB, consensus | [proof-mechanisms.md](proof-mechanisms.md) |
| Oracle Data | Pyth, Switchboard, VRF | [oracle-integration.md](oracle-integration.md) |
| Device Management | Identity, anti-sybil, session keys | [device-identity.md](device-identity.md) |
| Scale/Compression | 1M devices, compressed accounts | [zk-compression.md](zk-compression.md) |
| Staking/Governance | Operator bonds, DAO, multisig | [staking-governance.md](staking-governance.md) |
| Monitoring/UX | Webhooks, DAS API, embedded wallets | [indexing-operator-onboarding.md](indexing-operator-onboarding.md) |
| Anchor Program | On-chain program logic | solana-dev → [programs-anchor.md](../solana-dev/programs-anchor.md) |

### 2. Pick the Right Agent

| Task Type | Agent | Model |
|-----------|-------|-------|
| High-level design, tokenomics | depin-architect | opus |
| Documentation | tech-docs-writer | sonnet |

### 3. Apply DePIN-Specific Patterns

**Token Rewards:**
- Use burn-and-mint equilibrium for data marketplaces (Helium model)
- SaaS buyback model for service-based DePINs (UpRock model)
- Multi-device consensus for proof of data (Hivemapper model)

**Proof Mechanisms:**
- Challenge-response for proof of location (Helium PoC model)
- Hardware attestation for device identity
- Stake gates for anti-sybil

**Reward Distribution:**
- Streamflow continuous streaming for recurring operator rewards
- Merkle distributor for one-time batch payouts
- ZK compressed drops for large-scale airdrops

### 4. Add Tests

- **Programs**: LiteSVM/Mollusk (see [testing.md](../solana-dev/testing.md))
- **SDK**: Integration tests with Helius/Streamflow/Switchboard
- **Two-strike rule**: If test fails twice, STOP and ask

### 5. Deliverables

When implementing changes, provide:
- Exact files changed with clear diffs
- Package dependencies (Cargo.toml, package.json)
- Build/test commands
- Deployment considerations (devnet vs mainnet)
- Cost estimates (rent, tx fees per reward cycle)

---

## Progressive Disclosure (Read When Needed)

### DePIN-Specific Skills (This Addon)

#### DePIN Fundamentals
- [overview.md](overview.md) — DePIN landscape, why Solana, architectural patterns, real-world project case studies

#### Token & Rewards
- [token-rewards.md](token-rewards.md) — Token mechanics: SPL Token 2022, inflation models (burn-and-mint, SaaS buyback), emission schedule design, reward formulas
- [reward-distribution.md](reward-distribution.md) — Operator payouts: Streamflow streaming/vesting/staking, Merkle distributor contracts, ZK compressed airdrops

#### Verification & Data
- [proof-mechanisms.md](proof-mechanisms.md) — Proof of Location (Helium PoC, H3 hex grids), Proof of Work/Compute (Nosana, Render), Proof of Bandwidth/Data (UpRock, Hivemapper consensus), challenge-response protocols
- [oracle-integration.md](oracle-integration.md) — Pyth price feeds, Switchboard custom feeds + VRF, geospatial oracles, custom oracle design

#### Device Management
- [device-identity.md](device-identity.md) — Compressed PDA registry, anti-sybil (TEEs, stake gates, trust scores), session keys, compressed NFT identity
- [zk-compression.md](zk-compression.md) — Helius ZK Compression APIs, batch commit patterns, cost modeling for 100K–1M devices

#### Operations
- [staking-governance.md](staking-governance.md) — Staking pools, slashing conditions, Realms DAO, Squads multisig, parameter governance
- [indexing-operator-onboarding.md](indexing-operator-onboarding.md) — Helius webhooks/DAS API, real-time dashboards, embedded wallets (auto-generated, session-based, email login)

### Core Solana Dev Skills (from solana-dev-skill)

> These are provided by [solana-dev-skill](../solana-dev/SKILL.md) — install if not present

#### Web Frontend
- [frontend-framework-kit.md](../solana-dev/frontend-framework-kit.md) — React hooks, wallet connection
- [kit-web3-interop.md](../solana-dev/kit-web3-interop.md) — Kit ↔ web3.js boundary patterns

#### Program Development
- [programs-anchor.md](../solana-dev/programs-anchor.md) — Anchor framework patterns
- [programs-pinocchio.md](../solana-dev/programs-pinocchio.md) — Pinocchio high-performance programs
- [idl-codegen.md](../solana-dev/idl-codegen.md) — IDL generation and client codegen

#### Core Testing & Security
- [testing.md](../solana-dev/testing.md) — LiteSVM, Mollusk, Surfpool
- [security.md](../solana-dev/security.md) — Security checklist (programs + clients)
- [payments.md](../solana-dev/payments.md) — Core payment patterns
- [resources.md](../solana-dev/resources.md) — Core Solana resources

---

## Task Routing Guide

| User asks about... | Primary skill file(s) |
|--------------------|----------------------|
| What is DePIN? | overview.md |
| How does Helium work? | overview.md, proof-mechanisms.md |
| Design tokenomics | token-rewards.md |
| Inflation model | token-rewards.md |
| Burn-and-mint | token-rewards.md |
| Reward operators daily | reward-distribution.md, token-rewards.md |
| Streamflow streaming | reward-distribution.md |
| Merkle distributor | reward-distribution.md |
| Compressed airdrop | reward-distribution.md, zk-compression.md |
| Proof of Location | proof-mechanisms.md |
| Proof of Work | proof-mechanisms.md |
| Challenge protocol | proof-mechanisms.md, oracle-integration.md |
| Pyth price feed | oracle-integration.md |
| Switchboard oracle | oracle-integration.md |
| VRF randomness | oracle-integration.md |
| Device registry | device-identity.md |
| Anti-sybil | device-identity.md |
| Session keys | device-identity.md |
| Compressed device accounts | zk-compression.md |
| State compression | zk-compression.md |
| ZK Compression costs | zk-compression.md |
| Operator staking | staking-governance.md |
| Slashing | staking-governance.md |
| DAO governance | staking-governance.md |
| Squads multisig | staking-governance.md |
| Helius webhooks | indexing-operator-onboarding.md |
| DAS API | indexing-operator-onboarding.md |
| Embedded wallet | indexing-operator-onboarding.md |
| **Anchor program** | solana-dev → programs-anchor.md |
| **Program testing** | solana-dev → testing.md |
| **Security review** | solana-dev → security.md |

---

## Commands

| Command | Description |
|---------|-------------|
| /design-depin-tokenomics | Design token economics for a DePIN project |
| /setup-reward-stream | Configure Streamflow reward streaming for operators |
| /generate-device-registry | Generate compressed device registry Anchor program |

## Agents

| Agent | Purpose |
|-------|---------|
| **depin-architect** | DePIN architecture, tokenomics, system design |
| **tech-docs-writer** | README files, API docs, integration guides |
