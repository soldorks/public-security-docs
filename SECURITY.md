# Security Policy — Sol Dorks

## Overview
Sol Dorks is a non-custodial Solana utility that allows users to identify and close empty SPL token accounts in order to reclaim locked SOL rent. The application operates entirely client-side and interacts with the Solana network using standard programs and user-approved wallet transactions.

Sol Dorks does not custody funds, store private keys, or perform any actions without explicit user consent.

## What Sol Dorks Can Do
- Read public wallet data (e.g. SPL token accounts) via a Solana RPC endpoint
- Construct transactions to close eligible empty SPL token accounts
- Submit user-signed transactions to the Solana network

## What Sol Dorks Cannot Do
- Access, store, or transmit private keys or seed phrases
- Sign transactions on behalf of users
- Move SOL or tokens without explicit wallet approval
- Execute transactions automatically or in the background

## Security Model
Sol Dorks follows a strict non-custodial security model:
- All transactions require explicit approval in the user’s wallet
- The application never has signing authority
- No backend services or custodial infrastructure are used
- Only standard Solana programs are invoked (System Program, SPL Token Program)

## Trust Assumptions
Users should be aware of the following trust assumptions:
- The connected wallet correctly displays transaction details and requires approval
- The RPC endpoint returns accurate and up-to-date on-chain data
- The Solana runtime enforces program constraints (e.g. CloseAccount behavior)

## Known Risks and Mitigations

### RPC Dependency
The application relies on public RPC endpoints for account discovery and transaction submission.

Mitigation:
- All state-changing actions require wallet approval
- Users can review transactions in their wallet before signing

### User Error
Users may accidentally attempt to close accounts they do not intend to.

Mitigation:
- The UI only surfaces accounts with a zero token balance
- Wallet confirmation acts as a final safeguard

### Unsupported Token Programs
Token accounts created under the Token-2022 program are not currently supported.

Mitigation:
- These accounts are ignored and cannot be closed through the interface

## Reporting a Vulnerability
If you believe you have discovered a security issue, please report it privately:

Email: security@soldorks.com  
Include a clear description of the issue and steps to reproduce if possible.

## Disclaimer
This documentation may reference public self-audit findings and automated tooling. It does not replace an independent third-party security audit. Undiscovered vulnerabilities may still exist.
