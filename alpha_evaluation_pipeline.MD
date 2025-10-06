# **Post On-Chain Safety: The Alpha Evaluation Pipeline**  
*What to check after confirming a token isn’t a rug*

This is where your bot shifts from **"Is this safe?"** → **"Is this promising?"**

---

## ✅ **Stage 2: Fundamental & Market Health Checks**

Run these **immediately after** on-chain safety passes.

### **1. Liquidity Depth & Stability**
- **Minimum threshold**: ≥10 SOL in LP (adjustable per strategy)
- **Check**: Raydium or Orca pool reserves
- **Why**: Low liquidity = high slippage + easy manipulation

```ts
const pool = await getRaydiumPool(tokenMint);
const solReserve = pool.solAmount; // e.g., 25.4 SOL
if (solReserve < MIN_LIQUIDITY_SOL) {
  return "LOW_LIQUIDITY";
}
```

> 💡 **Pro Tip**: Track **liquidity growth rate** — sudden LP adds often precede pumps.

---

### **2. Holder Distribution & Growth**
- **Healthy sign**: Rapid, organic holder increase
- **Red flag**: 1–2 wallets hold >50% supply

**How to check**:
```ts
const holders = await connection.getTokenLargestAccounts(tokenMint);
const top3Pct = holders.value
  .slice(0, 3)
  .reduce((sum, acc) => sum + acc.amount, 0n) / totalSupply;

if (top3Pct > 0.6) {
  return "CENTRALIZED_HOLDERS";
}
```

Also track **new holders per minute** (via Helius webhook on `TokenAccount` creations).

> 📈 **Ideal**: 50+ new holders in first 2 minutes post-launch.

---

### **3. Social & Narrative Signal (Lightweight)**
Even without AI, you can detect **basic hype**:

| Signal | How to Detect |
|-------|---------------|
| **X (Twitter) mentions** | Use Twitter API v2: count recent tweets with `$TOKEN` or contract |
| **Telegram group size** | Scrape public group member count (if link available) |
| **pump.fun description** | Check for keywords: “dog”, “cat”, “based”, “CEX soon” |

> 🔍 **No LLM needed**: Simple regex + count thresholds.

Example:
```ts
if (pumpFunDescription.match(/(dog|woof|bark)/i)) {
  narrativeScore += 20; // Dog coins trend often
}
if (twitterMentionsLast5Min > 50) {
  narrativeScore += 30;
}
```

---

### **4. Price Action & Volume Spike**
- **Buy signal**: Sudden volume surge with price stability
- **Avoid**: “Pump and dump” — price spikes 1000% in 10 seconds then crashes

**Track**:
- Volume (SOL) in last 60 seconds
- Price change % (vs. 30-sec ago)
- Number of trades

> 📊 **Rule of thumb**: Prefer **steady 10x over 1 min** vs. **100x in 5 sec**.

---

### **5. Dev Behavior (On-Chain)**
Even if mint is revoked, check **what the deployer did**:
- Did they **buy early**? (Good sign — skin in the game)
- Did they **immediately sell**? (Bad sign)
- Is their wallet **verified as a known builder**? (e.g., past successful launches)

Use **Helius Enhanced Transactions** to label the creator wallet.

---

## 🧠 **Stage 3: Scoring & Decision Engine**

Combine all signals into a **Trade Score (0–100)**:

| Factor                  | Weight | Max Points |
|------------------------|--------|------------|
| On-chain safety        | 30%    | 30         |
| Liquidity depth        | 20%    | 20         |
| Holder growth          | 20%    | 20         |
| Social/narrative       | 15%    | 15         |
| Price/volume stability | 15%    | 15         |

```ts
const tradeScore = 
  (safetyScore * 0.30) +
  (liquidityScore * 0.20) +
  (holderScore * 0.20) +
  (narrativeScore * 0.15) +
  (priceStabilityScore * 0.15);

if (tradeScore >= 75) {
  triggerBuySignal(tokenMint, tradeScore);
}
```

> ⚙️ **User-configurable**: Let users set their own thresholds (e.g., “Only trade if score > 80”).

---

## 🚦 **Final Output: Actionable Signal**

Your system now outputs one of:

| Signal | Action |
|--------|--------|
| `SAFE_BUT_DEAD` | Ignore (no momentum) |
| `PROMISING_ALPHA` | Notify user / auto-buy (if enabled) |
| `HIGH_RISK_PUMP` | Flag for manual review |
| `STRONG_BUY` | Auto-execute with user-defined size |

---

## 🔜 **What Comes After This?**

Once you have this **Alpha Pipeline**, you can layer on:

1. **AI Agents** (v2.2): Use Qwen/Gemini to *interpret* why a token is trending  
2. **Community Feeds**: Let creators *comment* on high-score tokens  
3. **$SIGIL Subscriptions**: Offer “early alpha alerts” to paying users  
4. **Copy Trading**: Let users mirror trades of top-performing agents

---

## ✅ Summary: The Full Pre-Trade Checklist

| Stage | Check | Tool/Method |
|------|------|-------------|
| **1. Safety** | Mint revoked? | `@solana/spl-token` |
| | Standard program? | ATA owner check |
| | Honeypot? | Simulate sell |
| **2. Fundamentals** | Liquidity ≥10 SOL? | Raydium API |
| | Decentralized holders? | `getTokenLargestAccounts` |
| | Growing community? | Helius + Twitter API |
| **3. Timing** | Stable price action? | Real-time OHLCV |
| | Dev is credible? | Wallet labeling |

> 🔑 **Key Insight**: The best meme coin bots don’t just avoid rugs — they **find the next viral token before it moons**.

---

Ready for a **TypeScript implementation** of this full pipeline? Or a flowchart for your dev team?
