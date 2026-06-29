# ZK Compression for DePIN at Scale

ZK Compression is a **game-changer for DePIN**. It reduces the cost of maintaining on-chain account state by 10-100×, making it economically viable to have millions of device accounts on-chain.

## The Problem

Each device in a DePIN needs an on-chain account for:
- Device registration and identity
- Reward tracking and balances
- State (location, firmware, trust score)
- Heartbeat and activity

**Uncompressed costs:**

| Device Count | Rent Cost (0.002 SOL/account) | Annual Rent |
|-------------|------------------------------|-------------|
| 1,000 | 2 SOL | ~$400 |
| 10,000 | 20 SOL | ~$4,000 |
| 100,000 | 200 SOL | ~$40,000 |
| 1,000,000 | 2,000 SOL | ~$400,000 |

**Compressed costs (ZK Compression via Helius):**

| Device Count | Rent Cost (~0.00002 SOL/account) | Annual Rent |
|-------------|--------------------------------|-------------|
| 1,000 | 0.02 SOL | ~$4 |
| 10,000 | 0.2 SOL | ~$40 |
| 100,000 | 2 SOL | ~$400 |
| 1,000,000 | 20 SOL | ~$4,000 |

## How ZK Compression Works

ZK Compression uses zero-knowledge proofs to maintain account state off-chain while verifiability on-chain.

```
Traditional Account:
┌──────────────────────────────────┐
│  Account                        │  Stored directly on Solana
│  - Device ID                    │  Cost: 0.002 SOL/account
│  - Owner                         │
│  - Balance                       │
│  - State                         │
└──────────────────────────────────┘

ZK Compressed Account:
┌──────────────────────────────────┐
│  Merkle Tree Root                │  Stored on Solana (tiny)
│  (state commitment)              │  Cost: ~0.00002 SOL/account
├──────────────────────────────────┤
│                                  │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐      │
│  │ D1│ │ D2│ │ D3│ │ D4│ ...   │  Off-chain (indexer/RPC)
│  └───┘ └───┘ └───┘ └───┘      │
│  (compressed account states)     │
└──────────────────────────────────┘
```

## Helius ZK Compression Setup

Helius (which acquired Light Protocol in 2025) now provides ZK Compression as a managed service.

### Installation

```bash
npm install @lightprotocol/stateless.js @solana/web3.js
```

### 1. Create a Compressed Mint for Rewards

```typescript
import { LightSystemProgram, createRpc } from '@lightprotocol/stateless.js';
import { Connection, Keypair, PublicKey } from '@solana/web3.js';

const connection = new Connection('https://mainnet.helius-rpc.com/?api-key=YOUR_KEY');
const rpc = createRpc(connection.rpcEndpoint);
const authority = Keypair.generate();

// Create compressed token mint
async function createCompressedRewardMint() {
  const mint = await LightSystemProgram.createCompressedMint({
    rpc,
    authority: authority.publicKey,
    payer: authority,
    // Optional: configure mint parameters
    decimals: 9,
  });

  console.log(`Compressed mint created: ${mint}`);
  return mint;
}
```

### 2. Compress Device State

```typescript
interface CompressedDeviceState {
  deviceId: string;
  owner: PublicKey;
  locationHash: number[];
  registeredAt: number;
  lastHeartbeat: number;
  totalRewards: number;
  trustScore: number;
}

async function registerCompressedDevice(
  device: CompressedDeviceState,
  authority: Keypair,
) {
  // Compress device state into a Merkle tree
  const compressedTx = await LightSystemProgram.compress({
    rpc,
    payer: authority,
    outputAccounts: [
      {
        // Compressed PDA for this device
        pubkey: deriveDevicePda(device.deviceId),
        owner: authority.publicKey,
        lamports: 0,  // No rent! (well, minimal)
        data: serializeDeviceState(device),
      },
    ],
  });

  console.log(`Device registered (compressed): ${compressedTx}`);
  return compressedTx;
}
```

### 3. Batch Update Device States

For DePINs with thousands of devices updating state simultaneously:

```typescript
async function batchUpdateHeartbeats(
  devices: Array<{
    deviceId: string;
    lastHeartbeat: number;
    rewardsEarned: number;
  }>,
  authority: Keypair,
) {
  const batchSize = 100; // Max 100 state updates per transaction

  for (let i = 0; i < devices.length; i += batchSize) {
    const batch = devices.slice(i, i + batchSize);

    const tx = await LightSystemProgram.compress({
      rpc,
      payer: authority,
      outputAccounts: batch.map(d => ({
        pubkey: deriveDevicePda(d.deviceId),
        owner: authority.publicKey,
        lamports: 0,
        data: serializeHeartbeatUpdate(d),
      })),
    });

    console.log(`Batch ${i / batchSize + 1} updated: ${tx}`);
  }
}
```

### 4. Query Compressed Device State

```typescript
import { getCompressedAccount } from '@lightprotocol/stateless.js';

async function getDeviceState(deviceId: string) {
  const pda = deriveDevicePda(deviceId);

  // Query compressed account via Helius DAS API
  const account = await getCompressedAccount(rpc, pda);

  if (!account) {
    return null; // Device not found
  }

  // Deserialize the compressed state
  return deserializeDeviceState(account.data);
}
```

## Cost Modeling

```typescript
interface CompressionCostModel {
  deviceCount: number;
  updatesPerDay: number;      // Average state updates per device per day
  batchSize: number;           // Max state updates per tx
  solPrice: number;            // SOL/USD price
}

function calculateCompressionCosts(model: CompressionCostModel) {
  const {
    deviceCount,
    updatesPerDay,
    batchSize,
    solPrice,
  } = model;

  // Initial compression cost
  const accountsPerBatch = Math.min(batchSize, 100);
  const batchesNeeded = Math.ceil(deviceCount / accountsPerBatch);
  const initCostSol = batchesNeeded * 0.00001; // ~0.00001 SOL per batch
  const initCostUsd = initCostSol * solPrice;

  // Daily update costs
  const dailyUpdates = deviceCount * updatesPerDay;
  const dailyBatches = Math.ceil(dailyUpdates / accountsPerBatch);
  const dailyCostSol = dailyBatches * 0.00001;
  const dailyCostUsd = dailyCostSol * solPrice;

  // Annual costs
  const annualCostSol = initCostSol + dailyCostSol * 365;
  const annualCostUsd = annualCostSol * solPrice;

  return {
    initialSetup: { sol: initCostSol.toFixed(6), usd: initCostUsd.toFixed(2) },
    dailyOperations: { sol: dailyCostSol.toFixed(6), usd: dailyCostUsd.toFixed(2) },
    annualTotal: { sol: annualCostSol.toFixed(6), usd: annualCostUsd.toFixed(2) },
  };
}

// Example: 100K devices, 1 heartbeat/day, 100 updates/tx
const costs = calculateCompressionCosts({
  deviceCount: 100_000,
  updatesPerDay: 1,
  batchSize: 100,
  solPrice: 200,
});
// Result: ~$8/year for 100K devices
```

## Compressed vs Uncompressed Decision Matrix

| Factor | Uncompressed | Compressed (ZK) |
|--------|-------------|-----------------|
| **Cost per account** | 0.002 SOL | ~0.00002 SOL |
| **Read speed** | Instant (direct account fetch) | RPC-dependent (proof verification) |
| **Write complexity** | Simple (standard CPI) | Batch + proof generation |
| **Data size limit** | 10KB | ~512 bytes per compressed account |
| **Tooling maturity** | Mature | Maturing (Helius managed) |
| **Best for** | <1K devices, complex state | 10K+ devices, simple state |

## When to Use Compression in DePIN

### ✅ Use ZK Compression for:
- Device registration state (ID, owner, location hash)
- Reward tracking per device
- Heartbeat timestamps
- Trust scores
- Token balances for micro-rewards

### ❌ Don't compress for:
- Complex program state that needs frequent on-chain reads
- Treasury accounts and vaults
- Governance state (proposals, votes)
- Oracle feeds and aggregator state

## Hybrid Approach

For most DePINs, a hybrid approach works best:

```
┌─────────────────────────────────────────────────────────────┐
│              Hybrid State Architecture                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Compressed (ZK):               Uncompressed:                │
│  ┌──────────────────────┐      ┌──────────────────────┐     │
│  │ ~100K Device Accounts │      │ Protocol Treasury     │     │
│  │ Device ID            │      │ Token Mint            │     │
│  │ Owner                │      │ Governance State      │     │
│  │ Location Hash        │      │ Oracle Feed           │     │
│  │ Trust Score          │      │ Distributor Root      │     │
│  │ Last Heartbeat       │      │ Staking Pool          │     │
│  │ Reward Earned (total)│      │                      │     │
│  └──────────────────────┘      └──────────────────────┘     │
│                                                              │
│  Cost: ~2 SOL initial                Cost: ~0.1 SOL          │
│       + ~0.5 SOL/year                                        │
└─────────────────────────────────────────────────────────────┘
```

## Security Considerations

1. **Proof verification**: Always verify ZK proofs on-chain before accepting compressed state updates.
2. **Indexer dependency**: Compressed state depends on indexers. Use Helius managed infrastructure for reliability.
3. **Proof generation**: Generating proofs is computationally intensive. Budget for off-chain proof generation servers.
4. **State synchronization**: Compressed and uncompressed state must stay in sync. Use atomic operations when possible.

## Related

- [device-identity.md](device-identity.md) — Compressed device registry design
- [reward-distribution.md](reward-distribution.md) — Compressed airdrops for mass distribution
- [indexing-operator-onboarding.md](indexing-operator-onboarding.md) — Querying compressed state via DAS API
