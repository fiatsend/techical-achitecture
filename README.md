# Fiatsend — Technical Architecture (Stellar / Soroban)

This repository contains Fiatsend’s technical architecture documentation for integrating with the Stellar ecosystem (Anchor Platform SEPs, Soroban, and key DeFi building blocks).

## Documents

- **SCF #41 — Integration Track**: [`Fiatsend-Technical-Architecture-SCF41-Integration.md`](./Fiatsend-Technical-Architecture-SCF41-Integration.md)

## Budget + deeper dive (supporting docs)

- **SCF #41 Integration budget (spreadsheet)**: [Fiatsend-SCF41-Integration-Budget-98k](https://docs.google.com/spreadsheets/d/1jMvARRxdBvZ_Wk2_2V1ncMwiOPEjkVvK/edit?usp=sharing&ouid=116734387907050013778&rtpof=true&sd=true)
- **More detailed architecture (Google Doc)**: [Fiatsend - Technical Architecture](https://docs.google.com/document/d/1oT5Ydl6UaGXqF9cLsIUQ8ClcWD3Ugx6p/edit?usp=sharing&ouid=116734387907050013778&rtpof=true&sd=true)

## What Fiatsend is building (high level)

- **Phone-number-first UX** for Stellar: phone login + account abstraction patterns, mapping phone identifiers to on-chain accounts.
- **On/off-ramp via Anchor Platform**: interactive deposit/withdraw (SEP-24) and cross-border receive (SEP-31), discovered through `stellar.toml` (SEP-1).
- **In-app swaps and yield**: swaps routed through Soroswap; yield via DeFindex vaults backed by Blend.

## Helpful references (canonical docs)

- **SEP-1 (stellar.toml)**: `https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0001.md`
- **SEP-24 (Hosted Deposit & Withdrawal)**: `https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0024.md`
- **SEP-31 (Cross-Border Payments)**: `https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0031.md`
- **Anchor Platform docs**: `https://developers.stellar.org/docs/platforms/anchor-platform`

## Contributing

Small, focused improvements are welcome (typos, clarity, diagrams, risk notes, and implementation details). If you’re proposing a material change, please include:

- what problem it solves
- the assumption(s) it relies on
- the impact on security / compliance / user UX

