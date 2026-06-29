# Proof Mechanisms for DePIN

The core challenge in DePIN: **how does the network verify that a device is honestly contributing, without centralized trust?** The proof mechanism determines the network's security model, cost structure, and trust assumptions.

## Proof Types Compared

| Mechanism | What It Proves | Trust Model | Cost | Used By |
|-----------|---------------|-------------|------|---------|
| Proof of Coverage (PoC) | Location + wireless signal | Cryptographic challenge | High (tx per challenge) | Helium |
| Proof of Work | Useful computation completed | Result verification | Medium (per job) | Nosana, Render |
| Proof of Bandwidth | Network throughput | Measurement | Low (heartbeat) | UpRock |
| Multi-Device Consensus | Data observation matches | Statistical | Low (batch) | Hivemapper |
| Proof of Connection | Physical device connectivity | Hardware-level | Low (one-time) | DIMO |
| TEE Attestation | Code runs in trusted environment | Hardware (SGX) | Medium | Switchboard |

## 1. Proof of Coverage (Helium Model)

Best for: **Wireless networks** where devices must prove physical location and radio coverage.

### How It Works

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Challenger│     │ Beaconer │     │ Witnesses│
│ (randomly │ ──► │ (target  │ ──► │ (nearby  │
│  selected)│     │  device) │     │  devices)│
└──────────┘     └──────────┘     └──────────┘
      │               │               │
      │  1. Challenge  │               │
      │ ──────────────►│               │
      │                │  2. Beacon    │
      │                │ ─────────────►│
      │                │               │
      │  3. Witness receipts           │
      │ ◄─────────────────────────────│
      │                │               │
      ▼                ▼               ▼
   ┌──────────────────────────────────────┐
   │          On-Chain Verification        │
   │  - Challenger issued challenge        │
   │  - Beaconer responded (location)      │
   │  - N witnesses confirmed signal       │
   │  - Reward distributed if valid        │
   └──────────────────────────────────────┘
```

### Anchor Program: Challenge-Response

```rust
use anchor_lang::prelude::*;
use switchboard_vrf::program::SwitchboardVRF;

#[account]
pub struct Challenge {
    pub challenger: Pubkey,
    pub target_device: Pubkey,
    pub challenge_id: [u8; 32],
    pub issued_at: i64,
    pub expires_at: i64,
    pub resolved: bool,
    pub witnesses: Vec<Pubkey>,
}

pub fn issue_challenge(
    ctx: Context<IssueChallenge>,
    challenge_id: [u8; 32],
) -> Result<()> {
    let challenge = &mut ctx.accounts.challenge;
    let now = Clock::get()?.unix_timestamp;

    challenge.challenger = ctx.accounts.challenger.key();
    challenge.target_device = ctx.accounts.target_device.key();
    challenge.challenge_id = challenge_id;
    challenge.issued_at = now;
    challenge.expires_at = now + 300; // 5 min to respond
    challenge.resolved = false;
    challenge.witnesses = vec![];

    Ok(())
}

pub fn submit_witness(
    ctx: Context<SubmitWitness>,
    signal_strength: i16,  // dBm
) -> Result<()> {
    let challenge = &mut ctx.accounts.challenge;
    require!(!challenge.resolved, ErrorCode::ChallengeResolved);
    require!(
        Clock::get()?.unix_timestamp < challenge.expires_at,
        ErrorCode::ChallengeExpired
    );

    // Verify witness is within range of target
    let distance = calculate_distance(
        &ctx.accounts.witness_device.location_hash,
        &ctx.accounts.target_device.location_hash,
    );
    require!(distance < MAX_WITNESS_RANGE, ErrorCode::OutOfRange);

    // Signal strength must be physically plausible
    require!(
        signal_strength > MIN_SIGNAL_THRESHOLD,
        ErrorCode::SignalTooWeak
    );

    challenge.witnesses.push(ctx.accounts.witness.key());
    Ok(())
}

pub fn resolve_challenge(ctx: Context<ResolveChallenge>) -> Result<()> {
    let challenge = &mut ctx.accounts.challenge;
    require!(!challenge.resolved, ErrorCode::ChallengeResolved);

    let witness_count = challenge.witnesses.len() as u8;

    // Reward based on witness count and signal quality
    if witness_count >= MIN_WITNESSES {
        // Challenge passed — device is where it claims
        let reward = calculate_poc_reward(witness_count, challenge.target_device);
        distribute_reward(&ctx, reward)?;
    } else {
        // Challenge failed — possible spoof
        apply_penalty(&ctx, challenge.target_device)?;
    }

    challenge.resolved = true;
    Ok(())
}
```

### Location Hashing

```rust
// Location is asserted at device registration
// Not the raw GPS — a commitment to location
pub fn assert_location(
    ctx: Context<AssertLocation>,
    latitude: i64,
    longitude: i64,
    accuracy: u8,  // meters
) -> Result<()> {
    let device = &mut ctx.accounts.device;

    // Hash location for privacy
    let location_bytes = [
        latitude.to_le_bytes(),
        longitude.to_le_bytes(),
        accuracy.to_le_bytes(),
    ].concat();
    device.location_hash = anchor_lang::solana_program::keccak::hash(&location_bytes).to_bytes();
    device.location_asserted_at = Clock::get()?.unix_timestamp;

    Ok(())
}
```

## 2. Proof of Work / Compute (Nosana & Render Model)

Best for: **Compute networks** where devices perform computational work.

### Nosana Pattern: Container-Based Workload

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Requester   │     │   Operator   │     │   Verifier   │
│  (submits     │ ──► │  (runs job   │ ──► │  (validates  │
│   workload)   │     │   container) │     │   result)    │
└──────────────┘     └──────────────┘     └──────────────┘
      │                                        │
      │  1. Submit job spec                    │
      │ ──────────────────────────────────────►│
      │                                        │
      │           2. Match operator            │
      │ ◄──────────────────────────────────────│
      │                                        │
      │                                        │
      │           3. Operator runs container   │
      │              Result hash submitted     │
      │ ◄──────────────────────────────────────│
      │                                        │
      │  4. Verify result                      │
      │ ──────────────────────────────────────►│
      │                                        │
      │  5. Release payment                    │
      │ ──────────────────────────────────────►│
```

### Anchor Program: Compute Verification

```rust
pub fn submit_compute_result(
    ctx: Context<SubmitResult>,
    job_id: [u8; 32],
    result_hash: [u8; 32],
    compute_units: u64,
) -> Result<()> {
    let job = &mut ctx.accounts.job;

    // Verify operator was assigned this job
    require!(job.assigned_operator == ctx.accounts.operator.key(), ErrorCode::NotAssigned);

    // Record result
    job.result_hash = result_hash;
    job.completed_at = Clock::get()?.unix_timestamp;
    job.compute_units_consumed = compute_units;

    // Calculate reward based on compute
    let reward = compute_units * PRICE_PER_CU;
    escrow_reward(&ctx, reward)?;

    Ok(())
}

// Challenge: Another operator re-runs to verify
pub fn challenge_result(
    ctx: Context<ChallengeResult>,
    job_id: [u8; 32],
    my_result_hash: [u8; 32],
) -> Result<()> {
    let job = &ctx.accounts.job;

    require!(job.requires_verification, ErrorCode::NotRequired);

    if my_result_hash != job.result_hash {
        // Mismatch — flag for resolution
        job.disputed = true;
        job.disputed_at = Clock::get()?.unix_timestamp;
    }

    Ok(())
}
```

## 3. Proof of Bandwidth (UpRock Model)

Best for: **Bandwidth sharing networks** where devices contribute network throughput.

```rust
// Heartbeat with bandwidth measurement
pub fn submit_bandwidth_proof(
    ctx: Context<SubmitBandwidth>,
    upload_mbps: u32,
    download_mbps: u32,
    connection_type: ConnectionType,  // 4G, 5G, WiFi, Ethernet
    session_duration: u64,           // seconds
) -> Result<()> {
    let device = &mut ctx.accounts.device;

    // Score based on bandwidth quality
    let speed_score = calculate_speed_score(upload_mbps, download_mbps);
    let reliability_score = calculate_reliability_score(device.history);

    device.last_heartbeat = Clock::get()?.unix_timestamp;
    device.total_bandwidth_contributed = device.total_bandwidth_contributed
        .checked_add(upload_mbps as u64 + download_mbps as u64)
        .unwrap();

    // Reward proportional to contribution
    let reward = calculate_bandwidth_reward(
        speed_score,
        reliability_score,
        session_duration,
    );

    distribute_reward(&ctx, reward)?;
    Ok(())
}
```

## 4. Multi-Device Consensus (Hivemapper Model)

Best for: **Data collection networks** where multiple independent devices observe the same real-world phenomenon.

### H3 Geospatial Hex Grid

Hivemapper uses Uber's H3 hex grid system for fine-grained contribution tracking:

```typescript
import h3 from 'h3-js';

// Convert GPS coordinates to H3 hex cell
const cell = h3.geoToH3(latitude, longitude, resolution); // resolution 15 = ~1m²

// Reward all devices that contributed imagery to this hex
const devices = await getDevicesInHex(cell);
const uniqueDevices = new Set(devices.map(d => d.deviceId));

// Multi-device bonus: more independent observations = higher confidence
if (uniqueDevices.size >= 3) {
  // Valid data — reward all contributing devices
  await distributeHexReward(cell, uniqueDevices.size);
}
```

### Anchor Program: Consensus Reward

```rust
pub fn submit_data_point(
    ctx: Context<SubmitData>,
    hex_cell: u64,
    data_hash: [u8; 32],
    device_id: [u8; 32],
) -> Result<()> {
    let consensus = &mut ctx.accounts.consensus_state;

    // Track unique contributions per hex
    let entry = consensus.entries.iter_mut()
        .find(|e| e.hex_cell == hex_cell);

    match entry {
        Some(e) => {
            if !e.devices.contains(&device_id) {
                e.devices.push(device_id);
                e.unique_device_count = e.devices.len() as u8;
            }
        }
        None => {
            consensus.entries.push(ConsensusEntry {
                hex_cell,
                devices: vec![device_id],
                unique_device_count: 1,
                data_hash,
                submitted_at: Clock::get()?.unix_timestamp,
            });
        }
    }

    Ok(())
}
```

## 5. Proof of Connection (DIMO Model)

Best for: **Devices that physically connect** — vehicles, sensors, appliances.

```rust
// Hardware-level connection verification
pub fn register_device_connection(
    ctx: Context<RegisterConnection>,
    hardware_signature: [u8; 64],   // Signed by hardware secure element
    firmware_version: [u8; 8],
    can_bus_snapshot: Vec<u8>,       // Vehicle CAN bus data
) -> Result<()> {
    let device = &mut ctx.accounts.device;

    // Verify hardware signature
    require!(
        verify_hardware_signature(&hardware_signature, &device.public_key),
        ErrorCode::InvalidSignature
    );

    // Verify CAN bus data is realistic (anti-spoof)
    require!(
        validate_can_bus_data(&can_bus_snapshot),
        ErrorCode::InvalidData
    );

    device.connected = true;
    device.last_connection = Clock::get()?.unix_timestamp;
    device.firmware = firmware_version;

    Ok(())
}
```

## 6. Challenge-Response Protocol Design

The most robust DePINs use cryptographic challenge-response protocols. This is the **hardest pattern to get right** but provides the strongest guarantees.

### Design Principles

1. **Random selection**: Challengers chosen via VRF (verifiable random function)
2. **Time-bounded**: Challenges expire (prevents pre-computation)
3. **Cost-sensitive**: Challenge cost must be < operator reward (economic viability)
4. **Multi-witness**: Multiple independent witnesses prevent collusion
5. **Graduated response**: First failure = warning, repeated = slashing

### VRF-Based Challenge Selection

```typescript
import SwitchboardVRF from '@switchboard-xyz/vrf';

// Generate random challenge using Switchboard VRF
async function selectChallenger(
  deviceRegistry: PublicKey[],
  vrfAccount: PublicKey,
): Promise<{ index: number; proof: number[] }> {
  const vrf = new SwitchboardVRF({ connection, vrfAccount });

  // Request randomness
  const randomness = await vrf.requestRandomness({
    authority: protocolWallet,
  });

  // Wait for fulfillment
  const result = await randomness.waitForFulfillment();

  // Use randomness to select challenger
  const randomValue = result.toBigInt();
  const index = Number(randomValue % BigInt(deviceRegistry.length));

  return { index, proof: result.proof };
}
```

## Security Considerations

1. **Sybil resistance**: All proof mechanisms are vulnerable to sybil attacks. Always combine with stake requirements.
2. **Economic bounds**: Proof cost must be < operator profit margin. If proving is too expensive, honest operators are unprofitable.
3. **Time synchronization**: Challenge-response relies on accurate clocks. Use on-chain timestamps, not device-reported times.
4. **Front-running**: Challenge selection via VRF prevents prediction. Don't reveal challenge targets before the challenge period.
5. **Collusion**: Multi-witness systems are vulnerable to colluding device clusters. Require geographic diversity in witnesses.

## Related

- [oracle-integration.md](oracle-integration.md) — VRF for randomness, custom oracles for geo-data
- [device-identity.md](device-identity.md) — Device registration, anti-sybil foundations
- [token-rewards.md](token-rewards.md) — Reward formulas that factor in proof quality
