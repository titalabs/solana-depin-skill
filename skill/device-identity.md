# Device Identity for DePIN

Managing device identity at scale is one of the hardest problems in DePIN. Each device needs an on-chain identity for reward tracking, proof verification, and state management. With 100K+ devices, naive account-per-device patterns cost thousands of SOL in rent.

## Device Registry Design


### Design Decisions

| Decision | Option A | Option B | Option C |
|----------|----------|----------|----------|
| **Storage** | Uncompressed PDA | Compressed PDA (ZK) | Off-chain (Merkle tree) |
| **Identity** | Wallet-based | Compressed NFT | Session key |
| **Proof** | On-chain verification | Oracle-verified | Challenge-based |
| **Cost/device** | 0.002 SOL | ~0.00002 SOL | ~0 SOL (tx cost only) |

**Recommendation**: Compressed PDAs via Helius ZK Compression for most DePINs. This gives you on-chain verifiability at a fraction of the cost.

### Anchor Program: Device Registry

```rust
use anchor_lang::prelude::*;
use anchor_lang::solana_program::keccak;

declare_id!("Dev1ceRegistry11111111111111111111111111111");

#[account]
#[derive(InitSpace)]
pub struct DeviceRegistry {
    pub authority: Pubkey,
    pub device_count: u64,
    pub mint: Option<Pubkey>,        // Optional: token for operator staking
    pub min_stake: u64,              // Minimum stake to register
    pub challenge_interval: i64,     // Seconds between challenges
    pub bump: u8,
}

#[account]
#[derive(InitSpace)]
pub struct DeviceAccount {
    pub device_id: Pubkey,           // Device's public key
    pub owner: Pubkey,               // Operator wallet
    pub location_hash: [u8; 32],     // Keccak256(lat, lng, accuracy)
    pub registered_at: i64,
    pub last_heartbeat: i64,
    pub total_rewards_earned: u64,
    pub trust_score: u8,             // 0-100
    pub challenge_count: u64,
    pub passed_challenges: u64,
    pub firmware_version: [u8; 8],
    pub is_active: bool,
    pub bump: u8,
}

pub fn register_device(
    ctx: Context<RegisterDevice>,
    device_id: Pubkey,
    location_hash: [u8; 32],
    firmware_version: [u8; 8],
) -> Result<()> {
    let registry = &mut ctx.accounts.registry;
    let device = &mut ctx.accounts.device;

    // Verify stake is sufficient
    if let Some(mint) = registry.mint {
        let stake_amount = ctx.accounts.operator_stake.amount;
        require!(stake_amount >= registry.min_stake, ErrorCode::InsufficientStake);
    }

    device.device_id = device_id;
    device.owner = ctx.accounts.operator.key();
    device.location_hash = location_hash;
    device.registered_at = Clock::get()?.unix_timestamp;
    device.last_heartbeat = Clock::get()?.unix_timestamp;
    device.trust_score = 50;  // Start at neutral
    device.firmware_version = firmware_version;
    device.is_active = true;
    device.bump = ctx.bumps.device;

    registry.device_count = registry.device_count.checked_add(1).unwrap();

    Ok(())
}
```

## Anti-Sybil Mechanisms

Sybil attacks — one person running thousands of fake devices to farm rewards — are the #1 vulnerability in DePIN. Layer multiple defenses:

### 1. Stake Requirements (First Line of Defense)

Operators must lock tokens to register a device. The cost of sybil becomes: `stake_per_device × fake_device_count`.

```rust
pub fn calculate_stake_requirement(
    trust_score: u8,
    base_stake: u64,
) -> u64 {
    // New devices stake more, trusted devices stake less
    match trust_score {
        0..=20 => base_stake * 5,     // 5x for unproven
        21..=50 => base_stake * 2,    // 2x for new
        51..=80 => base_stake,        // Standard
        81..=100 => base_stake / 2,   // 0.5x for trusted
        _ => base_stake,
    }
}
```

### 2. Hardware Attestation (Strong Defense)

Use TEEs (Trusted Execution Environments) to prove the device is real hardware:

```typescript
// Intel SGX / AMD SEV attestation verification
interface HardwareAttestation {
  provider: 'sgx' | 'sev' | 'nitro';
  quote: Buffer;          // TEE-signed quote
  publicKey: Buffer;      // Device's public key
  timestamp: number;
}

async function verifyAttestation(attestation: HardwareAttestation): Promise<boolean> {
  switch (attestation.provider) {
    case 'sgx':
      return verifyIntelSGXQuote(attestation.quote);
    case 'sev':
      return verifyAMDSEVQuote(attestation.quote);
    case 'nitro':
      return verifyAWSNitrox(attestation.quote);
  }
}
```

### 3. Gradual Trust Building (Operational Defense)

New devices earn trust over time through consistent behavior:

```rust
pub fn update_trust_score(
    device: &mut Account<DeviceAccount>,
    challenge_passed: bool,
) {
    if challenge_passed {
        // Gradual increase — hard to earn
        device.trust_score = (device.trust_score as u16
            .checked_add(1))
            .unwrap_or(100) as u8;
        device.passed_challenges = device.passed_challenges.checked_add(1).unwrap();
    } else {
        // Rapid decrease — easy to lose
        device.trust_score = device.trust_score
            .checked_sub(10)
            .unwrap_or(0);
    }

    // Dynamic reward multiplier based on trust
    let reward_multiplier = match device.trust_score {
        0..=20 => 0.0,       // No rewards until trust rebuilt
        21..=50 => 0.5,      // Half rewards
        51..=80 => 1.0,      // Full rewards
        81..=100 => 1.2,     // Bonus for trusted
    };
}
```

### 4. Device Fingerprinting (Preventive)

Collect and verify device characteristics that are hard to fake:

```rust
pub struct DeviceFingerprint {
    pub hardware_id: [u8; 32],     // TPM / secure element ID
    pub mac_address_hash: [u8; 32],
    pub firmware_hash: [u8; 32],
    pub boot_signature: [u8; 64],  // Signed by manufacturer
}

pub fn verify_device_uniqueness(
    ctx: Context<VerifyFingerprint>,
    fingerprint: DeviceFingerprint,
) -> Result<()> {
    // Check no existing device has the same hardware ID
    let existing = ctx.accounts.registry.devices.iter()
        .find(|d| d.fingerprint.hardware_id == fingerprint.hardware_id);
    require!(existing.is_none(), ErrorCode::DeviceAlreadyRegistered);
    Ok(())
}
```

## Session Keys for IoT Devices

IoT devices often can't hold Solana private keys securely. Use session keys — temporary, limited-privilege keys authorized by the operator.

### Session Key Pattern

```rust
#[account]
pub struct SessionKey {
    pub device: Pubkey,
    pub session_key: Pubkey,
    pub issued_at: i64,
    pub expires_at: i64,
    pub permissions: u8,  // Bitfield: read, write rewards, update state
    pub max_daily_rewards: u64,
    pub rewards_used_today: u64,
}

pub fn authorize_session_key(
    ctx: Context<AuthorizeSession>,
    session_key: Pubkey,
    duration: i64,
    permissions: u8,
    max_daily_rewards: u64,
) -> Result<()> {
    let session = &mut ctx.accounts.session;
    let now = Clock::get()?.unix_timestamp;

    session.device = ctx.accounts.device.key();
    session.session_key = session_key;
    session.issued_at = now;
    session.expires_at = now.checked_add(duration).unwrap();
    session.permissions = permissions;
    session.max_daily_rewards = max_daily_rewards;
    session.rewards_used_today = 0;

    Ok(())
}

// Device uses session key to submit heartbeat
pub fn submit_heartbeat_session(
    ctx: Context<HeartbeatSession>,
) -> Result<()> {
    let session = &ctx.accounts.session;
    let now = Clock::get()?.unix_timestamp;

    // Verify session is valid
    require!(now < session.expires_at, ErrorCode::SessionExpired);
    require!(
        ctx.accounts.session_authority.key() == session.session_key,
        ErrorCode::Unauthorized
    );

    // Check daily reward limit
    require!(
        session.rewards_used_today < session.max_daily_rewards,
        ErrorCode::DailyLimitExceeded
    );

    // Process heartbeat with reward
    let device = &mut ctx.accounts.device;
    device.last_heartbeat = now;
    Ok(())
}
```

## Compressed NFT Device Identity

For networks with 100K+ devices, consider using compressed NFTs (cNFTs) as device identity. Each device gets a unique cNFT that encodes its state.

```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { mintCompressedNft } from '@metaplex-foundation/umi';

const umi = createUmi('https://api.mainnet-beta.solana.com');

async function registerDeviceAsCNFT(
  deviceId: string,
  operatorWallet: PublicKey,
  metadata: {
    name: string;
    uri: string;  // JSON metadata with device attributes
  },
) {
  const cNft = await mintCompressedNft(umi, {
    owner: operatorWallet,
    uri: metadata.uri,
    name: metadata.name,
    sellerFeeBasisPoints: 0,
    compression: {
      // ZK compressed — minimal cost
      useZkCompression: true,
    },
  });

  return cNft;
}
```

## Device Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                  Device Lifecycle                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REGISTERED ───► ACTIVE ───► CHALLENGING ───► ACTIVE        │
│      │              │              │              │          │
│      │              │              │              │          │
│      ▼              ▼              ▼              ▼          │
│  Pending       Heartbeat      Challenge      Reward         │
│  Approval      Received       Verdict       Distributed     │
│                                                              │
│      │              │                                        │
│      ▼              ▼                                        │
│  REJECTED       INACTIVE                                      │
│  (stake        (missed heartbeats,                            │
│   returned)     slashed)                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Security Considerations

1. **Key management**: IoT devices lose keys. Use session keys with expiration and secure enclaves.
2. **Re-registration attacks**: Prevent de-registering and re-registering the same device to reset trust score.
3. **Location spoofing**: GPS can be spoofed. Combine with RF fingerprinting, Wi-Fi triangulation, and challenge-response.
4. **Firmware attacks**: Verify firmware hashes on registration and periodically.
5. **Stake draining**: Rate-limit device de-registration to prevent rapid stake withdrawal.

## Related

- [zk-compression.md](zk-compression.md) — Compressed device accounts at scale
- [proof-mechanisms.md](proof-mechanisms.md) — Challenge-response for device verification
- [oracle-integration.md](oracle-integration.md) — Location verification via oracles
- [staking-governance.md](staking-governance.md) — Operator staking requirements
