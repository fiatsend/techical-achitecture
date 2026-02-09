# Fiatsend Technical Architecture — SCF #41 Integration Track

_Status_: Living document (implementation details will evolve during delivery)  
_Last updated_: 2026-02-08

## Table of contents

- [1. Introduction](#1-introduction)
- [2. Integration Summary](#2-integration-summary)
- [3. Architecture Overview](#3-architecture-overview)
- [4. Stellar Building Block Integrations (Detailed)](#4-stellar-building-block-integrations-detailed)
  - [4.1 Anchor Platform — SEP-1, SEP-24, SEP-31](#41-anchor-platform--sep-1-sep-24-sep-31)
  - [4.2 Soroswap — DEX Aggregation](#42-soroswap--dex-aggregation)
  - [4.3 DeFindex + Blend Protocol — On-Chain Yield](#43-defindex--blend-protocol--on-chain-yield)
- [5. Soroban Smart Contracts](#5-soroban-smart-contracts)
- [6. Technology Stack](#6-technology-stack)
- [7. Security Architecture](#7-security-architecture)
- [8. Deployment & Scalability](#8-deployment--scalability)
- [9. Integration Timeline (Mapped to Tranches)](#9-integration-timeline-mapped-to-tranches)
- [10. How This Benefits the Stellar Ecosystem](#10-how-this-benefits-the-stellar-ecosystem)
- [11. Reference links](#11-reference-links)

## 1. Introduction

Fiatsend is a crypto-native payment platform that provides mobile-money-like stablecoin accounts to users in emerging markets. This document describes Fiatsend's technical plan to integrate with existing Stellar ecosystem building blocks — specifically the **Anchor Platform (SEP-1/24/31)**, **DeFindex**, **Blend Protocol**, and **Soroswap** — as part of our SCF #41 Integration Track submission.

Fiatsend is currently live on an EVM-compatible chain at app.fiatsend.network with 12,000+ users and $3M+ in processed volume. The goal of this integration is to migrate core payment and yield functionality onto Stellar and Soroban, leveraging the network's fast finality, low fees, and composable DeFi ecosystem.

---

## 2. Integration Summary

| Stellar Building Block | Integration Purpose | Tranche |
|---|---|---|
| **Anchor Platform (SEP-1)** | Publish Fiatsend asset & org metadata via stellar.toml | T1 |
| **Anchor Platform (SEP-24)** | Interactive hosted deposit/withdrawal — USDC ↔ local stablecoin via mobile money | T1 |
| **Anchor Platform (SEP-31)** | Cross-border receive — external Stellar wallets send to Fiatsend users for local mobile money payout | T2 |
| **Soroswap** | In-app DEX aggregation for USDC ↔ XLM swaps | T1 |
| **DeFindex** | Yield vaults — route idle user USDC into autocompound strategies | T2 |
| **Blend Protocol** | Underlying lending pool powering DeFindex vault yield | T2 |
| **Stellar SDK** | Wallet creation, signing, balance queries mapped to phone-number accounts | T1 |
| **Soroban Smart Contracts** | MobileNumber NFT identity, payment routing, programmable conversion logic | T1–T2 |

---

## 3. Architecture Overview

### 3.1 High-Level System Design

```
┌─────────────────────────────────────────────────────┐
│                  USER LAYER                          │
│   Mobile App (React Native) / Web App (Next.js 14)  │
│   Phone-number login → Privy + Stellar SDK          │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│               FIATSEND API LAYER                     │
│   Node.js API Routes (Vercel) + Appwrite Cloud       │
│   Auth · KYC · Payments · Notifications · Analytics  │
└──┬──────────┬──────────┬──────────┬─────────────────┘
   │          │          │          │
   ▼          ▼          ▼          ▼
┌──────┐ ┌────────┐ ┌────────┐ ┌──────────────────┐
│Anchor│ │Soroswap│ │DeFindex│ │  Mobile Money     │
│Platfm│ │  DEX   │ │+ Blend │ │  PSP Gateway      │
│SEP-1 │ │Aggregtr│ │ Vaults │ │  (MTN/Telecel/    │
│SEP-24│ │USDC↔XLM│ │  USDC  │ │   AirtelTigo)     │
│SEP-31│ │        │ │  Yield │ │                    │
└──┬───┘ └───┬────┘ └───┬────┘ └────────┬───────────┘
   │         │          │               │
   ▼         ▼          ▼               ▼
┌─────────────────────────────────────────────────────┐
│              STELLAR / SOROBAN LAYER                 │
│  Stellar Network (Horizon API)                       │
│  Soroban Contracts:                                  │
│   • MobileNumber NFT (identity + routing)            │
│   • Conversion Contract (USDC → local stablecoin)    │
│   • Payment Link Contract                            │
│  Native Assets: USDC, XLM, local stablecoin tokens   │
└─────────────────────────────────────────────────────┘
```

### 3.2 Key Design Principles

- **Composability-first:** Every DeFi feature (yield, swaps) uses existing Stellar building blocks rather than custom implementations.
- **Phone-number UX:** All on-chain interactions are abstracted behind phone-number accounts via MobileNumber NFTs.
- **Modular mobile money:** PSP adapters are pluggable per market (currently Ghana; extensible to other mobile-money-dominant markets).
- **Compliance-native:** KYC tiers (Level 0–2) enforced at the contract level via NFT metadata.

---

## 4. Stellar Building Block Integrations (Detailed)

### 4.1 Anchor Platform — SEP-1, SEP-24, SEP-31

**What it is:** Stellar's standard infrastructure for fiat on/off-ramps and cross-border payments.

**How Fiatsend integrates:**

**Note:** While this document highlights SEP-1/24/31, a production Anchor Platform integration typically relies on additional SEPs (e.g., SEP-10 auth and SEP-12 customer/KYC flows, and optionally SEP-38 quotes). Fiatsend’s plan is to use Anchor Platform’s standard flows where possible and keep business logic (KYC tiering, limits, PSP adapters) in Fiatsend-controlled services.

**SEP-1 (stellar.toml):**
- Publish Fiatsend's organization metadata, supported assets (USDC, local stablecoin), and SEP-24/31 service endpoints.
- Hosted at fiatsend.network/.well-known/stellar.toml.

**SEP-24 (Interactive Deposit & Withdrawal):**
- **Deposit flow:** User initiates deposit via any Stellar wallet → Fiatsend's SEP-24 server presents a hosted UI → user sends USDC → Fiatsend credits local stablecoin to user's phone-number account.
- **Withdrawal flow:** User requests withdrawal in Fiatsend app → SEP-24 server initiates mobile money payout via PSP gateway → user receives funds on their mobile money wallet.
- Fiatsend's backend acts as the SEP-24 anchor server, handling interactive auth, transaction status callbacks, and KYC checks (tied to MobileNumber NFT tier).

**SEP-31 (Cross-Border Payments):**
- External sending wallets/exchanges initiate a SEP-31 payment to Fiatsend as the receiving anchor.
- Fiatsend receives the USDC, converts to local stablecoin, and pays out to the recipient's mobile money account using their phone number as the identifier.
- The sending institution never needs to integrate with local mobile money rails — Fiatsend handles last-mile delivery.

**Integration sequence (SEP-24 withdrawal):**
```
User → Fiatsend App: "Withdraw 50 USDC to mobile money"
Fiatsend App → Anchor Server: POST /sep24/transactions/withdraw/interactive
Anchor Server → User: Hosted withdrawal UI (amount, mobile money details)
User → Anchor Server: Confirms withdrawal
Anchor Server → Stellar: Submit USDC transfer to Fiatsend hot wallet
Anchor Server → PSP Gateway: Initiate mobile money payout
PSP Gateway → Anchor Server: Payout confirmed
Anchor Server → Stellar: Update transaction status
Fiatsend App → User: Push notification "Withdrawal complete"
```

### 4.2 Soroswap — DEX Aggregation

**What it is:** The first DEX aggregator on Stellar, enabling optimal token swaps across multiple liquidity sources.

**How Fiatsend integrates:**
- Fiatsend's backend calls the Soroswap Router contract to execute USDC ↔ XLM swaps on behalf of users.
- Users see a simple "Swap" button in-app; Soroswap handles routing, slippage, and best-price execution under the hood.
- This enables users to acquire XLM for transaction fees or diversify holdings without leaving Fiatsend.

**Integration flow:**
```
User → Fiatsend App: "Swap 10 USDC to XLM"
Fiatsend Backend → Soroswap Router Contract: get_best_route(USDC, XLM, amount)
Soroswap Router → Fiatsend Backend: Returns optimal route + quote
Fiatsend Backend → Soroban: Submit swap transaction (signed by user's keypair)
Soroban → Fiatsend Backend: Swap confirmed
Fiatsend App → User: Updated balances shown
```

### 4.3 DeFindex + Blend Protocol — On-Chain Yield

**What it is:** DeFindex provides plug-and-play vault infrastructure on Stellar. Blend Protocol is a lending/borrowing protocol. DeFindex vaults can deploy capital into Blend pools to earn yield automatically.

**How Fiatsend integrates:**
- Fiatsend creates (or connects to) a DeFindex vault configured with a Blend USDC lending strategy.
- Users tap "Earn" in the Fiatsend app → their idle USDC is deposited into the DeFindex vault → vault deploys into Blend → yield accrues automatically.
- Users can withdraw at any time; vault shares (dfTokens) are held in their Stellar account.
- Fiatsend displays real-time APY, deposited balance, and accrued yield in the wallet UI.

**Integration flow:**
```
User → Fiatsend App: "Deposit 100 USDC to Earn"
Fiatsend Backend → DeFindex Vault Contract: deposit(user_address, 100 USDC)
DeFindex Vault → Blend Pool: Supply 100 USDC to lending pool
Blend Pool → DeFindex Vault: Confirm supply, start accruing interest
DeFindex Vault → User's Stellar Account: Issue dfTokens (vault shares)
Fiatsend App → User: "Earning X% APY on 100 USDC"

// Withdrawal
User → Fiatsend App: "Withdraw from Earn"
Fiatsend Backend → DeFindex Vault Contract: withdraw(user_address, dfToken_amount)
DeFindex Vault → Blend Pool: Redeem USDC + accrued interest
DeFindex Vault → User's Stellar Account: Return USDC
Fiatsend App → User: Updated balance with earned yield
```

**Why this matters:** Traditional mobile money offers 0% yield. By composing DeFindex + Blend, Fiatsend users earn real on-chain returns on idle balances — a powerful retention and growth mechanism that differentiates Fiatsend from every mobile money provider.

---

## 5. Soroban Smart Contracts

### 5.1 MobileNumber NFT Contract

Maps phone numbers to Stellar addresses, enabling phone-number-based payment routing across the Stellar network.

```
contract MobileNumberNFT {
    // Mint NFT at signup, tied to phone number hash
    fn mint(user: Address, phone_hash: BytesN<32>, kyc_tier: u32) -> Result<TokenId, Error>;

    // Lookup: resolve phone number to Stellar address
    fn resolve(phone_hash: BytesN<32>) -> Result<Address, Error>;

    // Upgrade KYC tier (Level 0 → 1 → 2)
    fn upgrade_tier(token_id: TokenId, new_tier: u32) -> Result<(), Error>;

    // Check KYC tier for compliance gating
    fn get_tier(token_id: TokenId) -> Result<u32, Error>;
}
```

**Used by:** SEP-24/31 flows (to resolve recipients), Fiatsend P2P payments, external wallet routing.

### 5.2 Conversion Contract

Handles stablecoin-to-local-stablecoin conversion with rate management.

```
contract FiatsendConverter {
    // Convert USDC to local stablecoin
    fn convert(user: Address, amount: u128, target_asset: Address) -> Result<u128, Error>;

    // Update conversion rate (admin, oracle-fed)
    fn update_rate(from: Address, to: Address, rate: u128) -> Result<(), Error>;

    // Get current rate
    fn get_rate(from: Address, to: Address) -> Result<u128, Error>;
}
```

### 5.3 Payment Link Contract

Generates unique on-chain payment addresses for receiving crypto with auto-conversion.

```
contract PaymentLink {
    // Create a payment link for a user
    fn create(user: Address, auto_convert: bool, target_asset: Address) -> Result<Address, Error>;

    // Process incoming payment and trigger conversion
    fn process_incoming(link_address: Address, amount: u128, asset: Address) -> Result<(), Error>;
}
```

---

## 6. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14, TypeScript, Tailwind CSS, Stellar SDK, Privy |
| Backend | Node.js 18+, Next.js API Routes, Appwrite Cloud |
| Blockchain | Stellar Network, Soroban Smart Contracts, Horizon API |
| Stellar Integrations | Anchor Platform (SEP-1/24/31), Soroswap, DeFindex, Blend |
| Mobile Money | PSP gateway adapters (OAuth2/HMAC/JWT per provider) |
| Identity | Veriff KYC + MobileNumber NFT on Soroban |
| Security | AES-256 encryption, TLS 1.3, multi-sig wallets, JWT auth |
| Deployment | Vercel, Appwrite Cloud, Cloudflare CDN/DDoS |

---

## 7. Security Architecture

- **Wallet security:** AES-256-GCM encrypted private keys with PBKDF2 key derivation. Multi-sig (user + Fiatsend + backup) for high-value operations.
- **Auth:** JWT + role-based access (User/Admin/Support). Privy for social/phone login abstraction.
- **Compliance:** KYC tier enforced at contract level — MobileNumber NFT tier gates feature access (e.g., withdrawal limits scale with KYC level).
- **Fraud detection:** Risk-scored transactions with automatic review/block thresholds.
- **Rate limiting:** Per-user limits on conversions (10/hr), withdrawals (5/day), API calls (1000/hr).
- **Data protection:** GDPR-compliant PII handling, encrypted at rest, audit-logged.

---

## 8. Deployment & Scalability

- **Production:** Vercel (frontend + API) + Appwrite Cloud (DB + auth + storage) + Cloudflare (CDN, DDoS, SSL).
- **Stellar:** Horizon API for queries, Soroban RPC for contract calls, testnet for staging.
- **Scaling:** Stateless API routes auto-scale on Vercel. Appwrite supports read replicas and sharding. Redis cache layer for frequently accessed data (balances, rates).
- **Monitoring:** Custom analytics dashboard tracking transaction volume, conversion success rates, vault TVL, mobile money provider SLAs, and Stellar network health.

---

## 9. Integration Timeline (Mapped to Tranches)

| Phase | Timeframe | Building Blocks | Deliverable |
|---|---|---|---|
| T0 | Upon approval | — | Project setup, infrastructure provisioning, Anchor Platform scaffolding |
| T1 | Weeks 1–8 (by Apr 2026) | Anchor Platform (SEP-1/24), Stellar SDK, Soroswap | Testnet on/off-ramp + in-app swaps + phone-number wallets |
| T2 | Weeks 9–14 (by May 2026) | Anchor Platform (SEP-31), DeFindex, Blend | Staging cross-border receive + yield vaults + payment routing |
| T3 | Weeks 15–18 (by Jun 2026) | All | Mainnet launch, production yield, user testing, documentation |

**Budget breakdown:** [Fiatsend-SCF41-Integration-Budget-98k (Google Sheet)](https://docs.google.com/spreadsheets/d/1jMvARRxdBvZ_Wk2_2V1ncMwiOPEjkVvK/edit?usp=sharing&ouid=116734387907050013778&rtpof=true&sd=true)

---

## 10. How This Benefits the Stellar Ecosystem

- **Real anchor volume:** 12,000+ existing users generating real deposit/withdrawal flow through SEP-24/31.
- **DeFi TVL:** Idle user USDC routed into Blend via DeFindex increases protocol TVL and utilization.
- **Swap volume:** In-app USDC ↔ XLM swaps via Soroswap add organic trading volume.
- **New user segment:** Mobile-money-first users in emerging markets who have never used Stellar before — onboarded via familiar phone-number UX.
- **Composability showcase:** Fiatsend demonstrates real-world composability across 4+ Stellar building blocks in a single consumer product.

---

## 11. Reference links

- **SEP-1 (Stellar Info File / `stellar.toml`)**: `https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0001.md`
- **SEP-24 (Hosted Deposit & Withdrawal)**: `https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0024.md`
- **SEP-31 (Cross-Border Payments)**: `https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0031.md`
- **Anchor Platform docs**: `https://developers.stellar.org/docs/platforms/anchor-platform`
- **SCF #41 Integration budget**: `https://docs.google.com/spreadsheets/d/1jMvARRxdBvZ_Wk2_2V1ncMwiOPEjkVvK/edit?usp=sharing&ouid=116734387907050013778&rtpof=true&sd=true`
- **More detailed architecture doc**: `https://docs.google.com/document/d/1oT5Ydl6UaGXqF9cLsIUQ8ClcWD3Ugx6p/edit?usp=sharing&ouid=116734387907050013778&rtpof=true&sd=true`
