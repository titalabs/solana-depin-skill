# Staking, Slashing & Governance for DePIN

## Operator Staking

Staking aligns operator incentives with network health. Operators lock tokens (protocol tokens or SOL) that can be slashed for misbehavior.

### Why Stake?

1. **Sybil resistance**: Makes running fake devices economically irrational
2. **Skin in the game**: Operators have something to lose
3. **Service quality**: Stake can be slashed for poor performance
4. **Governance weight**: Staked operators earn voting rights

### Staking Pool Design

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

#[account]
pub struct StakingPool {
    pub authority: Pubkey,
    pub stake_mint: Pubkey,
    pub reward_mint: Pubkey,
    pub total_staked: u64,
    pub min_stake: u64,
    pub unstake_period: i64,    // Lock period before unstake
    pub reward_rate: u64,       // Tokens per epoch per staked token
    pub last_distribution: i64,
    pub epoch_duration: i64,
    pub bump: u8,
}

#[account]
pub struct OperatorStake {
    pub operator: Pubkey,
    pub device: Pubkey,          // Associated device
    pub amount: u64,
    pub staked_at: i64,
    pub lock_until: i64,
    pub rewards_earned: u64,
    pub slash_count: u8,
}

pub fn stake_tokens(
    ctx: Context<StakeTokens>,
    amount: u64,
) -> Result<()> {
    let pool = &mut ctx.accounts.pool;
    let stake = &mut ctx.accounts.operator_stake;
    let now = Clock::get()?.unix_timestamp;

    // Transfer tokens from operator to pool
    token::transfer(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.operator_token.to_account_info(),
                to: ctx.accounts.pool_vault.to_account_info(),
                authority: ctx.accounts.operator.to_account_info(),
            },
        ),
        amount,
    )?;

    // Update stake record
    if stake.amount == 0 {
        // First time staking
        stake.operator = ctx.accounts.operator.key();
        stake.device = ctx.accounts.device.key();
        stake.staked_at = now;
    }

    stake.amount = stake.amount.checked_add(amount).unwrap();
    stake.lock_until = now.checked_add(pool.unstake_period).unwrap();

    pool.total_staked = pool.total_staked.checked_add(amount).unwrap();

    Ok(())
}

pub fn unstake_tokens(ctx: Context<UnstakeTokens>) -> Result<()> {
    let pool = &mut ctx.accounts.pool;
    let stake = &mut ctx.accounts.operator_stake;
    let now = Clock::get()?.unix_timestamp;

    require!(now >= stake.lock_until, ErrorCode::TokensLocked);

    let amount = stake.amount;

    // Transfer tokens back
    let seeds = &[b"pool_vault".as_ref(), &[pool.bump]];
    token::transfer(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.pool_vault.to_account_info(),
                to: ctx.accounts.operator_token.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
            &[seeds],
        ),
        amount,
    )?;

    stake.amount = 0;
    pool.total_staked = pool.total_staked.checked_sub(amount).unwrap();

    Ok(())
}
```

### Streamflow Staking Pools (No Anchor Code Required)

For simpler setups, use Streamflow's staking pools:

```typescript
import Streamflow from '@streamflow/stream';

// Create a continuous staking pool
const pool = await streamflow.createStaking({
  poolType: 'continuous-funding',
  mint: rewardMint,
  amount: rewardBudget,       // Total reward budget
  rewardPeriod: 7 * 24 * 60 * 60, // Weekly rewards
  stakeDuration: 14 * 24 * 60 * 60, // Min 14-day lock
  minStake: minStakeAmount,
  maxStake: maxStakeAmount,
});

// Operator stakes
await streamflow.stake({
  poolId: pool.id,
  amount: operatorStake,
});

// Claim rewards
const rewards = await streamflow.claimRewards({
  poolId: pool.id,
  staker: operatorWallet,
});
```

## Slashing Conditions

Clear, deterministic slashing conditions are essential for fairness.

### Slashing Events

| Event | Severity | Slash Amount | Notes |
|-------|----------|-------------|-------|
| Missed challenge | Minor | 5% of stake | First offense: warning |
| Failed PoC | Minor | 10% of stake | Repeated failures escalate |
| Location spoofing | Major | 50% of stake | Requires oracle verification |
| Data fraud | Major | 100% of stake (slash + ban) | Proven by consensus |
| Collusion | Critical | 100% of stake (slash + ban) | Multiple independent proofs |
| Extended offline | Minor | 2% per day | After grace period |

### Anchor Program: Slash

```rust
pub fn slash_operator(
    ctx: Context<SlashOperator>,
    slash_percentage: u8,  // 1-100
    reason: SlashReason,
) -> Result<()> {
    let pool = &mut ctx.accounts.pool;
    let stake = &mut ctx.accounts.operator_stake;
    let device = &mut ctx.accounts.device;

    // Verify slasher authority (DAO or automated challenge)
    require!(
        ctx.accounts.slasher.key() == pool.authority
            || ctx.accounts.slasher.key() == ctx.accounts.challenge_authority.key(),
        ErrorCode::Unauthorized
    );

    let slash_amount = stake.amount * (slash_percentage as u64) / 100;
    require!(slash_amount > 0, ErrorCode::SlashWouldBeZero);

    // Burn slashed tokens or send to treasury
    let seeds = &[b"pool_vault".as_ref(), &[pool.bump]];
    token::transfer(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.pool_vault.to_account_info(),
                to: ctx.accounts.treasury.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
            &[seeds],
        ),
        slash_amount,
    )?;

    stake.amount = stake.amount.checked_sub(slash_amount).unwrap();
    pool.total_staked = pool.total_staked.checked_sub(slash_amount).unwrap();
    stake.slash_count = stake.slash_count.checked_add(1).unwrap();

    // Update device trust score
    device.trust_score = device.trust_score
        .checked_sub(20)
        .unwrap_or(0);

    // Ban if critical or repeated
    if matches!(reason, SlashReason::DataFraud | SlashReason::Collusion)
        || stake.slash_count >= 3
    {
        device.is_active = false;
    }

    Ok(())
}
```

### Governance-Based Slashing

For subjective slashing decisions (e.g., disputed data quality), use DAO voting:

```typescript
// Squads multisig for admin slashing
const squads = new SquadsClient(connection, multisigWallet);

// Create a slash proposal
const proposal = await squads.createProposal({
  multisigPda: multisigPda,
  transactionIndex: 0,
  proposal: {
    title: `Slash operator ${operatorWallet} for location spoofing`,
    description: `Evidence: Device claimed location X but oracle data shows location Y. ${evidenceLink}`,
  },
});

// 3/5 signers must approve
await squads.approve({ multisigPda, proposalIndex: proposal.index });
await squads.execute({ multisigPda, proposalIndex: proposal.index });
```

## DAO Governance (Realms / SPL Governance)

For community-governed DePINs, parameter changes go through the DAO.

### DePIN Parameters Suitable for DAO Control

| Parameter | Description | Why DAO Control? |
|-----------|-------------|-----------------|
| `reward_rate` | Tokens per epoch per operator | Community adjusts based on operator count |
| `challenge_interval` | Seconds between challenges | Balances security vs cost |
| `min_stake` | Minimum stake to register | Adjusts with token price |
| `slash_percentage` | Slash severity per offense | Community sets fairness |
| `operator_allocation` | % of emissions to operators | Budget allocation |

### Setting Up Realms for a DePIN

```typescript
import { Governance, Realm } from '@solana/spl-governance';

// Create a realm for your DePIN
const realm = await Realm.create({
  connection,
  payer: deployer,
  realm: {
    name: 'My DePIN DAO',
    minCommunityTokensToCreateGovernance: 1_000_000,  // 1M tokens
    communityMint: depinTokenMint,
  },
});

// Create a governance for reward rate
const rewardGovernance = await Governance.create({
  connection,
  payer: deployer,
  realm: realm.pubkey,
  governance: {
    governedAccount: depinProgramState,
    config: {
      minCommunityWeightToVote: 100_000,  // 100K tokens to vote
      minInstructionHoldupTime: 86400,    // 1-day delay
      voteThreshold: VoteThreshold.Percentage,  // 60% majority
      voteThresholdValue: 60,
      proposalCount: 0,
    },
  },
});
```

### On-Chain: Parameter Governance via Program

```rust
#[account]
pub struct GovernanceParams {
    pub reward_rate: u64,
    pub challenge_interval: i64,
    pub min_stake: u64,
    pub slash_percentage: u8,
    pub operator_allocation: u8,  // % of emissions
    pub last_update: i64,
    pub governance: Pubkey,  // Realms governance address
}

// Only the DAO can update parameters
pub fn update_parameter(
    ctx: Context<UpdateParam>,
    param: ParameterType,
    value: u64,
) -> Result<()> {
    require!(
        ctx.accounts.governance_authority.key() == ctx.accounts.params.governance,
        ErrorCode::Unauthorized
    );

    let params = &mut ctx.accounts.params;

    match param {
        ParameterType::RewardRate => params.reward_rate = value,
        ParameterType::ChallengeInterval => params.challenge_interval = value as i64,
        ParameterType::MinStake => params.min_stake = value,
        ParameterType::SlashPercentage => {
            require!(value <= 100, ErrorCode::InvalidValue);
            params.slash_percentage = value as u8;
        }
        ParameterType::OperatorAllocation => {
            require!(value <= 100, ErrorCode::InvalidValue);
            params.operator_allocation = value as u8;
        }
    }

    params.last_update = Clock::get()?.unix_timestamp;

    Ok(())
}
```

## Squads Multisig (Treasury & Upgrades)

For protocol treasury management and program upgrades, use Squads v5.

### Setting Up Squads for a DePIN

```typescript
import { SquadsClient } from '@squads/sdk';

const squads = new SquadsClient(connection, wallet);

// Create a 3/5 multisig for protocol treasury
const multisig = await squads.createMultisig({
  threshold: 3,       // 3 of 5 signers required
  signers: [
    founder1.publicKey,
    founder2.publicKey,
    treasuryManager.publicKey,
    leadDev.publicKey,
    community.publicKey,
  ],
});

// Set as program upgrade authority
await squads.setProgramAuthority({
  programId: depinProgramId,
  newAuthority: multisig.publicKey,
});
```

### Common Squads Flows for DePIN

```typescript
// Batch transfer: pay operator rewards
const tx = await squads.createTransaction({
  multisigPda: multisigPda,
  instructions: [
    createTransferInstruction(treasuryVault, operatorWallet, rewardAmount),
    createTransferInstruction(treasuryVault, devFund, devAllocation),
  ],
});

await squads.approve({ multisigPda, transactionIndex: tx.index });
await squads.execute({ multisigPda, transactionIndex: tx.index });
```

## Governance Maturity Model

```
Phase 1: Founding Team Controls
├── Multisig (Squads 2/3) for treasury + program upgrades
├── Team sets all parameters
├── No token voting yet

Phase 2: Token Launch
├── Token holders can stake
├── Staking rewards begin
├── Foundation holds reserve voting power

Phase 3: Community Governance
├── Realms DAO for parameter votes
├── Token-weighted voting on reward rates, challenge intervals
├── Squads retains upgrade authority (timelocked)

Phase 4: Full Decentralization
├── DAO controls program upgrades (timelock + multisig)
├── All parameters governed by token vote
├── Treasury managed by DAO
├── Squads upgrades only via DAO proposal
```

## Related

- [device-identity.md](device-identity.md) — Stake requirements for device registration
- [token-rewards.md](token-rewards.md) — Reward rate design and staking rewards
- [reward-distribution.md](reward-distribution.md) — Staking pool integration via Streamflow
- [proof-mechanisms.md](proof-mechanisms.md) — Challenge protocol that triggers slashing
