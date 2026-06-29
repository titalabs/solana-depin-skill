# Indexing & Operator Onboarding for DePIN

## Helius Webhooks for Real-Time Device Monitoring

Webhooks are essential for DePINs — they let your backend react to on-chain events in real time without polling.

### Webhook Event Types

| Event | When to Use | Example |
|-------|-------------|---------|
| `account` | Monitor specific device accounts | Track `last_heartbeat` changes |
| `program` | All instructions in your DePIN program | Catch every device interaction |
| `token` | Token transfers and mints | Detect reward distributions |
| `transaction` | Specific transaction patterns | Webhook on `claim_reward` pattern |

### Setting Up Helius Webhooks

```typescript
// Helius Webhook SDK
const webhookUrl = 'https://api.helius.xyz/v0/webhooks';

async function createDeviceWebhook(
  webhookId: string,
  programId: string,
  callbackUrl: string,
) {
  const response = await fetch(`${webhookUrl}?api-key=${HELIUS_API_KEY}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      webhookURL: callbackUrl,
      transactionTypes: ['Any'],
      accountAddresses: [programId],
      webhookType: 'rawDevnet',   // or 'rawMainnet'
      // Filter for specific instruction patterns
      // (e.g., only heartbeat submissions)
      txnStatus: 'success',
    }),
  });

  return response.json();
}
```

### Webhook Handler: Device Heartbeat Monitoring

```typescript
// Express.js webhook handler for device heartbeats
import express from 'express';
import { Connection, PublicKey } from '@solana/web3.js';

const app = express();
app.use(express.json());

interface HeartbeatEvent {
  deviceId: string;
  timestamp: number;
  trustScore: number;
  rewardsEarned: number;
}

app.post('/webhooks/device-heartbeat', async (req, res) => {
  const events: HeartbeatEvent[] = req.body;

  for (const event of events) {
    // Update internal database
    await db.collection('devices').updateOne(
      { deviceId: event.deviceId },
      {
        $set: {
          lastHeartbeat: event.timestamp,
          trustScore: event.trustScore,
          rewardsEarned: event.rewardsEarned,
        },
        $inc: { heartbeatCount: 1 },
      },
      { upsert: true },
    );

    // Alert if device is stale
    const now = Math.floor(Date.now() / 1000);
    if (now - event.timestamp > 3600) {
      await alerts.sendOperatorAlert(event.deviceId, 'Device heartbeat overdue');
    }
  }

  res.status(200).send('OK');
});
```

### Webhook Handler: Reward Distribution Trigger

```typescript
app.post('/webhooks/reward-distribution', async (req, res) => {
  const event = req.body;

  if (event.type === 'COMPRESSED_MINT' || event.type === 'TRANSFER') {
    // A reward was distributed — update dashboard
    await db.collection('rewards').insertOne({
      deviceId: event.accountData.deviceId,
      amount: event.accountData.amount,
      timestamp: event.accountData.timestamp,
      txSignature: event.signature,
    });

    // Push notification to operator
    await notifications.sendPush(
      event.accountData.operatorWallet,
      `Reward claimed: ${event.accountData.amount} tokens`,
    );
  }

  res.status(200).send('OK');
});
```

## Helius DAS API for Querying Device State

The Digital Asset Standard (DAS) API lets you query compressed and uncompressed assets efficiently.

### Query Devices by Owner

```typescript
const DAS_API = 'https://mainnet.helius-rpc.com/?api-key=YOUR_KEY';

async function getDevicesByOwner(operatorWallet: string) {
  const response = await fetch(DAS_API, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      id: 'devices-by-owner',
      method: 'getAssetsByOwner',
      params: {
        ownerAddress: operatorWallet,
        // Filter by your DePIN program
        grouping: ['program', depinProgramId],
        page: 1,
        limit: 1000,
      },
    }),
  });

  const data = await response.json();
  return data.result.items;
}
```

### Search Devices by Location (Geospatial)

```typescript
async function getDevicesInRadius(
  centerLat: number,
  centerLng: number,
  radiusKm: number,
) {
  const response = await fetch(DAS_API, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      id: 'devices-by-location',
      method: 'searchAssets',
      params: {
        // Search by attribute — requires consistent metadata
        ownerType: 'program',
        programId: depinProgramId,
        // Asset attributes stored in compressed metadata
        // Filter by location hex (H3 cell)
        // Requires devices to store their H3 index in metadata
        attributes: [
          { key: 'h3_cell', value: latLngToH3(centerLat, centerLng, 9) },
        ],
      },
    }),
  });

  return response.json();
}
```

### Real-Time Device Dashboard via WebSocket

```typescript
import WebSocket from 'ws';

function subscribeToDeviceUpdates(deviceIds: string[]) {
  const ws = new WebSocket(`wss://mainnet.helius-rpc.com/ws?api-key=${HELIUS_API_KEY}`);

  ws.on('open', () => {
    // Subscribe to account updates for all devices
    ws.send(JSON.stringify({
      jsonrpc: '2.0',
      id: 'device-sub',
      method: 'accountSubscribe',
      params: deviceIds.map(id => ({
        account: id,
        commitment: 'confirmed',
      })),
    }));
  });

  ws.on('message', (data) => {
    const update = JSON.parse(data.toString());
    // Update dashboard state
    dashboardStore.updateDevice(update.params.result);
  });

  return ws;
}
```

## Operator Onboarding UX

Most DePIN operators are **not crypto-native**. They're drivers, farmers, drone pilots, and homeowners. The onboarding UX must be dead simple.

### Embedded Wallet Patterns

#### 1. Auto-Generated Wallets (Best for Mobile Apps)

```typescript
import { Keypair } from '@solana/web3.js';
import { encrypt } from 'crypto-js/aes';

// When a user downloads the app, auto-generate a wallet
function onboardNewOperator(
  userId: string,
  deviceId: string,
) {
  // Generate a new Solana keypair
  const wallet = Keypair.generate();

  // Encrypt and store private key (user never sees it)
  const encryptedKey = encrypt(
    Buffer.from(wallet.secretKey).toString('base64'),
    userProvidedPIN,   // User picks a PIN
  );

  // Store encrypted key locally (or in iCloud Keychain)
  localStorage.setItem(`wallet_${userId}`, encryptedKey);

  // Register the device on-chain
  return registerDevice(deviceId, wallet.publicKey);
}

// Sign transactions without exposing the private key
function signWithEmbeddedWallet(
  userId: string,
  transaction: Transaction,
): Promise<Transaction> {
  const encrypted = localStorage.getItem(`wallet_${userId}`);
  const decrypted = decrypt(encrypted, userProvidedPIN);
  const secretKey = Buffer.from(decrypted.toString(), 'base64');
  const keypair = Keypair.fromSecretKey(secretKey);

  transaction.sign(keypair);
  return transaction;
}
```

#### 2. Email/Magic Link Login (for Web Dashboards)

```typescript
// Use a custodial or threshold-signing service
import { ThresholdSigner } from '@crypto-studio/solana-auth';

const signer = new ThresholdSigner({
  // 2-of-3 threshold: user email auth + protocol server + backup
  threshold: 2,
  parties: [
    { id: 'user', authMethod: 'email', email: userEmail },
    { id: 'protocol-server', envVar: 'PROTOCOL_SIGNER_KEY' },
    { id: 'backup', envVar: 'BACKUP_SIGNER_KEY' },
  ],
});

async function signTransactionForOperator(
  transaction: Transaction,
  userEmail: string,
): Promise<Transaction> {
  // User authenticates via magic link
  const userAuth = await authenticateEmail(userEmail);

  // Protocol server signs (automated for reward claims)
  const partialSig = await signer.partialSign({
    party: 'protocol-server',
    transaction,
  });

  // Combine signatures
  const combined = await signer.combine([
    partialSig,
    userAuth.signature,
  ]);

  transaction.addSignature(combined);
  return transaction;
}
```

#### 3. Session Keys (for IoT Devices)

See [device-identity.md](device-identity.md) — Session Keys for IoT Devices for a detailed implementation.

### Onboarding Flow

```
┌─────────────────────────────────────────────────────────────┐
│              Operator Onboarding Flow                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Download App / Visit Dashboard                           │
│     └─ Auto-generate wallet (no seed phrase!)               │
│                                                              │
│  2. Register Device                                          │
│     └─ Scan QR code / Connect hardware / Enter device ID    │
│     └─ Submit location (GPS / address)                      │
│     └─ Stake required tokens (shown in fiat equivalent)     │
│                                                              │
│  3. Verify Device                                            │
│     └─ Firmware check, hardware attestation                  │
│     └─ First proof-of-location challenge                     │
│     └─ Start earning trust score                             │
│                                                              │
│  4. Active Operation                                         │
│     └─ Dashboard shows: rewards earned, uptime, challenges  │
│     └─ Auto-claim rewards (or one-click claim)              │
│     └─ Push notifications for important events              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Dashboard Metrics

A DePIN operator dashboard should show:

```typescript
interface OperatorDashboard {
  // Device status
  deviceId: string;
  status: 'active' | 'inactive' | 'challenged';
  uptimePercent: number;       // Last 30 days
  lastHeartbeat: number;       // Unix timestamp

  // Rewards
  totalEarned: number;         // Lifetime
  pendingRewards: number;      // Unclaimed
  dailyRate: number;           // Current daily earnings
  nextClaimDate: number;       // For streamed rewards

  // Trust & reputation
  trustScore: number;          // 0-100
  challengesPassed: number;
  challengesFailed: number;

  // Staking
  stakedAmount: number;
  unlockDate: number | null;
  slashWarnings: number;
}
```

## Recommended Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              DePIN Monitoring Architecture                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  On-Chain Events                                             │
│  ┌──────────────┐                                            │
│  │ Solana       │                                            │
│  │ (DePIN       │                                            │
│  │  Program)    │                                            │
│  └──────┬───────┘                                            │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐    ┌──────────────┐                       │
│  │ Helius       │───►│ Webhook      │───► Backend Logic    │
│  │ Webhooks     │    │ Handler      │    (reward calc,      │
│  └──────────────┘    └──────────────┘     alerts, storage)  │
│                                              │              │
│  ┌──────────────┐                           ▼              │
│  │ Helius DAS   │◄──────────────────────────────────┐      │
│  │ API          │    Query device state on demand    │      │
│  └──────────────┘                                   │      │
│         ▲                                            │      │
│         │                                            │      │
│  ┌──────────────┐    ┌──────────────┐              │      │
│  │ Operator     │───►│ Dashboard    │◄─────────────┘      │
│  │ (Device)     │    │ (Web/Mobile) │                     │
│  └──────────────┘    └──────────────┘                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Related

- [device-identity.md](device-identity.md) — Device registration and session keys
- [zk-compression.md](zk-compression.md) — Querying compressed device state
- [reward-distribution.md](reward-distribution.md) — Triggering rewards via webhooks
