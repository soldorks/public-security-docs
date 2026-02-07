# Sol Dorks — Public Self-Audit

**Audit Type:** Self-Audit
**Review Date:** February 2026
**Scope:** Client-side Solana application for burning dust tokens, closing empty token accounts, and reclaiming locked SOL rent
**Version Reviewed:** v1.1.0

---

## Summary

This document presents a public self-audit of Sol Dorks, a non-custodial Solana utility that enables users to burn dust tokens, close empty SPL Token and Token-2022 accounts, and reclaim locked SOL rent.

The goal of this self-audit is to document the application's architecture, security properties, trust assumptions, and known limitations. This review is based on static analysis of the codebase and does not replace an independent third-party security audit.

---

## Architecture Overview

Sol Dorks is implemented as a fully client-side web application built with React, TypeScript, and Vite. The application does not operate backend services and interacts directly with the Solana network via standard RPC endpoints.

Wallet connectivity is handled through Reown AppKit (formerly WalletConnect), supporting 120+ wallets natively including Phantom, Solflare, and Exodus. Only the user's public wallet address is accessed during a session.

The application:

- Queries the Solana RPC for SPL Token and Token-2022 accounts owned by the connected wallet
- Identifies accounts with zero balances (empty) and accounts with non-zero dust balances
- For dust accounts: constructs a Burn instruction to destroy the remaining balance, followed by a CloseAccount instruction
- For empty accounts: constructs a CloseAccount instruction only
- Appends a single System Program transfer instruction for a transparent 4.5% (450 BPS) service fee on total rent reclaimed
- Bundles all instructions into an atomic transaction (all succeed or all fail together)

All transactions require explicit user approval via the connected wallet before submission.

---

## Token Programs Supported

Sol Dorks supports both major Solana token standards:

| Program | Program ID | Support |
|---------|-----------|---------|
| SPL Token Program | `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` | Full |
| Token-2022 (Token Extensions) | `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` | Full |

Both programs are scanned in parallel during wallet analysis. Burn and CloseAccount instructions use the correct program ID for each respective account.

---

## Token Burning

When a token account contains a non-zero dust balance, the account cannot be closed directly (the Solana runtime enforces this). Sol Dorks handles this by constructing an atomic transaction that:

1. **Burns** the remaining dust token balance using opcode 8 (Burn instruction) with the raw amount as an 8-byte little-endian u64
2. **Closes** the now-empty account using opcode 9 (CloseAccount instruction), returning the rent SOL to the user
3. **Transfers** the service fee via System Program

All three steps are bundled into a single atomic transaction. If the burn fails, the close does not execute, and the user's account remains untouched.

Accounts requiring a burn are clearly flagged in the UI with a visible "burn" badge. Users can deselect any tokens they wish to keep before approving the transaction.

**Burning tokens is irreversible.** Once confirmed on the Solana blockchain, burned tokens cannot be recovered.

---

## Transaction Building

### Single Transaction Mode
For small claims, all instructions fit within a single Versioned (V0) transaction.

### Address Lookup Table (ALT) Optimisation
When a single transaction exceeds the 1,232-byte Solana limit, Sol Dorks creates a temporary Address Lookup Table to compress non-signer account addresses from 32 bytes to 1-byte indices. The ALT is deactivated after use.

### Chunked Transaction Fallback
If even with an ALT the instructions are too large, Sol Dorks splits accounts into multiple transaction chunks using conservative byte estimates:

- Burn + Close pair: ~180 bytes per account
- Close only: ~80 bytes per account
- Transaction overhead: ~200 bytes (header, blockhash, signatures, fee transfer)
- Budget per transaction: ~1,032 bytes

Each chunk is processed sequentially with a fresh blockhash and requires individual user approval.

---

## Non-Custodial Confirmation

The application is confirmed to be non-custodial:

- No private keys, seed phrases, or signing authority are accessed or stored
- All transactions are constructed client-side and signed by the user's wallet
- The application cannot execute transactions independently
- Wallet connection is handled through standard Solana wallet adapters via Reown AppKit
- No backend servers process or relay transactions

---

## On-Chain Programs Used

Sol Dorks does not deploy or interact with custom on-chain programs. It relies exclusively on:

| Program | Instructions Used |
|---------|------------------|
| SPL Token Program | `Burn` (opcode 8), `CloseAccount` (opcode 9) |
| Token-2022 Program | `Burn` (opcode 8), `CloseAccount` (opcode 9) |
| System Program | `Transfer` (for service fee and optional tips) |
| Address Lookup Table Program | `CreateLookupTable`, `ExtendLookupTable`, `DeactivateLookupTable` (for transaction size optimisation) |

This minimises attack surface and reduces protocol risk. No custom or third-party programs are invoked.

---

## Fee Structure

- **Service Fee:** 4.5% (450 basis points) of the total rent SOL reclaimed
- **Fee Calculation:** `Math.floor((totalRentLamports * 450) / 10_000)`
- **Fee Transfer:** A single System Program transfer instruction appended to each transaction
- **Fee Wallets:**
  - `BUgV4wyuS91Hb9eo3fFTb9ZkmQFwJga98cpnJ6jB3qhL`
  - `Fk3x2T8zJo2DbGFpAw8FjvBRzMNtig8PiuHMfcoo1zwj`
- **Transparency:** The fee percentage and amount are clearly displayed in the UI before the user approves any transaction

---

## Transaction Security

- **Preflight Simulation Enabled:** All transactions are submitted with `skipPreflight: false`, ensuring wallets (Phantom, Solflare, etc.) can simulate the transaction before the user signs. This prevents scam/phishing warnings and allows users to preview expected outcomes.
- **Atomic Execution:** All instructions within a transaction are atomic. If any instruction fails (e.g., a burn fails), the entire transaction reverts and the user's accounts remain unchanged.
- **Confirmation:** Transactions are confirmed at the `confirmed` commitment level with a 90-second timeout.
- **Rent Exemption Safety:** Before claiming, the app verifies the wallet will maintain the minimum rent-exempt balance (with a 20,000 lamport safety buffer) to prevent transaction failures.
- **Balance Validation:** Pre-claim checks ensure sufficient SOL for network fees (~5,000 lamports minimum).

---

## Optional Tip Functionality

After a successful claim, users may optionally send a tip to support the Sol Dorks team:

- Tips are entirely voluntary and user-initiated
- The user selects a tip amount and explicitly approves the transaction in their wallet
- Tips are sent as a standard System Program transfer to the fee wallet
- No tip is ever charged automatically
- Tip amounts and status are clearly displayed in the UI

---

## On-Chain Leaderboard

Sol Dorks features an on-chain leaderboard with no database:

- Built by scanning SystemProgram transfer history to the fee wallets
- Validates that each transaction contains a CloseAccount instruction (SPL Token or Token-2022) to filter out non-claim transfers
- Skips failed transactions (`tx.meta?.err` check)
- Aggregates fee transfers per-signature to prevent double-counting
- Merges batched/chunked transactions from the same wallet within a 60-second window
- Displays the top 20 wallets by total SOL claimed

### RPC Resilience

- **Retry Logic:** 2 retries per RPC call with exponential backoff (500ms base delay)
- **Rate Limiting:** Concurrency limited to 3 simultaneous requests with 100ms inter-batch delays
- **Error Handling:** Individual transaction fetch failures do not crash the leaderboard; failed fetches return null and are skipped

---

## Stats Dashboard

The Stats section displays three live on-chain metrics:

1. **Total SOL Reclaimed** (across both fee wallets)
2. **Total Claims** (number of claim transactions)
3. **Unique Wallets** (number of distinct claiming wallets)

Stats are computed by scanning both fee wallet transaction histories and merging the results. The section is non-critical: if RPC calls fail, the stats section fails silently without affecting other functionality.

---

## Receipt & Share System

Users can view and share transaction receipts at `/share/:sig`:

- Supports multiple transaction signatures (comma-separated) for batched claims
- Displays SOL reclaimed, fee paid, timestamp, and transaction slot
- Includes gamified character tiers based on SOL reclaimed (9 tiers)
- Receipts can be downloaded as PNG images or shared via the native Web Share API
- Links to Solscan for independent transaction verification

### Receipt Resilience

- Per-signature retry logic (3 attempts with 800ms exponential backoff)
- Partial data warning banner if some signatures fail to load
- Network fees are tracked and added back to gross reclaim calculation for accuracy
- Images are pre-converted to data URLs for reliable PNG export via `html-to-image`

---

## Security Properties

| Property | Status |
|----------|--------|
| Explicit user approval required for all transactions | Confirmed |
| Transactions are atomic (all-or-nothing execution) | Confirmed |
| Preflight simulation enabled (no skipPreflight bypass) | Confirmed |
| CloseAccount ownership enforced by Solana runtime | Confirmed |
| Burn instructions enforce token account authority on-chain | Confirmed |
| No custom on-chain programs deployed | Confirmed |
| Non-custodial (no access to private keys) | Confirmed |
| No backend servers (fully client-side) | Confirmed |
| Fee amount and percentage displayed before signing | Confirmed |
| Dust token burn clearly flagged in UI before approval | Confirmed |

---

## Findings

| Severity | Description | Status |
|----------|-------------|--------|
| Critical | None identified | — |
| High | None identified | — |
| Medium | Reliance on RPC-provided account data for balance and rent values | Accepted |
| Medium | Address Lookup Tables are created on-chain during large claims | Mitigated (ALTs are deactivated after use) |
| Low | UI-driven account selection for burn/close | Mitigated via wallet confirmation and burn badge UI |
| Low | Leaderboard and stats depend on RPC availability | Mitigated via retry logic and graceful degradation |
| Info | Tip functionality is post-claim and user-initiated | Acceptable |

No critical or high-severity issues were identified during this review.

---

## Known Limitations

- The application relies on Helius RPC as its primary endpoint with a public RPC fallback; extended outages on both could impact functionality
- Fee estimation uses a fixed 4.5% BPS calculation; network fee fluctuations during extreme congestion are not dynamically accounted for
- Token balance and rent values rely on parsed RPC data which could theoretically be manipulated by a compromised RPC provider
- Leaderboard scans are limited to the most recent transaction history available from the fee wallets (subject to RPC `getSignaturesForAddress` limits)
- Batched/chunked transactions are grouped using a 60-second time window heuristic, which may occasionally merge or split unrelated claims
- Address Lookup Tables incur a small on-chain creation cost (rent) during large claims
- Token metadata is fetched from Helius DAS API; unavailable metadata results in "unknown token" display

These limitations do not result in custody risk but may impact reliability or user experience in edge cases.

---

## Recommendations

Future improvements may include:

- Commissioning an independent third-party security audit
- Adding transaction simulation preview in the UI before wallet signing
- Implementing dynamic fee estimation based on current network conditions
- Adding fallback RPC endpoint rotation if the primary provider is unreachable
- Displaying estimated network fees alongside service fees before claiming
- Implementing automatic ALT cleanup/garbage collection for deactivated lookup tables
- Adding client-side validation that RPC-returned account data matches expected on-chain state

---

## Disclosure

This self-audit was conducted without a third-party security firm. While care has been taken to identify obvious risks and document limitations, undiscovered vulnerabilities may still exist.

Sol Dorks does not deploy or interact with custom on-chain programs, which significantly reduces the attack surface compared to applications that rely on custom smart contracts.

The Sol Dorks team intends to commission an independent third-party security audit once resources and usage justify the cost.

---

## Contact

For questions or to report security concerns, contact the Sol Dorks team:

- **X (Twitter):** [@thesoldorks](https://x.com/thesoldorks)
- **Instagram:** [@thesoldorks](https://www.instagram.com/thesoldorks/)
- **Telegram:** [soldorks](https://t.me/soldorks)
- **Email:** admin@soldorks.com
