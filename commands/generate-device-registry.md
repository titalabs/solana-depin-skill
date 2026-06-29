---
description: "Generate a compressed device registry Anchor program for DePIN"
---

You are generating a compressed device registry Anchor program for a Solana DePIN. Follow these steps:

## Step 1: Gather Requirements

Ask the user about:
- **Target scale**: How many devices will be registered?
- **Device fields**: What data does each device store? (ID, location, firmware, trust score, etc.)
- **Operator identity**: Wallet-based? Session key? Compressed NFT?
- **Anti-sybil**: Stake requirement? Hardware attestation? Trust scoring?
- **Update frequency**: How often do devices send heartbeats?

## Step 2: Install Dependencies

```bash
# Anchor program
anchor init device-registry
cd device-registry
npm install @lightprotocol/stateless.js @solana/web3.js

# For Helius RPC
# Add to Anchor.toml:
# [provider]
# cluster = "mainnet"
# wallet = "~/.config/solana/id.json"
```

## Step 3: Generate Device Registry Program

```rust
// programs/device-registry/src/lib.rs
use anchor_lang::prelude::*;
use helius_zk_compression::cpi::accounts::Compress;

declare_id!("YOUR_PROGRAM_ID");

#[account]
pub struct DeviceRegistry {
    pub authority: Pubkey,
    pub device_count: u64,
    pub mint: Option<Pubkey>,
    pub min_stake: u64,
    pub challenge_interval: i64,
    pub bump: u8,
}

#[account]
pub struct DeviceAccount {
    pub device_id: Pubkey,
    pub owner: Pubkey,
    pub location_hash: [u8; 32],
    pub registered_at: i64,
    pub last_heartbeat: i64,
    pub total_rewards_earned: u64,
    pub trust_score: u8,
    pub is_active: bool,
    pub bump: u8,
}
```

### Instructions

- `initialize_registry` — Create the registry
- `register_device` — Register a new device (with stake verification)
- `update_heartbeat` — Update device state via ZK Compression
- `deactivate_device` — Remove a device (slash stake if fraudulent)

## Step 4: Generate Compression Layer

```typescript
// scripts/compress-devices.ts
import { LightSystemProgram, createRpc } from '@lightprotocol/stateless.js';

// See skill/zk-compression.md for full compression patterns
```

## Step 5: Generate Tests

```rust
// tests/device-registry.ts
import * as anchor from '@coral-xyz/anchor';
// ... test each instruction with compressed state
```

## Step 6: Deliverables

- Complete Anchor program (`programs/device-registry/`)
- Compression integration script (`scripts/compress-devices.ts`)
- Test suite (`tests/device-registry.ts`)
- Deployment instructions
- Cost estimate (rent per device compressed vs uncompressed)
