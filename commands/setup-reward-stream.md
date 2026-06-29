---
description: "Set up Streamflow reward streaming for DePIN operators"
---

You are configuring Streamflow reward streaming for a Solana DePIN. Follow these steps:

## Step 1: Gather Requirements

Ask the user about:
- **Operator count**: How many operators need reward streams?
- **Reward period**: Daily, weekly, monthly?
- **Token mint**: Which SPL token is used for rewards?
- **Tier structure**: All operators same rate, or tiered by performance?
- **Cliff period**: Any waiting period before first payout?
- **Staking**: Do operators need to stake to earn rewards?

## Step 2: Install Dependencies

```bash
npm install @streamflow/stream @solana/web3.js @solana/spl-token
```

## Step 3: Generate Stream Configuration

```typescript
import Streamflow from '@streamflow/stream';
import { Connection, Keypair, PublicKey } from '@solana/web3.js';

const streamflow = new Streamflow({
  connection: new Connection('https://api.mainnet-beta.solana.com'),
  commitment: 'confirmed',
});

// Configure based on requirements
const config = {
  sender: treasuryKeypair,
  mint: new PublicKey('TOKEN_MINT_ADDRESS'),
  period: PERIOD_IN_SECONDS,     // e.g., 30 * 24 * 60 * 60 for monthly
  cliff: CLIFF_IN_SECONDS,       // e.g., 14 * 24 * 60 * 60 for 2-week cliff
  canTopUp: true,
  cancellable: true,
  automaticWithdrawal: true,
};
```

## Step 4: Create Streams

### Single Stream

```typescript
const stream = await streamflow.createStream({
  ...config,
  recipient: operatorWallet,
  amount: monthlyReward,  // Raw token units
});
```

### Batch Streams (Multiple Operators)

```typescript
async function createBatchStreams(operators: OperatorReward[]) {
  for (const op of operators) {
    const stream = await streamflow.createStream({
      ...config,
      recipient: op.wallet,
      amount: op.monthlyAmount,
      name: `Operator ${op.deviceId} Rewards`,
    });
    console.log(`Stream created for ${op.deviceId}: ${stream.id}`);
  }
}
```

### Tiered Streams

```typescript
const tierAmounts = {
  bronze: 100_000,
  silver: 250_000,
  gold: 500_000,
  platinum: 1_000_000,
};
```

## Step 5: Verify

```typescript
// List all streams
const streams = await streamflow.getStreams({
  sender: treasuryKeypair.publicKey,
});

console.log(`Active streams: ${streams.length}`);
console.log(`Total streamed: ${streams.reduce((a, s) => a + s.depositedAmount, 0)}`);
```

## Step 6: Deliverables

- Streamflow integration script
- Operator stream list (CSV)
- Monitoring setup for stream health
