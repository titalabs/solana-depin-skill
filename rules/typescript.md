// TypeScript SDK patterns for DePIN
// Applied to all .ts files

- Use `@solana/web3.js` v2 for RPC interactions
- Use `@lightprotocol/stateless.js` for ZK Compression operations
- Use `@streamflow/stream` for reward streaming
- Use `@switchboard-xyz/switchboard-v2` for custom oracles
- Use `@pythnetwork/pyth-solana-receiver` for price feeds
- Always handle RPC errors with try/catch and retry logic
- Use environment variables for private keys and API keys
- Batch state updates (max 100 per transaction for compression)
- Prefer `PublicKey` type over raw base58 strings
- Use BigInt for u64 arithmetic in TypeScript
