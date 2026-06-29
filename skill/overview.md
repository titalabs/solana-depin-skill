# DePIN on Solana — Overview

## What is DePIN?

Decentralized Physical Infrastructure Networks (DePIN) are blockchain-coordinated networks of physical devices that provide real-world services. Participants contribute hardware (sensors, wireless hotspots, GPUs, cameras, vehicles) and earn token rewards for their contribution.

Solana is the **dominant chain for DePIN** — the SVM's low fees, high throughput, and parallel execution make it uniquely suited for networks with thousands of devices producing continuous data and transactions.

## Major Solana DePIN Projects & Their Patterns

| Project | Category | Token | Proof Mechanism | Unique Pattern |
|---------|----------|-------|-----------------|----------------|
| **Helium** | Wireless (IoT + 5G) | HNT, IOT, MOBILE | Proof of Coverage (challenge-response) | Burn-and-mint equilibrium, subDAO tokens |
| **Hivemapper** | Mapping | HONEY | Vision-based multi-device consensus | H3 geospatial hex rewards, consumption rewards |
| **Render Network** | GPU Rendering | RNDR | Proof of Render (frame validation) | OCTANE benchmarking, validator nodes |
| **Nosana** | GPU Compute (AI) | NOS | Proof of Work (container execution) | Container-based workload marketplace |
| **GEODNET** | GPS Correction | GEOD | Proof of Location (RTK station) | Rooftop station placement, cm-precision GPS |
| **DIMO** | Vehicle Data | DIMO | Proof of Connection (CAN bus) | API-first device network, granular data permissions |
| **UpRock** | Bandwidth/Compute | UPT | Proof of Bandwidth (proxy-based) | SaaS revenue buyback, mobile-first mining |
| **ShdwDrive** | Storage | SHDW | — | SPL-based object storage with CDN distribution |

## Common DePIN Architectural Patterns

### 1. Token Incentive Layer
Every DePIN needs a token that rewards device operators and aligns incentives. The key design choices:

- **Inflation model**: Fixed supply? Inflationary with emission schedule? Burn-and-mint equilibrium?
- **Reward formula**: Per-device? Per-data-unit? Per-time? Quality-weighted?
- **Distribution mechanism**: Direct transfer? Streamed? Merkle drop? Compressed drop?

See [token-rewards.md](token-rewards.md) and [reward-distribution.md](reward-distribution.md).

### 2. Proof Mechanism
How does the network verify that devices are honestly contributing? The mechanism determines trust assumptions and cost:

- **Proof of Location**: Cryptographic challenge-response (Helium), GPS + WiFi triangulation (GEODNET)
- **Proof of Work/Compute**: Validate work output (Nosana containers, Render frames)
- **Proof of Bandwidth**: Measure actual network throughput (UpRock)
- **Proof of Data**: Multi-device consensus on observations (Hivemapper)
- **Proof of Connection**: Hardware-level connection verification (DIMO CAN bus)

See [proof-mechanisms.md](proof-mechanisms.md).

### 3. Oracle Integration
DePINs need real-world data on-chain:
- Price feeds for reward calculations
- Weather, traffic, or location data for reward logic
- Randomness for challenge protocols
- Custom data feeds for project-specific needs

See [oracle-integration.md](oracle-integration.md).

### 4. Device Identity at Scale
With thousands to millions of devices, on-chain storage costs matter:
- Each device needs a state account → rent cost
- Solution: ZK Compression reduces costs by 10-100×
- Device registry design: compressed PDAs vs uncompressed accounts

See [device-identity.md](device-identity.md) and [zk-compression.md](zk-compression.md).

### 5. Operator Economics
- Staking requirements for operators (skin in the game)
- Slashing conditions for misbehavior
- Trust scores and gradual reputation building
- Governance for parameter adjustments

See [staking-governance.md](staking-governance.md).

### 6. Indexing & Operator UX
- Real-time monitoring of device state
- Webhooks for reward distribution triggers
- Embedded wallets for non-crypto-native operators
- Performance dashboards

See [indexing-operator-onboarding.md](indexing-operator-onboarding.md).

## Architecture Decision Flow

```
┌─────────────────────────────────────────────┐
│       DePIN Architecture Decisions          │
├─────────────────────────────────────────────┤
│                                              │
│  1. What physical resource is shared?        │
│     └─ Wireless? Compute? Data? Storage?     │
│                                              │
│  2. How to verify honest contribution?       │
│     └─ PoL · PoW · PoB · Consensus          │
│                                              │
│  3. Token model?                             │
│     └─ Inflationary · Fixed · BME · Buyback  │
│                                              │
│  4. Reward distribution mechanism?           │
│     └─ Stream · Merkle · Compressed          │
│                                              │
│  5. Device identity approach?                │
│     └─ Compressed PDA · NFT · Off-chain      │
│                                              │
│  6. Staking & governance?                    │
│     └─ Required · Optional · None            │
│                                              │
│  7. Oracle requirements?                     │
│     └─ Prices only · Custom data · VRF       │
│                                              │
│  8. Scale targets?                           │
│     └─ <1K · 10K · 100K · 1M+ devices       │
└─────────────────────────────────────────────┘
```

## When to Use This Skill vs. Other Skills

| If the user wants... | Route to... |
|---------------------|-------------|
| Build a DePIN program | This skill (tokenomics, proof, identity) + solana-dev (Anchor/Pinocchio) |
| General Solana program | solana-dev → programs-anchor.md |
| DeFi protocol | sendai DeFi skills |
| NFT project | metaplex skill |
| Security audit | trailofbits skills |
| Game development | solana-game-skill |
