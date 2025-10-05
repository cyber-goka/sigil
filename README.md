# sigil
Solana Trading Platform
# **Comprehensive Technical Specification: Solana Meme Coin Trading & Community Platform with Native Token Economy**  
**Version 1.0**  
*For Engineering, Product, and DevOps Teams*

---

## **1. Executive Summary**

This document defines a full-stack platform that unifies:
- **Real-time Solana meme coin monitoring and trading** (manual + automated)
- **Creator-driven community hubs** with monetization
- **A native SPL utility token (`$SIGIL`)** used exclusively for all subscriptions

The system replaces traditional trading fees with a **tokenized subscription model**, creating a closed-loop, deflationary economy aligned with Solana’s speed and composability.

---

## **2. System Architecture**

### **2.1. High-Level Diagram**
```
┌─────────────────────┐     ┌──────────────────────┐
│   Frontend (Web)    │<--->│   API Gateway        │
└─────────────────────┘     └──────────┬───────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
┌────────▼─────────┐        ┌──────────▼──────────┐       ┌──────────▼──────────┐
│ Trading Engine   │        │ Community Service   │       │ Auth & User Service │
│ - DEX Integration│        │ - Feeds             │       │ - Wallet Login      │
│ - Auto-Sell      │        │ - Subscriptions     │       │ - Profile Mgmt      │
│ - Sniper Logic   │        │ - Token-Gating      │       └─────────────────────┘
└────────┬─────────┘        └──────────┬──────────┘
         │                             │
         └──────────────┬──────────────┘
                        │
               ┌────────▼─────────┐
               │  Token Service   │ ←─── $SIGIL Economy
               │ - Balance Mgmt   │
               │ - Payments       │
               │ - Rewards        │
               └────────┬─────────┘
                        │
               ┌────────▼─────────┐
               │  Data Layer      │
               │ - PostgreSQL     │
               │ - Redis          │
               │ - S3 (Media)     │
               └────────┬─────────┘
                        │
               ┌────────▼─────────┐
               │  Blockchain      │
               │ - Solana RPC     │
               │ - Helius         │
               │ - Jupiter API    │
               │ - $SIGIL Mint    │
               └──────────────────┘
```

---

## **3. Core Services**

### **3.1. Trading Engine**
**Purpose**: Low-latency meme coin discovery and execution.

**Tech Stack**:
- **Language**: Rust (sniper logic), TypeScript (orchestration)
- **Libraries**: `@solana/web3.js`, `@jup-ag/api`, `helius-sdk`
- **Features**:
  - WebSocket monitoring of pump.fun → Raydium migrations
  - Jito bundle support for sub-100ms execution
  - Auto-sell with trailing stop-loss
  - Scam detection (rugcheck.xyz integration)

**Key Endpoints**:
```ts
POST /trade/buy
{
  wallet: string,        // User's public key
  tokenMint: string,     // SPL token address
  amountSol: number,
  slippage: number,      // e.g., 0.1 = 10%
  dex: "raydium" | "jupiter"
}

POST /trade/sell
{
  wallet: string,
  tokenMint: string,
  percentage: number     // e.g., 1.0 = 100%
}
```

---

### **3.2. Community Service**
**Purpose**: Creator profiles, community feeds, and engagement.

**Data Models**:

#### **CreatorProfile**
```prisma
model CreatorProfile {
  id            String    @id @default(uuid())
  userId        String    @unique
  displayName   String
  bio           String?
  verified      Boolean   @default(false)
  socialLinks   Json?     // { twitter: "...", telegram: "..." }
  walletAddress String?   // For on-chain verification
  subscriptionTiers SubscriptionTier[]
}
```

#### **SubscriptionTier**
```prisma
model SubscriptionTier {
  id          String   @id @default(uuid())
  creatorId   String
  name        String   // "Alpha Tier", "Free"
  priceSIGIL  Int      // e.g., 1000 = $10 at 1 $SIGIL = $0.01
  gatedToken  String?  // SPL mint for token-gating
  gatedAmount Int?     // min token balance required
  perks       Json     // ["early_access", "copy_trade"]
}
```

#### **Post**
```prisma
model Post {
  id          String   @id @default(uuid())
  creatorId   String
  content     String
  tradeData   Json?    // { tokenMint, amountSol, txHash }
  visibility  String   // "public" | "subscribers" | "tier:abc123"
  createdAt   DateTime @default(now())
}
```

**Key Endpoints**:
```ts
POST /community/posts
GET /community/feed?limit=20&cursor=...
POST /community/subscribe
```

---

### **3.3. Token Service** *(New)*
**Purpose**: Manage `$SIGIL` economy.

**Responsibilities**:
- Track user $SIGIL balances
- Process subscription payments
- Distribute creator rewards
- Handle buyback & burn

**Data Model**:
```prisma
model Subscription {
  id           String   @id @default(uuid())
  userId       String
  tierId       String
  tokenAmount  Int      // $SIGIL amount (e.g., 1000)
  startDate    DateTime
  endDate      DateTime
  status       String   // "active", "expired"
}
```

**Key Endpoints**:
```ts
GET /token/balance?wallet=...
POST /token/transfer // Internal only
POST /subscriptions // Purchase flow
```

---

### **3.4. Auth & User Service**
- **Wallet Login**: Solana Wallet Adapter (Phantom, Solflare)
- **Session**: JWT in HTTP-only cookies
- **Profile**: Multi-wallet linking, risk preferences

---

## **4. Native Token: `$SIGIL`**

### **4.1. Token Specifications**
| Property        | Value                                  |
|-----------------|----------------------------------------|
| **Name**        | Sigil Token                           |
| **Symbol**      | `$SIGIL`                               |
| **Decimals**    | 6                                      |
| **Total Supply**| 1,000,000,000                          |
| **Mint Authority** | **Revoked** at launch               |
| **Freeze Authority** | **Revoked**                        |

### **4.2. Initial Distribution**
| Allocation        | %     | Amount       | Use Case                     |
|-------------------|-------|--------------|------------------------------|
| Platform Treasury | 40%   | 400,000,000  | Subscriptions, rewards, dev  |
| Creator Rewards   | 25%   | 250,000,000  | Payouts to top creators      |
| User Airdrop      | 20%   | 200,000,000  | Free claims at launch        |
| Team              | 10%   | 100,000,000  | 4-year vesting               |
| Liquidity         | 5%    | 50,000,000   | Raydium pool (with SOL)      |

### **4.3. Subscription Pricing (Fixed Rate)**
| Tier    | USD Value | $SIGIL Cost |
|---------|-----------|-------------|
| Free    | $0        | 0           |
| Pro     | $10       | 1,000       |
| Elite   | $50       | 5,000       |

> **Rate**: 1 $SIGIL = $0.01 (adjusted quarterly via DAO).

---

## **5. Blockchain Integrations**

### **5.1. Real-Time Monitoring**
- **Provider**: Helius WebSockets
- **Programs Tracked**:
  - `Tokenkeg` (SPL Token Program)
  - `675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8` (pump.fun)

### **5.2. DEX Integration**
| DEX       | Method                     |
|-----------|----------------------------|
| Jupiter   | REST API v6 (`/quote`)     |
| Raydium   | SDK + CPI                  |
| Pump.fun  | IDL + account subscription |

### **5.3. $SIGIL Token Flow**
1. User holds $SIGIL in their wallet
2. On subscription:  
   - Frontend requests transfer approval
   - User signs SPL transfer to **Treasury Wallet**
   - Backend confirms → activates subscription

---

## **6. Infrastructure**

| Component          | Technology                          |
|--------------------|-------------------------------------|
| **Frontend**       | Next.js 14, Tailwind, Zustand       |
| **Backend**        | NestJS (TS) + Rust binaries         |
| **Database**       | PostgreSQL (Prisma)                 |
| **Cache**          | Redis (feeds, balances)             |
| **Storage**        | AWS S3                              |
| **RPC**            | Helius (Primary), QuickNode (Backup)|
| **Realtime**       | Socket.io                           |
| **Deployment**     | Docker + Kubernetes (EKS)           |
| **Monitoring**     | Datadog + Prometheus                |

---

## **7. Security**

- **No private keys stored**: All signing client-side
- **Mint authority revoked**: Immutable token supply
- **Rate limiting**: 100 req/min per IP
- **Transaction simulation**: Prevents malicious approvals
- **Idempotency keys**: Avoid duplicate subscription charges

---

## **8. APIs (Key Endpoints)**

### **8.1. Trade Execution**
```yaml
POST /trade/buy
Request:
  wallet: "9fX...abc"
  tokenMint: "EKp...xyz"
  amountSol: 5.0
  slippage: 0.1
```

### **8.2. Subscription Management**
```yaml
POST /subscriptions
Request:
  tierId: "tier_abc123"
  wallet: "9fX...abc"

Response:
  { id: "sub_xyz", endDate: "2025-06-01T00:00:00Z" }
```

### **8.3. Token Balance**
```yaml
GET /token/balance?wallet=9fX...abc
Response:
  { balance: 2500, decimals: 6 }
```

---

## **9. Deployment Roadmap**

| Phase | Milestones |
|------|-----------|
| **1. Core** | Deploy $SIGIL, build Trading Engine, basic UI |
| **2. Community** | Creator profiles, free posts, follower system |
| **3. Monetization** | $SIGIL subscriptions, creator payouts |
| **4. Scale** | Mobile app, DAO governance, CEX listings |

---

## **10. Appendix**

### **A. $SIGIL Mint Deployment (Rust Snippet)**
```rust
// Revoke mint authority by setting to system_program
invoke(
  &initialize_mint(
    &spl_token::ID,
    &mint_pubkey,
    &system_program::ID, // ← Revoked
    None,                // ← No freeze auth
    6,
  )?,
  &[mint_account, rent_sysvar],
)?;
```

### **B. Subscription Payment Flow**
1. User selects tier → frontend calculates $SIGIL cost
2. Backend checks balance via `getTokenAccountBalance`
3. If sufficient, prepare SPL transfer instruction
4. User signs → tx sent to network
5. On confirmation, activate subscription in DB

### **C. Treasury Wallet**
- Dedicated wallet for all $SIGIL revenue
- Funds used for:  
  - 85% → Creator payouts  
  - 15% → Platform (dev + buyback)

---

**Document Owner**: Sigil Platform  
**Last Updated**: Oct 2025  

This spec provides a complete, production-ready blueprint for a **tokenized Solana meme coin platform** that merges trading, community, and economics into one cohesive product. The native `$SIGIL` token eliminates friction, aligns incentives, and creates a defensible moat through network effects.
