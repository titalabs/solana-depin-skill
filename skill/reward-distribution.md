# Reward Distribution for DePIN

Once you've designed your token model and reward formula (see [token-rewards.md](token-rewards.md)), you need to actually distribute rewards to operators. For DePINs with thousands of devices, naive per-operator transfer patterns don't scale.

## Distribution Mechanisms Compared

| Mechanism | Tx Cost | Scalability | When to Use |
|-----------|---------|-------------|-------------|
| Direct transfer | ~0.000005 SOL/tx | Poor (~100K SOL/yr for 50K daily ops) | <100 operators |
| Streamflow streaming | ~0.002 SOL/stream creation | Good | Recurring time-based rewards |
| Merkle distributor | ~0.00001 SOL/claim + verification | Great | Batch payouts, periodic claims |
| ZK compressed airdrop | ~0.000001 SOL/claim | Excellent | Large-scale drops, micro-rewards |

## 1. Streamflow Reward Streaming (Recurring Payouts)

Best for: **Continuous operator rewards** — pay operators per hour/day of uptime.

### Installation

```bash
npm install @streamflow/stream @solana/web3.js
```

### Create a Continuous Reward Stream

```typescript
import Streamflow, { StreamType } from '@streamflow/stream';
import { Connection, Keypair, PublicKey } from '@solana/web3.js';

const streamflow = new Streamflow({
  connection: new Connection('https://api.mainnet-beta.solana.com'),
  commitment: 'confirmed',
});

// Monthly reward for an operator
async function createOperatorStream(
  sender: Keypair,
  recipient: PublicKey,
  rewardMint: PublicKey,
  monthlyAmount: number,
  cliffDays: number = 14,    // 2-week cliff before first payout
) {
  const stream = await streamflow.createStream({
    sender: sender.publicKey,
    recipient,
    mint: rewardMint,
    amount: monthlyAmount,
    period: 30 * 24 * 60 * 60,  // 30 days
    cliff: cliffDays * 24 * 60 * 60,
    canTopUp: true,
    cancellable: true,
    automaticWithdrawal: true,
    name: `Operator Rewards - ${recipient.toBase58()}`,
  });

  console.log(`Stream created: ${stream.id}`);
  return stream;
}
```

### Staking Pool (Fixed Budget)

```typescript
// Create a staking pool with a fixed reward budget
const stakingPool = await streamflow.createStaking({
  poolType: 'fund-once',  // Fixed budget, depletes over time
  mint: rewardMint,
  amount: totalRewardBudget,  // e.g., 1M tokens
  period: 30 * 24 * 60 * 60,
  stakeDuration: 7 * 24 * 60 * 60,  // Min 7-day stake
});

// Operators stake tokens to earn rewards
await streamflow.stake({
  poolId: stakingPool.id,
  amount: operatorStake,
});
```

### Automated Reward Tiers

```typescript
// Create tiered streams based on device quality score
async function distributeTieredRewards(
  operators: Array<{
    wallet: PublicKey;
    tier: 'bronze' | 'silver' | 'gold' | 'platinum';
  }>,
  rewardMint: PublicKey,
) {
  const tierAmounts = {
    bronze: 100_000,    // 100K tokens/month
    silver: 250_000,    // 250K tokens/month
    gold: 500_000,      // 500K tokens/month
    platinum: 1_000_000, // 1M tokens/month
  };

  for (const op of operators) {
    const amount = tierAmounts[op.tier];
    await streamflow.createStream({
      sender: treasury.publicKey,
      recipient: op.wallet,
      mint: rewardMint,
      amount,
      period: 30 * 24 * 60 * 60,
      cliff: 7 * 24 * 60 * 60,
      canTopUp: true,
      cancellable: true,
      automaticWithdrawal: true,
    });
  }
}
```

## 2. Merkle Distributor (Batch Payouts)

Best for: **Periodic batch payouts** — distribute accumulated rewards every epoch.

The Merkle distributor pattern allows you to create a single on-chain commitment to a reward list. Operators claim their rewards with a Merkle proof, paying only the claim transaction fee.

### Anchor Program: Merkle Distributor

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};
use std::collections::hash_map::DefaultHasher;

declare_id!("Dist4ributor1111111111111111111111111111111");

#[account]
pub struct Distributor {
    pub authority: Pubkey,
    pub mint: Pubkey,
    pub root: [u8; 32],        // Merkle root of reward tree
    pub total_rewards: u64,
    pub claimed: u64,
    pub epoch: u64,
    pub bump: u8,
}

#[account]
pub struct ClaimStatus {
    pub claimant: Pubkey,
    pub epoch: u64,
    pub amount: u64,
    pub claimed: bool,
}

#[derive(AnchorSerialize, AnchorDeserialize)]
pub struct RewardLeaf {
    pub claimant: Pubkey,
    pub amount: u64,
}

pub fn new_distributor(
    ctx: Context<NewDistributor>,
    root: [u8; 32],
    epoch: u64,
    total_rewards: u64,
) -> Result<()> {
    let distributor = &mut ctx.accounts.distributor;
    distributor.authority = ctx.accounts.authority.key();
    distributor.mint = ctx.accounts.mint.key();
    distributor.root = root;
    distributor.epoch = epoch;
    distributor.total_rewards = total_rewards;
    distributor.claimed = 0;
    distributor.bump = ctx.bumps.distributor;
    Ok(())
}

pub fn claim_reward(
    ctx: Context<ClaimReward>,
    amount: u64,
    proof: Vec<[u8; 32]>,
) -> Result<()> {
    let distributor = &ctx.accounts.distributor;
    let claim_status = &mut ctx.accounts.claim_status;

    require!(!claim_status.claimed, ErrorCode::AlreadyClaimed);
    require!(distributor.epoch == claim_status.epoch, ErrorCode::WrongEpoch);

    // Verify Merkle proof
    let leaf = hash_leaf(&RewardLeaf {
        claimant: ctx.accounts.claimant.key(),
        amount,
    });
    require!(verify_proof(proof, distributor.root, leaf), ErrorCode::InvalidProof);

    // Transfer tokens
    let seeds = &[b"distributor".as_ref(), &[distributor.bump]];
    token::transfer(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.claimant_token.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
            &[seeds],
        ),
        amount,
    )?;

    claim_status.claimed = true;
    distributor.claimed = distributor.claimed.checked_add(amount).unwrap();
    Ok(())
}

fn hash_leaf(leaf: &RewardLeaf) -> [u8; 32] {
    let serialized = [
        leaf.claimant.as_ref(),
        &leaf.amount.to_le_bytes(),
    ].concat();
    anchor_lang::solana_program::keccak::hash(&serialized).to_bytes()
}

fn verify_proof(proof: Vec<[u8; 32]>, root: [u8; 32], leaf: [u8; 32]) -> bool {
    let mut computed = leaf;
    for sibling in proof {
        let mut combined = [0u8; 64];
        if computed <= sibling {
            combined[..32].copy_from_slice(&computed);
            combined[32..].copy_from_slice(&sibling);
        } else {
            combined[..32].copy_from_slice(&sibling);
            combined[32..].copy_from_slice(&computed);
        }
        computed = anchor_lang::solana_program::keccak::hash(&combined).to_bytes();
    }
    computed == root
}
```

### TypeScript: Generating the Merkle Tree

```typescript
import { keccak_256 } from 'js-sha3';
import { PublicKey } from '@solana/web3.js';

interface RewardLeaf {
  claimant: string;  // base58 wallet address
  amount: number;    // token amount (raw units)
}

class MerkleDistributor {
  private leaves: Buffer[] = [];
  private tree: Buffer[][] = [];
  public root: Buffer;

  constructor(rewards: RewardLeaf[]) {
    this.leaves = rewards.map(r => this.hashLeaf(r));
    this.tree = this.buildTree(this.leaves);
    this.root = this.tree[this.tree.length - 1][0];
  }

  private hashLeaf(leaf: RewardLeaf): Buffer {
    const claimant = new PublicKey(leaf.claimant).toBytes();
    const amount = Buffer.alloc(8);
    amount.writeBigUInt64LE(BigInt(leaf.amount));
    const data = Buffer.concat([claimant, amount]);
    return Buffer.from(keccak_256.arrayBuffer(data));
  }

  private buildTree(leaves: Buffer[]): Buffer[][] {
    if (leaves.length === 0) return [];
    const tree: Buffer[][] = [leaves];
    while (tree[tree.length - 1].length > 1) {
      const level: Buffer[] = [];
      const last = tree[tree.length - 1];
      for (let i = 0; i < last.length; i += 2) {
        if (i + 1 < last.length) {
          const combined = Buffer.concat(
            last[i].compare(last[i + 1]) <= 0
              ? [last[i], last[i + 1]]
              : [last[i + 1], last[i]]
          );
          level.push(Buffer.from(keccak_256.arrayBuffer(combined)));
        } else {
          level.push(last[i]);
        }
      }
      tree.push(level);
    }
    return tree;
  }

  getProof(leafIndex: number): Buffer[] {
    const proof: Buffer[] = [];
    for (let i = 0; i < this.tree.length - 1; i++) {
      const level = this.tree[i];
      const isRight = leafIndex % 2 === 0
        ? leafIndex + 1 < level.length
        : true;
      const siblingIndex = isRight
        ? leafIndex + (leafIndex % 2 === 0 ? 1 : -1)
        : leafIndex - 1;
      if (siblingIndex < level.length) {
        proof.push(level[siblingIndex]);
      }
      leafIndex = Math.floor(leafIndex / 2);
    }
    return proof;
  }
}
```

## 3. ZK Compressed Airdrops (Mass Scale)

Best for: **One-time drops to millions of operators** — token distribution at genesis, retroactive rewards.

ZK Compression (via Helius/Light Protocol) reduces the cost of token distribution by 10-100×.

```typescript
import { LightSystemProgram } from '@lightprotocol/stateless.js';
import { createRpc } from '@lightprotocol/stateless.js';

// Initialize compressed token mint
const rpc = createRpc('https://mainnet.helius-rpc.com/?api-key=YOUR_KEY');

// Compressed airdrop to 100K operators
async function compressedAirdrop(
  recipients: Array<{ owner: PublicKey; amount: number }>,
  mint: PublicKey,
  authority: Keypair,
) {
  const compressedMint = await LightSystemProgram.createCompressedMint({
    rpc,
    authority: authority.publicKey,
    mint,
    payer: authority,
  });

  // Batch mint compressed tokens to recipients
  const batchSize = 1000;
  for (let i = 0; i < recipients.length; i += batchSize) {
    const batch = recipients.slice(i, i + batchSize);
    const compressedTx = await LightSystemProgram.mintCompressedTokens({
      rpc,
      mint: compressedMint,
      recipients: batch.map(r => ({
        owner: r.owner,
        amount: r.amount,
      })),
      authority,
      payer: authority,
    });

    console.log(`Batch ${i / batchSize + 1}: ${compressedTx}`);
  }
}
```

See [zk-compression.md](zk-compression.md) for details on compressed accounts.

## 4. Hybrid Approach (Recommended for Most DePINs)

For production DePINs, combine multiple mechanisms:

```
┌─────────────────────────────────────────────────────────────┐
│              Hybrid Reward Distribution                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Weekly:                                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. Calculate rewards off-chain                      │    │
│  │    └─ Per-operator amount based on contribution      │    │
│  └─────────────────────────────────────────────────────┘    │
│                            │                                 │
│                            ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 2. Generate Merkle tree                             │    │
│  │    └─ Commit root on-chain                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                            │                                 │
│                            ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 3. Operators claim via Merkle proof                 │    │
│  │    └─ Pay only claim tx fee (~0.00001 SOL)          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Monthly:                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Streamflow streams for consistent operators          │    │
│  │ └─ Auto-deposit recurring rewards                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  One-time:                                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ ZK compressed drops for retroactive / genesis        │    │
│  │ └─ 100K+ recipients, pennies per drop               │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## Cost Estimation

```typescript
function estimateAnnualDistributionCost(
  operatorCount: number,
  distributionFrequency: 'daily' | 'weekly' | 'monthly',
  mechanism: 'stream' | 'merkle' | 'compressed',
): { txCost: number; annualCost: number } {
  const lamportsPerSignature = 5000;
  const txCostPerClaim = mechanism === 'stream' ? 0.002
    : mechanism === 'merkle' ? 0.00001
    : 0.000001; // SOL

  const frequency = distributionFrequency === 'daily' ? 365
    : distributionFrequency === 'weekly' ? 52
    : 12;

  const annualTxCost = txCostPerClaim * operatorCount * frequency;

  return {
    txCost: txCostPerClaim,
    annualCost: annualTxCost,
  };
}

// Example: 50K operators, weekly Merkle distribution
const cost = estimateAnnualDistributionCost(50000, 'weekly', 'merkle');
console.log(`Annual cost: ${cost.annualCost} SOL`);
// Annual cost: 26 SOL/year for 50K operators
```

## Related

- [token-rewards.md](token-rewards.md) — Token mechanics and reward formula design
- [zk-compression.md](zk-compression.md) — Compressed token distribution at scale
- [indexing-operator-onboarding.md](indexing-operator-onboarding.md) — Monitoring reward claims
