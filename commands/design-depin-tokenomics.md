---
description: "Design token economics for a DePIN project"
---

You are designing token economics for a Solana DePIN (Decentralized Physical Infrastructure Network). Follow these steps:


## Step 1: Gather Requirements

Ask the user about:
- **Physical resource**: What is being shared? (wireless, compute, bandwidth, data, storage)
- **Target scale**: How many devices? (100, 1K, 10K, 100K, 1M+)
- **Revenue model**: Is there a service to sell? (data credits, API access, subscriptions)
- **Operator type**: Who operates devices? (crypto-native? general consumers? enterprises?)
- **Token purpose**: Reward only? Also governance? Utility? All three?

## Step 2: Generate Tokenomics Spec

Create a comprehensive tokenomics specification:

### Token Model

```markdown
## Token Supply Model

**Recommended**: [Burn-and-Mint Equilibrium | Inflationary | Fixed Supply | SaaS Buyback]

### Parameters
| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Max Supply | [X tokens] | [Explanation] |
| Initial Supply | [X%] | [For treasury, team, investors] |
| Daily Emission | [X/day] | [Based on target operator count] |
| Halving | [Every X years] | [Inflation control] |
| Operator Allocation | [X%] | [Rewards for device operators] |
| Treasury Allocation | [X%] | [Protocol development] |
| Staking Allocation | [X%] | [Network security incentives] |
```

### Reward Formula

```markdown
## Reward Formula

**Formula**: [Flat | Contribution-Weighted | Quality-Weighted | Uptime-Weighted]

reward = base_rate × quality_multiplier × uptime_multiplier

### Parameters
| Parameter | Value | Description |
|-----------|-------|-------------|
| Base Rate | [X tokens/day] | Per-device base |
| Quality Multiplier | [0.5x-2.0x] | Based on trust score |
| Uptime Multiplier | [0x-1.2x] | Based on uptime % |
```

## Step 3: Generate Code

### Emission Schedule Contract

```rust
// Anchor program: Time-based emission
pub fn calculate_emission(ctx: Context<CalculateEmission>) -> Result<()> {
    // ... see skill/token-rewards.md for patterns
}
```

### Streamflow Integration

```typescript
// See skill/reward-distribution.md for Streamflow setup
```

## Step 4: Cost Analysis

```markdown
## Cost Analysis

| Metric | Value |
|--------|-------|
| Annual reward budget | [X tokens] |
| Annual distribution cost | [X SOL] |
| Cost per device per year | [X SOL] |
| Breakeven token price | [$X] |
```

## Step 5: Deliverables

- `tokenomics.md` — Full tokenomics specification
- Anchor program for reward calculation
- Streamflow integration script
- Cost analysis spreadsheet
