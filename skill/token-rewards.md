# Token Rewards for DePIN

## Token Design Decisions

Every DePIN needs a token that rewards operators and aligns incentives across the network. The three core decisions are:

1. **What is the token for?** — Reward? Utility? Governance? All three?
2. **What is the supply model?** — Fixed? Inflationary? Burn-and-mint?
3. **How are rewards calculated?** — Per-device? Per-contribution? Time-weighted?

## Token Supply Models

### Fixed Supply
Token supply is capped from genesis. All tokens are pre-mined or minted on a schedule.

**Pros**: Predictable, familiar to investors, no inflation pressure
**Cons**: No long-term reward budget, may run dry as network grows
**Used by**: Render Network (RNDR — fixed supply, distributed as work is completed)

```rust
// Anchor: Fixed supply token mint
use anchor_spl::token::{self, MintTo, Token};

pub fn initialize_token(ctx: Context<InitializeToken>, supply: u64) -> Result<()> {
    let mint = &ctx.accounts.mint;
    // Mint total supply to treasury
    token::mint_to(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                mint: mint.to_account_info(),
                to: ctx.accounts.treasury.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
        ),
        supply,
    )?;
    // Disable future minting
    token::set_authority(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            SetAuthority {
                account_or_mint: mint.to_account_info(),
                current_authority: ctx.accounts.authority.to_account_info(),
            },
        ),
        spl_token::instruction::AuthorityType::MintTokens,
        None, // Set to None to disable
    )?;
    Ok(())
}
```

### Inflationary with Emission Schedule
New tokens are minted according to a schedule (daily, monthly, yearly). Often includes a halving mechanism.

**Pros**: Continuous reward budget, aligns with network growth
**Cons**: Inflation dilutes holders, needs careful scheduling
**Used by**: Helium (HNT), Hivemapper (HONEY)

```rust
// Anchor: Time-based reward emission
pub fn claim_rewards(ctx: Context<ClaimRewards>) -> Result<()> {
    let state = &ctx.accounts.state;
    let now = Clock::get()?.unix_timestamp;
    let time_elapsed = now - state.last_distribution;

    // Calculate emission based on time and remaining supply
    let daily_emission = calculate_daily_emission(state.total_staked, state.emission_rate);
    let reward = (daily_emission as i128 * time_elapsed as i128 / 86400) as u64;

    // Mint and distribute
    let seeds = &[b"treasury".as_ref(), &[state.bump]];
    token::mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                mint: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.operator_token.to_account_info(),
                authority: ctx.accounts.treasury.to_account_info(),
            },
            &[seeds],
        ),
        reward,
    )?;

    state.last_distribution = now;
    state.total_minted = state.total_minted.checked_add(reward).unwrap();
    Ok(())
}
```

### Burn-and-Mint Equilibrium (BME)
Network usage burns tokens, creating demand that funds operator rewards. New tokens are minted to reward operators, and usage burns tokens — the equilibrium between burn and mint determines the token price.

**Pros**: Self-balancing, usage creates demand, aligns operators with network value
**Cons**: Complex to model, needs sufficient usage to sustain rewards
**Used by**: Helium (HNT → Data Credits), Hivemapper (HONEY → Map Credits)

```
┌─────────────────────────────────────────────────────────────┐
│                 Burn-and-Mint Equilibrium                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                    ┌──────────────┐                          │
│                    │  Token Pool   │                          │
│                    └──────┬───────┘                          │
│                           │                                   │
│           ┌───────────────┼───────────────┐                   │
│           │                               │                   │
│           ▼                               ▼                   │
│     ┌──────────┐                   ┌──────────┐              │
│     │  MINT    │                   │   BURN   │              │
│     │(Rewards) │                   │ (Usage)  │              │
│     └──────────┘                   └──────────┘              │
│           ▲                               ▲                   │
│           │                               │                   │
│    Operators provide               Customers consume         │
│    physical resources              network services          │
│                                                              │
│    If mint > burn → downward price pressure                  │
│    If burn > mint → upward price pressure                    │
│    Equilibrium: mint ≈ burn → stable price                   │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// TypeScript: BME reward calculation
interface BMEParams {
  totalBurnedPastPeriod: number;   // Tokens burned from usage
  operatorCount: number;
  qualityMultiplier: (deviceId: string) => number;
  targetBurnRatio: number;         // Target burn/mint ratio (e.g., 0.8)
}

function calculateOperatorReward(
  operatorContribution: number,
  totalContributions: number,
  params: BMEParams
): number {
  const targetMint = params.totalBurnedPastPeriod / params.targetBurnRatio;
  const operatorShare = operatorContribution / totalContributions;
  return targetMint * operatorShare * params.qualityMultiplier(operatorId);
}
```

### SaaS Revenue Buyback
Protocol generates revenue (subscriptions, API fees). Revenue buys tokens from the market → distributes to operators or burns.

**Pros**: Real revenue backs token value, sustainable long-term
**Cons**: Needs real revenue, complex tokenomics
**Used by**: UpRock ($UPT — SaaS revenue funds token buyback)

```
Revenue → Buyback tokens from market → Distribute to operators / Burn
```

## Emission Schedule Design

### Key Parameters

| Parameter | Description | Example (50K device network) |
|-----------|-------------|------------------------------|
| **Max Supply** | Hard cap (if fixed) | 1,000,000,000 tokens |
| **Initial Supply** | Tokens minted at genesis | 15% (150M for treasury, team, investors) |
| **Daily Emission** | Tokens minted per day for rewards | Start: 500K/day, halving every 2 years |
| **Halving Schedule** | How fast emissions decrease | 50% reduction every 2 years |
| **Operator Allocation** | % of emissions to operators | 70% |
| **Treasury Allocation** | % to protocol development | 20% |
| **Validator/Staking Allocation** | % to network security | 10% |

### Example: 4-Year Emission Schedule

```
Year 1: 500,000 tokens/day → 182.5M/year  (70% to operators = 350K/day)
Year 2: 500,000 tokens/day → 182.5M/year
Year 3: 250,000 tokens/day → 91.25M/year   (halving)
Year 4: 250,000 tokens/day → 91.25M/year
Year 5: 125,000 tokens/day → 45.6M/year    (halving)
...
Total over 10 years: ~685M tokens emitted (68.5% of max supply)
```

```typescript
// TypeScript: Emission schedule calculator
function calculateEmission(
  startTimestamp: number,
  currentTimestamp: number,
  initialDailyRate: number,  // 500_000
  halvingInterval: number,   // 2 years in seconds
): number {
  const elapsed = currentTimestamp - startTimestamp;
  const halvings = Math.floor(elapsed / halvingInterval);
  const dailyRate = initialDailyRate / Math.pow(2, halvings);
  const daysElapsed = elapsed / 86400;
  return dailyRate * daysElapsed;
}
```

## Reward Formula Patterns

### Flat Per-Device
Every device earns the same amount over time. Simple but doesn't differentiate quality.

```rust
pub fn calculate_reward(
    device_count: u64,
    daily_emission: u64,
    uptime_seconds: u64,
) -> u64 {
    // Naive: all devices split rewards equally
    if device_count == 0 { return 0; }
    let per_device_daily = daily_emission / device_count;
    per_device_daily * uptime_seconds / 86400
}
```

### Contribution-Weighted
Rewards proportional to contribution (data submitted, compute provided, bandwidth shared).

```rust
pub fn calculate_weighted_reward(
    operator_contribution: u64,
    total_contribution: u64,
    daily_emission: u64,
) -> u64 {
    if total_contribution == 0 { return 0; }
    daily_emission * operator_contribution / total_contribution
}
```

### Quality-Weighted
Like contribution-weighted, but with a quality multiplier.

```rust
pub fn calculate_quality_reward(
    operator_contribution: u64,
    total_contribution: u64,
    daily_emission: u64,
    quality_score: u8,  // 0-100
) -> u64 {
    if total_contribution == 0 { return 0; }
    let quality_multiplier = (quality_score as u128 + 100) / 100; // 1.0x to 2.0x
    let weighted_contribution = (operator_contribution as u128)
        .checked_mul(quality_multiplier)
        .unwrap();
    let weighted_total = (total_contribution as u128)
        .checked_mul(150u128) // average quality = 50 → 1.5x
        .unwrap();
    if weighted_total == 0 { return 0; }
    (daily_emission as u128)
        .checked_mul(weighted_contribution)
        .unwrap()
        .checked_div(weighted_total)
        .unwrap() as u64
}
```

### Uptime-Weighted
Rewards proportional to device uptime and reliability.

```rust
pub fn calculate_uptime_reward(
    uptime_hours: u64,
    total_possible_hours: u64,
    daily_emission: u64,
    device_count: u64,
) -> u64 {
    if device_count == 0 || total_possible_hours == 0 { return 0; }
    let uptime_ratio = uptime_hours as u128 * 100 / total_possible_hours as u128;
    // Bonus for >99% uptime, penalty for <90%
    let uptime_multiplier = match uptime_ratio {
        0..=50 => 0,           // Severe underperformance
        51..=90 => 80,         // Below target
        91..=98 => 100,        // Acceptable
        99 => 110,             // Great
        100 => 120,            // Perfect
        _ => 100,
    };
    let per_device_base = daily_emission as u128 / device_count as u128;
    (per_device_base * uptime_multiplier / 100) as u64
}
```

## Multi-Token Models

Some DePINs use multiple tokens for different functions:

### Helium Model (Parent + SubDAO Tokens)
- **HNT**: Base layer — earned by all operators, burned for Data Credits
- **IOT**: IoT subnet token — HNT emissions routed to IoT subDAO
- **MOBILE**: 5G subnet token — HNT emissions routed to Mobile subDAO
- **DC (Data Credits)**: Stable-value utility token — created by burning HNT

```
HNT Emission Pool
├── 33% → IoT SubDAO → IOT token → IoT hotspot operators
├── 33% → Mobile SubDAO → MOBILE token → 5G operators
└── 34% → HNT Stakers → Network validators

Usage: HNT → Burn → Data Credits → Pay for network data
```

### Dual Token Model
- **Reward Token**: Earned by operators, inflationary
- **Governance Token**: Fixed supply, grants voting rights, earned by staking

## Token Vesting for Team & Investors

Use Streamflow for customizable vesting schedules:

```typescript
import Streamflow from '@streamflow/stream';

// Team token vesting — 4-year linear vesting, 1-year cliff
const vestingContract = await Streamflow.createStream({
  sender: protocolTreasury,
  recipient: teamMember,
  amount: teamAllocation,       // e.g., 200M tokens
  period: 4 * 365 * 24 * 60 * 60, // 4 years
  cliffAmount: 0,
  cliffTime: 365 * 24 * 60 * 60,  // 1-year cliff
  canTopUp: false,
  cancellable: false,
  automaticWithdrawal: true,
});
```

## Key Security Considerations

1. **Oracle manipulation**: If rewards depend on price oracles, ensure fresh data and multiple sources
2. **Sybil attacks**: Prevent fake devices from claiming rewards (see [device-identity.md](device-identity.md))
3. **Re-entrancy**: On reward claims, use checks-effects-interactions pattern
4. **Rounding**: Integer division can accumulate over time. Use remainder tracking
5. **Pause mechanism**: Allow emergency pause if exploit detected
6. **Rate limiting**: Cap max reward per transaction to limit damage from bugs

## Related

- [reward-distribution.md](reward-distribution.md) — How to distribute rewards (streaming, Merkle, compressed)
- [device-identity.md](device-identity.md) — Anti-sybil for reward protection
- [oracle-integration.md](oracle-integration.md) — Price feeds for reward calculations
- solana-dev → [programs-anchor.md](../solana-dev/programs-anchor.md) — Anchor program patterns
