// Anchor program patterns for DePIN
// Applied to all .rs files in Anchor programs


- Use `#[account]` with `#[derive(InitSpace)]` for account sizing
- Always validate account ownership with `require!` checks
- Use CPI for SPL Token operations (mint, transfer, burn)
- Use `Clock::get()?.unix_timestamp` for time-based operations
- Implement pausable pattern for emergency stops
- Always check oracle data freshness before using it
- Use `checked_add` / `checked_sub` / `checked_mul` for arithmetic safety
- Signer checks on all privileged operations
- For compressed accounts: use `helius_zk_compression::cpi` patterns
- For Merkle distribution: use `keccak256` hashing for proof verification
