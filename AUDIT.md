# Sol Dorks — Public Self-Audit

Audit Type: Self-Audit  
Review Date: February 2026  
Scope: Client-side Solana application for closing empty SPL token accounts  
Version Reviewed: v1.0.0  

## Summary
This document presents a public self-audit of Sol Dorks, a non-custodial Solana utility that enables users to close empty SPL token accounts and reclaim locked SOL rent.

The goal of this self-audit is to document the application’s architecture, security properties, trust assumptions, and known limitations. This review is based on static analysis of the codebase and does not replace an independent third-party security audit.

## Architecture Overview
Sol Dorks is implemented as a fully client-side web application built with React, TypeScript, and Vite. The application does not operate backend services and interacts directly with the Solana network via standard RPC endpoints.

Wallet connectivity is handled through widely used Solana wallet adapters. Only the user’s public wallet address is accessed.

The application:
- Queries the Solana RPC for SPL token accounts owned by the connected wallet
- Filters accounts with a zero token balance
- Constructs a single atomic transaction per claim

Each transaction includes:
- An SPL Token Program `CloseAccount` instruction
- A System Program transfer instruction for a transparent service fee

All transactions require explicit user approval via the connected wallet.

## Non-Custodial Confirmation
The application is confirmed to be non-custodial:
- No private keys, seed phrases, or signing authority are accessed or stored
- All transactions are signed by the user’s wallet
- The application cannot execute transactions independently

## On-Chain Programs Used
Sol Dorks does not deploy or interact with custom on-chain programs. It relies exclusively on:
- SPL Token Program
- System Program

This minimizes attack surface and reduces protocol risk.

## Security Properties
- Explicit user approval is required for all transactions
- Transactions are atomic (all instructions succeed or fail together)
- CloseAccount instructions are enforced by the Solana runtime
- Token account ownership is enforced on-chain

## Findings

| Severity | Description | Status |
|--------|------------|--------|
| Critical | None identified | — |
| High | None identified | — |
| Medium | Reliance on RPC-provided account data | Accepted |
| Low | UI-driven account selection | Mitigated via wallet confirmation |

No critical or high-severity issues were identified during this review.

## Known Limitations

- Token-2022 accounts are not currently supported
- The application relies on a single public RPC endpoint
- Fee estimation uses conservative fixed values and may fail during congestion
- Token balance checks rely on parsed RPC data
- No transaction simulation preview is shown in the UI

These limitations do not result in custody risk but may impact reliability or user experience.

## Recommendations
Future improvements may include:
- Using raw token amount fields instead of UI-parsed values
- Adding fallback RPC endpoints
- Explicitly surfacing unsupported token account types
- Improving transaction preview visibility

## Disclosure
This self-audit was conducted without a third-party security firm. While care has been taken to identify obvious risks and document limitations, undiscovered vulnerabilities may still exist.

The Sol Dorks team intends to commission an independent third-party security audit once resources and usage justify the cost.
