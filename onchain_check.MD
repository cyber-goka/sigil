# **Technical Guide: On-Chain Rug & Honeypot Detection for Solana Meme Coins**  
**Version 1.0**  
*For Trading Engine Developers*

---

## **1. Why This Matters**

- **~3.3% of new SPL tokens are honeypots** (source: [DexLabs, 2024](https://dexlab.space))
- **Common scam patterns**:
  - **Rug pull**: Dev removes liquidity → token value = $0
  - **Honeypot**: Buyers can’t sell (code blocks `transfer`)
  - **Mint authority retained**: Dev can inflate supply
  - **Hidden taxes**: 99% sell tax drains buyers

Your bot **must** check these **before allowing a buy**.

---

## **2. Step-by-Step Verification Flow**

When a new token appears (e.g., on pump.fun or Raydium), run this **5-point safety checklist**:

### ✅ **Check 1: Is Mint Authority Revoked?**
**Why**: If mint authority exists, dev can print infinite tokens → dilution.

**How**:
```ts
import { getMint } from '@solana/spl-token';

const mintInfo = await getMint(connection, tokenMint);
if (mintInfo.mintAuthority !== null) {
  throw new Error("MINT_AUTHORITY_NOT_REVOKED");
}
```

> ✅ **Pass**: `mintAuthority === null`  
> ❌ **Fail**: Any address present

---

### ✅ **Check 2: Is Freeze Authority Revoked?**
**Why**: Rarely used in meme coins, but if present, dev can freeze all wallets.

**How**:
```ts
if (mintInfo.freezeAuthority !== null) {
  throw new Error("FREEZE_AUTHORITY_NOT_REVOKED");
}
```

> ✅ **Pass**: `freezeAuthority === null`

---

### ✅ **Check 3: Analyze Token Program Ownership**
**Why**: Malicious tokens use **custom programs** (not standard SPL) to embed honeypot logic.

**How**:
1. Get the token’s **associated token account (ATA)** for a known wallet (e.g., Raydium LP)
2. Check the **owner** of that ATA

```ts
const ata = await getAssociatedTokenAddress(tokenMint, RAYDIUM_LP_WALLET);
const accountInfo = await connection.getAccountInfo(ata);
if (!accountInfo) throw new Error("NO_LP_FOUND");

// Owner should be Tokenkeg (SPL Token Program)
const SPL_TOKEN_PROGRAM_ID = "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA";
if (accountInfo.owner.toBase58() !== SPL_TOKEN_PROGRAM_ID) {
  throw new Error("NON_STANDARD_TOKEN_PROGRAM");
}
```

> ✅ **Pass**: Owner = `Tokenkeg...`  
> ❌ **Fail**: Custom program (e.g., `Honeypot111...`)

---

### ✅ **Check 4: Simulate a Sell Transaction**
**Why**: The only way to **confirm no honeypot** is to test if a sell would succeed.

**How**:
1. Create a **mock wallet** with 1 token
2. Build a **sell instruction** (via Jupiter or Raydium)
3. Use `connection.simulateTransaction()` with `sigVerify: false`

```ts
const mockWallet = Keypair.generate(); // Never fund this!
const sellTx = await buildSellTransaction(mockWallet, tokenMint, 1);

const simResult = await connection.simulateTransaction(sellTx, [], {
  sigVerify: false,
  replaceRecentBlockhash: true
});

if (simResult.value.err) {
  throw new Error(`HONEYPOT_DETECTED: ${JSON.stringify(simResult.value.err)}`);
}
```

> ✅ **Pass**: `err === null`  
> ❌ **Fail**: Any error (e.g., `Custom program error: 0x1` = common honeypot code)

> ⚠️ **Note**: Use a **fresh, unfunded wallet**. Never simulate with real funds.

---

### ✅ **Check 5: Check Liquidity & Ownership (Post-Launch)**
**Why**: Even if code is clean, dev can rug by pulling LP.

**How**:
- **For pump.fun tokens**: Check if migrated to Raydium
- **For Raydium pools**: Check if LP tokens are **burned or locked**

```ts
// Get Raydium LP token supply
const lpMint = raydiumPool.lpMint;
const lpSupply = await getTokenSupply(connection, lpMint);

// Check top holders of LP tokens
const largestAccounts = await connection.getTokenLargestAccounts(lpMint);
const topHolder = largestAccounts.value[0].address;

// If top holder is known burn address → good
const BURN_ADDRESSES = [
  "11111111111111111111111111111111", // System Program
  "5Q544fKrFoe6tsEbD7S8EmxGTJYAKtTVhAW5Q5pge4j1" // Common burn
];

if (!BURN_ADDRESSES.includes(topHolder.toBase58())) {
  console.warn("LIQUIDITY_NOT_LOCKED");
}
```

> ✅ **Ideal**: LP tokens burned → dev can’t remove liquidity  
> ⚠️ **Risky**: Dev wallet holds LP

---

## **3. Integration into Your Trading Bot**

### **Pre-Buy Safety Gate**
```ts
async function isTokenSafe(tokenMint: PublicKey): Promise<boolean> {
  try {
    await checkMintAuthorityRevoked(tokenMint);
    await checkFreezeAuthorityRevoked(tokenMint);
    await checkStandardTokenProgram(tokenMint);
    await simulateSell(tokenMint);
    await checkLiquiditySafety(tokenMint);
    return true;
  } catch (e) {
    console.error("Token unsafe:", e.message);
    return false;
  }
}

// In your buy flow
if (!await isTokenSafe(tokenMint)) {
  throw new Error("TOKEN_FAILED_SAFETY_CHECKS");
}
// Proceed to trade...
```

---

## **4. Advanced: Integrate Third-Party Scam APIs**

For extra confidence, layer in external checks:

| Service         | Endpoint                          | Use Case |
|-----------------|-----------------------------------|--------|
| **rugcheck.xyz** | `GET https://api.rugcheck.xyz/v1/tokens/{mint}/report` | Full audit score |
| **DexLab**       | `POST https://api.dexlab.space/v1/token/verify` | Honeypot detection |
| **SolanaFM**     | Token ownership explorer | Manual verification |

Example:
```ts
const rugcheck = await fetch(`https://api.rugcheck.xyz/v1/tokens/${mint}/report`);
const report = await rugcheck.json();
if (report.score < 80) {
  throw new Error("RUGCHECK_SCORE_TOO_LOW");
}
```

> 🔒 **Never rely solely on external APIs** — always run your own on-chain checks.

---

## **5. Performance & Latency Tips**

- **Cache results**: A token’s safety status rarely changes after 5 mins
- **Parallelize checks**: Run mint authority + program owner checks concurrently
- **Use private RPC**: Public RPCs fail under load → use Helius/QuickNode
- **Skip simulation for known-safe tokens**: Maintain allowlist of audited contracts

---

## **6. What This Catches**

| Threat                | Detected By |
|-----------------------|-------------|
| Mint inflation        | Check 1     |
| Wallet freezing       | Check 2     |
| Honeypot transfer lock| Check 4     |
| Fake token contracts  | Check 3     |
| Liquidity rug pull    | Check 5     |

---

## **7. Limitations**

- **New attack vectors** may emerge (e.g., time-locked rugs)
- **Simulation isn’t perfect**: Some honeypots only trigger after X holders
- **Social scams** (e.g., fake influencer) require AI/social layer (Phase 2)

---

## **8. Next Steps**

Once this safety layer is solid, you can **safely add**:
1. **Auto-buy on verified launches**
2. **AI sentiment analysis** (to prioritize which safe tokens to trade)
3. **$SIGIL subscriptions** for premium safety + alpha

---

This is your **immune system**. Build it first. 🔒
