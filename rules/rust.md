// Rust program patterns for DePIN
// Applied to all .rs files in non-Anchor programs

- Use `solana_program` types for raw program development
- Prefer Anchor framework for new programs
- Use `msg!()` macro sparingly (compute unit cost)
- Implement `checked_` arithmetic on all numeric operations
- Use `Pubkey::find_program_address` for PDA derivation
- Respect compute budget limits (max 200K CU per instruction for DePIN)
- For large state: prefer off-chain storage with Merkle commitments
- Use `invoke_signed` for CPI with PDA signers
- Validate all account discriminators before reading
- Use `freeze` / `thaw` patterns for emergency controls
