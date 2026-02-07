# Security Policy — Sol Dorks

## Overview
Sol Dorks is a non-custodial Solana utility that allows users to burn dust tokens, close empty and dust token accounts across both the SPL Token and Token-2022 (Token Extensions) programs, and reclaim locked SOL rent. The application operates entirely client-side and interacts with the Solana network using standard on-chain programs and user-approved wallet transactions.

Sol Dorks does not custody funds, store private keys, or perform any actions without explicit user consent.

## What Sol Dorks Can Do
- Read public wallet data (SPL Token and Token-2022 accounts) via Solana RPC endpoints
- Fetch token metadata (names, symbols, images) via the Helius DAS API using public mint addresses
- Construct atomic transactions that burn dust token balances and close token accounts
- Create temporary Address Lookup Tables to optimise transaction size for large claims
- Split large claims into multiple chunked transactions when necessary
- Submit user-signed transactions to the Solana network
- Display an on-chain leaderboard and stats dashboard built from public fee wallet transaction history
- Generate shareable transaction receipts

## What Sol Dorks Cannot Do
- Access, store, or transmit private keys or seed phrases
- Sign transactions on behalf of users
- Move SOL or tokens without explicit wallet approval
- Execute transactions automatically or in the background
- Bypass wallet preflight simulation (all transactions use `skipPreflight: false`)

## Security Model
Sol Dorks follows a strict non-custodial security model:
- All transactions require explicit approval in the user's connected wallet
- The application never has signing authority over any funds
- No backend services, databases, or custodial infrastructure are used
- Only standard Solana programs are invoked (see On-Chain Programs below)
- Preflight simulation is enabled on all transaction submissions, allowing wallets to preview and validate transactions before the user signs
- All transactions are atomic — if any instruction fails (e.g., a burn fails), the entire transaction reverts and no accounts are modified

## On-Chain Programs Used
Sol Dorks does not deploy or interact with custom on-chain programs. It relies exclusively on:

| Program | Instructions Used |
|---------|------------------|
| SPL Token Program | `Burn` (opcode 8), `CloseAccount` (opcode 9) |
| Token-2022 Program | `Burn` (opcode 8), `CloseAccount` (opcode 9) |
| System Program | `Transfer` (service fee and optional tips) |
| Address Lookup Table Program | `CreateLookupTable`, `ExtendLookupTable`, `DeactivateLookupTable` |

## Fee Structure
- A transparent 4.5% (450 BPS) service fee is applied to the SOL rent reclaimed from closed accounts
- The fee amount is clearly displayed in the UI before the user approves any transaction
- The fee is included as a System Program transfer instruction within the same atomic transaction
- Optional post-claim tips are entirely voluntary, user-initiated, and require separate wallet approval

## Trust Assumptions
Users should be aware of the following trust assumptions:
- The connected wallet correctly displays transaction details and requires explicit approval
- The RPC endpoint returns accurate and up-to-date on-chain data
- The Solana runtime enforces program constraints (e.g., Burn authority, CloseAccount ownership)
- Token metadata returned by the Helius DAS API is accurate

## Known Risks and Mitigations

### Irreversible Token Burns
Dust tokens are permanently destroyed (burned) before an account can be closed. This is irreversible.

**Mitigation:**
- Accounts requiring a burn are clearly flagged with a visible "burn" badge in the UI
- Users can deselect any tokens they wish to keep before approving
- Wallet confirmation acts as a final safeguard before any burn executes

### RPC Dependency
The application relies on RPC endpoints (Helius primary, public node fallback) for account discovery, transaction submission, and on-chain data.

**Mitigation:**
- All state-changing actions require wallet approval regardless of RPC data
- Retry logic with exponential backoff is implemented across all RPC calls (leaderboard, receipts, stats)
- Concurrency is rate-limited to prevent RPC throttling
- Leaderboard and stats fail gracefully without affecting core claim functionality

### Transaction Size and Chunking
Large claims may exceed the 1,232-byte Solana transaction size limit and require splitting into multiple transactions.

**Mitigation:**
- Address Lookup Tables are used to compress transaction size when possible
- Conservative byte estimates ensure transactions stay within limits
- Each chunked transaction is individually signed and approved by the user
- ALTs are deactivated after use

### User Error
Users may accidentally select accounts they do not intend to close or burn.

**Mitigation:**
- The UI clearly separates empty accounts from dust accounts with burn badges
- Users review and approve the full transaction in their wallet before execution
- Pre-claim balance validation ensures the wallet maintains minimum rent-exempt balance

### Wallet Balance Safety
Claiming could theoretically leave a wallet below the minimum rent-exempt balance.

**Mitigation:**
- The app checks the wallet balance before constructing transactions
- A 20,000 lamport safety buffer is enforced above the minimum rent-exempt threshold
- A minimum of ~5,000 lamports is required for network fees

## Supported Wallets
Wallet connectivity is handled through Reown AppKit (formerly WalletConnect), supporting 120+ wallets natively including Phantom, Solflare, and Exodus.

## Reporting a Vulnerability
If you believe you have discovered a security issue, please report it privately:

**Email:** admin@soldorks.com
**X (Twitter):** [@thesoldorks](https://x.com/thesoldorks)

Include a clear description of the issue and steps to reproduce if possible.

## Disclaimer
This security policy documents the application's architecture, trust model, and known risks. It does not replace an independent third-party security audit. While care has been taken to identify and mitigate risks, undiscovered vulnerabilities may still exist. Use Sol Dorks at your own risk.
