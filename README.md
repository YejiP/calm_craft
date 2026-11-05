# CalmCraft

Ever typed a text in anger or anxiety, hit send, and immediately wished you could take it back?

CalmCraft is a real-time 1-on-1 chat app that acts as an **emotional shock absorber**. When you send a message driven by fear or frustration, it intercepts the text, uses Cognitive Behavioral Therapy (CBT) principles to strip out cognitive distortions (like mind reading or catastrophic thinking), and offers constructive alternatives before the other person sees it.

I built this project to explore **distributed systems engineering**—tackling real-time constraints, background job queues, and caching beyond standard CRUD—while practicing **AI-assisted software engineering** to accelerate architecture design and system implementation.

---

### The Technical Challenge

- **Two Distinct Latency Budgets:**
  - **Direct Chat (CBT OFF):** Requires real-time delivery in **<50ms**.
  - **CBT-Assisted Chat (CBT ON):** LLM cognitive restructuring and alternative suggestion generation take **1.5s–3.0s**.
- **The System Conflict:**
  - Synchronously holding WebSocket worker threads during a 3-second LLM inference under load exhausts socket connections and blocks routine chat.
  - Every message should not blindly hit expensive AI models—everyday banter must bypass the pipeline entirely, while emotionally charged messages require safe, resilient asynchronous decoupling.

### Architecture

CalmCraft uses **Client-Driven Selective Routing** to separate lightweight instant traffic from heavy background inference, combining Redis Streams with multi-tier caching.

Plaintext

```
[Sender Client]
      │
      ├── (CBT Mode: OFF) ───────── [Direct Pass <30ms] ─────────────▶ [Recipient Client]
      │
      └── (CBT Mode: ON)  ──▶ [WebSocket Gateway]
                                    │
                                    ├── (Instant ACK <15ms) ─────────▶ [Sender UI: "Refining..."]
                                    │
                                    ├──▶ [L1 Exact Cache (SHA256)] ─────────┐ (Hit: ~5ms)
                                    │                                       │
                                    ├──▶ [L2 Semantic Cache (Vector)] ──────┤ (Hit: ~25ms)
                                    │                                       │
                                    ▼ (Miss: XADD Task)                     │
                           [Redis Streams Queue]                            │
                                    │                                       │
                                    ▼ (XREADGROUP)                          │
                             [Worker Pool]                                  │
                                    │                                       │
                                    ├──▶ [Circuit Breaker] ──▶ [LLM API]   │
                                    │                                       │
                                    └───────────────────────────────────────┘
                                                │
                                                ▼ (Push Suggestions)
                                         [Sender Client]
                                                │
                                                ▼ (User Selects Option)
                                         [Recipient Client]
```

#### How the Pipeline Operates:

1. **Selective Routing (`cbt_mode: false`):** Routine messages skip AI queues entirely, streaming straight through the WebSocket broker to the recipient in under 30ms.
2. **Asynchronous Buffer (`cbt_mode: true`):** The server enqueues the raw text into Redis Streams and hands the sender a 15ms `ACK` receipt. The sender enters a natural cool-down state while the socket connection remains non-blocking.
3. **Multi-Tier Cache Bypass:** Checks L1 exact hash and L2 Redis vector embeddings before invoking downstream LLMs, eliminating redundant API costs for common anxious expressions.
4. **Worker Decoupling & Fallback:** Background consumers process restructuring tasks at their own pace, protected by a circuit breaker that fails safe to regex-based de-escalation filters if external APIs stall.

