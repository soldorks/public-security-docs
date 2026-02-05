# Security Policy — Sol Dorks

## Overview
Sol Dorks is a non-custodial Solana tool that scans a connected wallet for empty SPL token accounts and allows the user to close them to reclaim locked SOL rent deposits. Sol Dorks cannot move user funds without explicit wallet signatures.

## What Sol Dorks Can Do
- Read wallet public data (e.g., token accounts) through an RPC connection.
- Build transactions that include instructions to close eligible empty token accounts.
- Submit the user-signed transaction to the Solana network.

## What Sol Dorks Cannot Do
- Access or store private keys or seed phrases.
- Sign transactions on behalf of the user.
- Transfer tokens/SOL without the user approving the transaction in their wallet.

## Trust Assumptions
- The user’s wallet provider (e.g., Phantom/Solflare/etc.) correctly displays transaction details and requires explicit approval.
- The RPC endpoint used by the app is not malicious (or the user chooses a trusted RPC).
- The user reviews and approves only transactions they intend to sign.

## Threat Model (Common Risks)
### Malicious UI / Supply Chain
If the website or dependencies are compromised, an attacker could attempt to trick users into signing unwanted transactions.

Mitigations:
- Use a locked dependency strategy (lockfiles).
- Prefer audited, widely-used libraries.
- Verify production builds and deployment pipeline integrity.

### RPC Manipulation / Inaccurate Reads
A malicious RPC could return incorrect account data and attempt to influence what the UI displays.

Mitigations:
- The wallet signature requirement is the final gate.
- Users should verify transaction details in the wallet before signing.

### User Error
Users may accidentally close accounts or interact with the wrong wallet/network.

Mitigations:
- Clear UI warnings and transaction previews.
- Conservative defaults (only show accounts that are empty and safe to close).

## Upgradeability / Admin Controls
Sol Dorks is a client application. It does not custody funds.
If Sol Dorks interacts with any on-chain program(s), upgrade authority details should be documented publicly (Program IDs + whether upgrade authority is retained or renounced).

## Reporting a Vulnerability
If you discover a security issue, please report it privately:

- Email: security@soldorks.com (or replace with your preferred email)
- Include: steps to reproduce, expected vs. actual behavior, and any relevant screenshots/tx links.

We will acknowledge valid reports and aim to publish a fix and disclosure notes when appropriate.

## Disclaimer
This project’s documentation may include self-audit and automated tooling results. This does not replace an independent third-party security audit. Undiscovered vulnerabilities may still exist.
