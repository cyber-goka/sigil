

# **Comprehensive Technical Specification: Sigil — Solana Meme Coin Trading & Community Platform with AI Agents & Native Token Economy**  
**Version 1.1**  
*For Product Team*

---

## **1. Executive Summary**

**Sigil** is an intelligent, community-driven platform for Solana meme coin trading that unifies:
- **Real-time on-chain monitoring** of new tokens (pump.fun → Raydium)
- **Hybrid trading**: manual execution + autonomous AI agents
- **Creator monetization**: influencers, devs, and news groups offer tiered content
- **Native utility token**: **`$SIGIL`** powers all subscriptions and rewards
- **AI co-pilots**: powered by **Qwen, OpenAI, and Gemini APIs** for signal synthesis, risk analysis, and trade rationale

By using **production-grade cloud LLMs**, Sigil avoids the cost and complexity of training/maintaining custom models while delivering high-quality, explainable AI insights.

---

## **2. System Architecture (Updated)**

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
└────────┬─────────┘        └──────────┬──────────┘       └─────────────────────┘
         │                             │
         └──────────────┬──────────────┘
                        │
               ┌────────▼─────────┐     ┌──────────────────────┐
               │  Token Service   │<--->│   AI Agent Service   │
               │ - $SIGIL Economy │     │ - Signal Generation  │
               └────────┬─────────┘     │ - LLM Orchestration  │
                        │               │ - Social + On-Chain  │
                        ▼               └──────────┬───────────┘
               ┌────────▼─────────┐                │
               │  Data Layer      │<───────────────┼───[ Qwen / OpenAI / Gemini ]
               │ - PostgreSQL     │                │
               │ - Redis          │                │
               │ - Vector DB      │                │
               │ - S3 (Media)     │                │
               └────────┬─────────┘                │
                        │                          │
               ┌────────▼─────────┐               │
               │  Blockchain      │<──────────────┘
               │ - Solana RPC     │
               │ - Helius         │
               │ - Jupiter API    │
               │ - $SIGIL Mint    │
               └──────────────────┘
```

---

## **3. AI Agent Service (Revised)**

### **3.1. Philosophy**
- **No fine-tuned models**. Use **best-in-class cloud LLMs** via API.
- **Multi-provider fallback**: If OpenAI is down, use Qwen or Gemini.
- **Cost-aware routing**: Use cheaper models (e.g., Qwen-Max) for simple tasks, GPT-4o/Gemini-1.5 for complex reasoning.

### **3.2. Supported LLM Providers**
| Provider   | Model Used          | Use Case                          | Why |
|-----------|---------------------|-----------------------------------|-----|
| **OpenAI** | `gpt-4o`            | High-stakes trade rationale, risk analysis | Best reasoning, low latency |
| **Google** | `gemini-1.5-flash`  | Social sentiment summarization    | Fast, cheap, strong at text |
| **Alibaba**| `qwen-max`          | General signal synthesis, fallback | Cost-effective, good Chinese/English support |

> ✅ All providers support **structured JSON output** → easy parsing.

### **3.3. Agent Workflow**
1. **Data Ingestion**:
   - On-chain: New token mint, LP add, whale buy (via Helius)
   - Social: X posts, Telegram public channels (via official APIs or RSS)
2. **Event Enrichment**:
   - Tag with metadata (e.g., “dog-themed”, “dev wallet locked”)
   - Store embeddings in **Qdrant** for similarity search
3. **LLM Prompting**:
   ```text
   You are SigilAI, a Solana meme coin analyst.
   Analyze this event and decide if it's a high-confidence buy signal.

   Token: $WOOF
   - 50 new holders in last 2 min
   - LP locked for 30 days
   - Dev wallet burned
   - X mentions: +200% in 1 hour
   - No honeypot flags

   Respond in JSON: { "action": "BUY"|"HOLD"|"AVOID", "confidence": 0.0-1.0, "rationale": "..." }
   ```
4. **Execution**:
   - If confidence > user threshold → send to Trading Engine (with approval if not auto-mode)

### **3.4. Tech Stack**
- **Orchestrator**: Python + FastAPI
- **LLM Router**: LangChain or LiteLLM (for multi-provider support)
- **Caching**: Redis (cache LLM responses for identical events)
- **Cost Control**: 
  - Max tokens per call
  - Daily budget per user
  - Fallback to cheaper model if GPT-4o quota exceeded

### **3.5. Key Endpoints**
```ts
GET /ai/feed
// Returns AI-generated signals with rationale from LLM

POST /ai/analyze
{
  tokenMint: "EKp...xyz",
  context: { holders: 50, lpLocked: true, ... }
}
// Returns LLM analysis in structured JSON
```

### **3.6. Safety & Explainability**
- **All outputs are JSON-enforced** → no hallucinated actions
- **Rationale always included** → user can audit AI logic
- **No autonomous execution without explicit opt-in**
- **Provider API keys stored in AWS Secrets Manager / HashiCorp Vault**

---

## **4. Updated Infrastructure**

| Component          | Technology                          |
|--------------------|-------------------------------------|
| **AI Runtime**     | Python 3.11 + FastAPI + LiteLLM     |
| **LLM Providers**  | OpenAI, Google AI Studio, Alibaba Cloud |
| **Vector DB**      | Qdrant (for event similarity)       |
| **Secrets**        | AWS Secrets Manager                 |
| **Monitoring**     | Log LLM latency, cost, error rate per provider |

---

## **5. Cost & Performance Considerations**

| Task                     | Recommended Model     | Avg Cost / Call |
|--------------------------|-----------------------|-----------------|
| Social sentiment summary | `gemini-1.5-flash`    | $0.0001         |
| Trade rationale          | `gpt-4o`              | $0.005          |
| Fallback analysis        | `qwen-max`            | $0.002          |

> 💡 **Optimization**: Cache identical prompts (e.g., same token event) for 5 minutes → reduce cost by ~40%.

---

## **6. Security & Compliance**

- **No user data sent to LLMs** without anonymization
- **Social data**: Only public posts; no private/user-specific data in prompts
- **Audit logs**: All LLM calls logged with user ID, token, and provider
- **GDPR/CCPA**: LLM providers must be compliant (OpenAI/Gemini/Qwen all offer enterprise agreements)

---

## **7. Appendix**

### **A. Example LLM Prompt (OpenAI)**
```python
response = openai.chat.completions.create(
  model="gpt-4o",
  response_format={ "type": "json_object" },
  messages=[
    {"role": "system", "content": "You are SigilAI..."},
    {"role": "user", "content": "Token: $MOON\n- 100 new holders\n- LP: 50 SOL\n- Dev wallet: renounced..."}
  ]
)
```

### **B. Multi-Provider Fallback Logic**
```python
def get_llm_analysis(context):
    try:
        return openai_analyze(context)
    except RateLimitError:
        try:
            return gemini_analyze(context)
        except:
            return qwen_analyze(context)  # Final fallback
```

---

**Document Owner**: Sigil Team  
**Last Updated**: Oct 2025  
