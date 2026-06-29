# Oracle Integration for DePIN

DePINs need real-world data on-chain: token prices for reward calculations, location data for proof of coverage, weather for sensor networks, and randomness for challenge protocols. This skill covers the two major oracle providers on Solana.

## Oracle Comparison

| Feature | Pyth | Switchboard |
|---------|------|-------------|
| **Data Type** | Price feeds (400+ assets) | Custom data (any API, JSON, SQL, WebSocket) |
| **Update Speed** | 400ms (push/pull) | sub-100ms (Surge streaming) |
| **Randomness** | Entropy (EVM only, Solana pending) | VRF (TEE-based, Solana available) |
| **Permissionless** | Publisher-whitelisted | Fully permissionless |
| **Use Case** | Token pricing, reward calculations | Geo-data, weather, custom feeds, VRF |
| **DePIN Fit** | Reward value calculations | Proof of location, challenge protocols |

## 1. Pyth Price Feeds

Best for: **Reward value calculations, staking APY, token pricing**.

### Installation

```bash
npm install @pythnetwork/pyth-solana-receiver @pythnetwork/price-feed-client
```

### Pull Model (Recommended for DePIN)

The Pull model lets you fetch price data on-demand, paying only when you use it. This is ideal for DePINs where reward calculations happen on a schedule (daily/weekly), not continuously.

```typescript
import { PythSolanaReceiver } from '@pythnetwork/pyth-solana-receiver';
import { Connection, PublicKey } from '@solana/web3.js';

const connection = new Connection('https://api.mainnet-beta.solana.com');
const receiver = new PythSolanaReceiver({ connection });

// Get latest price for reward calculation
async function getTokenPrice(tokenSymbol: string): Promise<{
  price: number;
  confidence: number;
  timestamp: number;
}> {
  const priceFeed = await receiver.getPriceFeed(tokenSymbol);

  return {
    price: Number(priceFeed.getPriceUnchecked()),
    confidence: Number(priceFeed.getConfidenceRatio()),
    timestamp: priceFeed.getTimestamp(),
  };
}

// Example: Calculate operator reward in USD
async function calculateOperatorRewardUSD(
  tokenAmount: number,
  tokenSymbol: string,
): Promise<number> {
  const { price, confidence, timestamp } = await getTokenPrice(tokenSymbol);

  // Check freshness (reject data older than 60 seconds)
  const now = Math.floor(Date.now() / 1000);
  if (now - timestamp > 60) {
    throw new Error('Stale price data');
  }

  // Check confidence (reject if > 1% spread)
  if (confidence > 0.01) {
    throw new Error('Low confidence price');
  }

  return tokenAmount * price;
}
```

### On-Chain Pull via Hermes

```rust
use pyth_solana_receiver_sdk::price_update::PriceUpdateV2;
use anchor_lang::prelude::*;

pub fn calculate_reward_with_price(
    ctx: Context<CalculateReward>,
    feed_id: [u8; 32],
    amount: u64,
) -> Result<()> {
    let price_update = &ctx.accounts.price_update;

    // Verify and get price
    let price = price_update.get_price_no_older_than(&Clock::get()?, 60, &feed_id)?;

    // price.price is the current price
    // price.confidence is the confidence interval
    // price.exponent is the decimal exponent (e.g., -8 for 8 decimals)

    let price_value: i64 = price.price;
    let exponent: i32 = price.exponent;
    let confidence: u64 = price.confidence;

    // Check confidence ratio
    require!(
        (confidence as f64 / price_value.unsigned_abs() as f64) < 0.01,
        ErrorCode::LowConfidence
    );

    // Calculate USD value of reward
    // Reward USD = amount * price / 10^|exponent|
    let scaling = 10i64.pow(exponent.unsigned_abs()) as u128;
    let reward_usd = (amount as u128)
        .checked_mul(price_value.unsigned_abs() as u128)
        .unwrap()
        .checked_div(scaling)
        .unwrap();

    Ok(())
}
```

## 2. Switchboard Custom Feeds

Best for: **Custom data (weather, GPS, traffic), geospatial oracles, VRF for challenge protocols**.

### Installation

```bash
npm install @switchboard-xyz/switchboard-v2 @switchboard-xyz/vrf
```

### Custom Data Feed

```typescript
import { SwitchboardProgram, AggregatorAccount } from '@switchboard-xyz/switchboard-v2';

// Create a custom feed for DePIN data
async function createGeoLocationFeed() {
  const program = await SwitchboardProgram.load(
    'mainnet-beta',
    new Connection('https://api.mainnet-beta.solana.com'),
    payerKeypair,
  );

  // Create aggregator that pulls from a location verification API
  const aggregator = await program.createAggregator({
    queue: queueAccount,
    // Define oracle jobs — pull from location verification service
    jobs: [
      {
        tasks: [
          {
            // Fetch JSON from a location API
            httpTask: {
              url: 'https://api.location-service.com/verify?lat={latitude}&lng={longitude}',
              method: 'GET',
              headers: [{ key: 'Authorization', value: 'Bearer YOUR_API_KEY' }],
            },
          },
          {
            // Parse JSON response
            jsonParseTask: { path: '$.verified' },
          },
        ],
      },
    ],
    // Minimum time between updates (seconds)
    minUpdateDelaySeconds: 300,
    // Minimum number of oracle responses
    minOracleResults: 3,
  });

  return aggregator;
}
```

### On-Chain: Read Switchboard Feed

```rust
use switchboard_v2::AggregatorAccountData;

pub fn verify_device_location(
    ctx: Context<VerifyLocation>,
) -> Result<()> {
    let aggregator = &ctx.accounts.location_feed;
    let feed = aggregator.get_data()?;

    // Get latest confirmed value
    let latest_value = feed.get_latest_value()?;
    let result: f64 = latest_value.parse()?;

    // result = 1.0 if location verified, 0.0 if not
    require!(result > 0.5, ErrorCode::LocationNotVerified);

    Ok(())
}
```

### Switchboard VRF for Random Challenges

The most critical use of VRF in DePIN: **selecting challenge targets unpredictably** to prevent gaming.

```typescript
import { SwitchboardVRF } from '@switchboard-xyz/vrf';

async function requestRandomChallenge(deviceRegistry: PublicKey[]) {
  const vrf = new SwitchboardVRF({
    connection: new Connection('https://api.mainnet-beta.solana.com'),
    payer: protocolKeypair,
  });

  // Request a random number
  const request = await vrf.requestRandomness({
    authority: protocolKeypair.publicKey,
    // Optionally pass a "callback" program to receive the randomness
    callback: {
      programId: depinProgramId,
      accounts: [
        { pubkey: stateAccount, isSigner: false, isWritable: true },
      ],
      ixData: Buffer.from([]),
    },
  });

  // Wait for VRF fulfillment (typically 2-3 slots)
  const randomness = await request.waitForFulfillment();
  const randomValue = randomness.toBigInt();

  // Select challenge target
  const targetIndex = Number(randomValue % BigInt(deviceRegistry.length));
  const challengeTarget = deviceRegistry[targetIndex];

  return { challengeTarget, randomValue, proof: randomness.proof };
}
```

### On-Chain: VRF Callback

```rust
use switchboard_vrf::program::SwitchboardVRF;

#[derive(Accounts)]
pub struct VrfChallengeCallback<'info> {
    pub vrf: Account<'info, VrfAccount>,
    pub state: AccountMut<'info, ChallengeState>,
}

pub fn handle_vrf_callback(ctx: Context<VrfChallengeCallback>) -> Result<()> {
    let vrf = &ctx.accounts.vrf;
    let state = &mut ctx.accounts.state;

    // Get randomness from VRF
    let randomness = vrf.randomness()?;
    let random_value = randomness[..8].try_into().unwrap_or([0u8; 8]);
    let selection = u64::from_le_bytes(random_value);

    // Select challenge target
    let target_index = selection % state.device_count as u64;
    state.current_challenge_target = state.device_registry[target_index as usize];

    Ok(())
}
```

### Switchboard Surge (Sub-100ms Streaming)

For DePINs needing real-time data (bandwidth monitoring, compute metrics):

```typescript
import { SurgeClient } from '@switchboard-xyz/surge';

const surge = new SurgeClient({
  apiKey: 'YOUR_SURGE_API_KEY',
});

// Subscribe to streaming feed
const subscription = surge.subscribe('solana-depin-reward-oracle', {
  onData: (data) => {
    console.log('Real-time data:', data);
    // Trigger on-chain update if needed
  },
  onError: (err) => console.error('Surge error:', err),
});
```

## 3. Custom Oracle Design for DePIN

Sometimes you need data that neither Pyth nor Switchboard natively provides. Here's how to design a custom oracle for DePIN.

### Oracle Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Data Source │ ──► │  Oracle Node  │ ──► │  On-Chain    │
│  (API, IoT,  │     │  (aggregator, │     │  (verified   │
│   database)  │     │   signer)     │     │   state)     │
└──────────────┘     └──────────────┘     └──────────────┘
                          │
                          │ Optional:
                          ▼
                    ┌──────────────┐
                    │  Validator   │
                    │  (reputation,│
                    │   staking)   │
                    └──────────────┘
```

### Minimal On-Chain Oracle

```rust
#[account]
pub struct OracleFeed {
    pub authority: Pubkey,
    pub value: i64,
    pub timestamp: i64,
    pub confidence: u64,
    pub feed_name: String,
}

// Authorized oracle updates the feed
pub fn update_oracle_feed(
    ctx: Context<UpdateFeed>,
    value: i64,
    confidence: u64,
) -> Result<()> {
    let feed = &mut ctx.accounts.feed;

    // Only authorized oracle can update
    require!(feed.authority == ctx.accounts.oracle.key(), ErrorCode::Unauthorized);

    feed.value = value;
    feed.timestamp = Clock::get()?.unix_timestamp;
    feed.confidence = confidence;

    Ok(())
}

// DePIN program reads feed
pub fn use_oracle_data(ctx: Context<UseOracleData>) -> Result<()> {
    let feed = &ctx.accounts.feed;

    // Check freshness
    let now = Clock::get()?.unix_timestamp;
    require!(now - feed.timestamp < 3600, ErrorCode::StaleData); // 1 hour old

    // Check confidence
    require!(feed.confidence < 1000, ErrorCode::LowConfidence);

    // Use the value in reward calculation
    let reward = calculate_reward_with_external_data(feed.value);
    distribute_reward(&ctx, reward)?;

    Ok(())
}
```

## 4. Oracle Reliability Patterns

### Multiple Oracle Sources (Defense in Depth)

```typescript
async function getLocationWithFallback(deviceId: string): Promise<GeoLocation> {
  const sources = [
    // Primary: Switchboard custom feed
    switchboardFeed.getLatestValue(),
    // Secondary: Pyth (if available — geo oracles on Pyth are limited)
    // Tertiary: On-chain consensus from nearby devices
  ];

  // Wait for first successful response
  const result = await Promise.any(sources);
  return result;
}
```

### Staleness Checks

Always verify oracle data freshness before using it in reward calculations:

```rust
// Reject oracle data older than N seconds
pub fn validate_oracle_freshness(
    feed_timestamp: i64,
    max_age_seconds: i64,
) -> Result<()> {
    let now = Clock::get()?.unix_timestamp;
    let age = now.checked_sub(feed_timestamp)
        .ok_or(ErrorCode::ClockError)?;

    require!(age < max_age_seconds, ErrorCode::StaleOracleData);
    Ok(())
}
```

### SLA for Oracle Updates

For DePIN reward calculations that run on a schedule:

```typescript
// Before running weekly reward distribution, verify oracle health
async function verifyOracleHealth(): Promise<boolean> {
  const feeds = [
    { name: 'pyth-price', freshness: 60 },        // max 60s old
    { name: 'switchboard-location', freshness: 300 }, // max 5min old
  ];

  for (const feed of feeds) {
    const data = await getOracleData(feed.name);
    if (Date.now() / 1000 - data.timestamp > feed.freshness) {
      console.error(`Oracle feed ${feed.name} is stale`);
      return false;
    }
  }
  return true;
}
```

## Related

- [proof-mechanisms.md](proof-mechanisms.md) — VRF for challenge selection, oracle-verified proofs
- [token-rewards.md](token-rewards.md) — Oracle-based reward calculations
- [device-identity.md](device-identity.md) — Oracle-verified device data
