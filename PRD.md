# Autonomous  Website Generation and Management System (AGMS)

# Product Requirement Document (PRD): Autonomous Website Generation & Management System

**Role:** Technical Product Manager (Portfolio Exercise)

**Status:** Strategic Proposal / Feasibility Study

---

## 1. Executive Summary

This document outlines the product strategy for an **Autonomous Website Generation and Management System (AGMS)**—a system of specialized AI agents capable of generating, hosting, SEO-optimising, and continuously updating 180+ websites from a single dashboard. This system is designed to function as a **Headless CMS + Autonomous Workforce**, purpose-built for a medical web agency currently managing approximately 180 client websites that wishes to automate its end-to-end web development and maintenance pipeline.

This report evaluates three distinct implementation architectures, each with its own approximate cost and break-even analysis:

1. **The Sovereign Stack:** Fully on-premise, zero third-party API dependency (Maximum control, maximum hardware cost).
2. **The Hybrid Edge Stack:** Local compute for core code generation, cloud APIs for research and content (Balanced cost/performance).
3. **The Orchestration Layer:** Wrapping existing SaaS tools like Duda via API to simulate autonomy (Fastest time-to-market).

> **Context:** The client currently handles ~180 medical websites manually. The system does not need to acquire new clients to break even — it needs only to reduce or eliminate the cost of the human labour currently servicing those 180 sites. This fundamentally improves break-even projections across all three paths.
> 

---

## 2. Product Architecture & Workflow

### 2.1 Core Agentic Pipeline

The system treats a website as a **Living Data Structure**. The following workflow is the core specification:

| Phase | Agent/Action | Function | Key Technology |
| --- | --- | --- | --- |
| **Initiation** | **Task-Master** | Breaks user prompt into UI/UX, Backend Schema, and Market Research tasks. | DeepSeek-R1-Distill-Qwen-7B (Local) |
| **Orchestration** | **CrewAI** | Manages state, data hand-off, and task execution flow between agents. | CrewAI Flows & Crews |
| **Research** | AutoResearchClaw | Performs deep literature/web search on niche. Returns cited report. | Perplexity Sonar Pro API |
| **Architecture** | Dev Agent (Qwen-Coder) | Builds site skeleton (HTML/CSS/JS) or Headless CMS schema. | Qwen2.5-Coder-32B |
| **Population** | Generator Model | Fills skeleton with text/images based on research report. | Flux Schnell (Image) / DeepSeek-V3 (Text) |
| **Monitoring** | **Observer Model** | Non-intrusive layer that monitors agent actions, prevents loops, and logs errors. | Custom Middleware / Langfuse |
| **Maintenance** | Trend Monitor | Pulls analytics, detects trend shifts, triggers **Auto-Update** pipeline. | Custom Analytics Router |
| **Memory** | Context Store | Context retention across 180+ siloed projects without vector DB bloat. | Hierarchical Key-Value Store (custom-built) |

> **Note on model naming:** All models referenced in this document are verified, publicly available at the time of writing. "DeepSeek-V3" refers to the model released December 2024 (deepseek-chat via the DeepSeek API).
> 

---

### 2.2 Task-Master: DeepSeek-R1-Distill-Qwen-7B (via Ollama)

The Task-Master role requires a model capable of structured decomposition of high-level prompts into JSON task graphs for downstream agents.

- **Model Choice:** `DeepSeek-R1-Distill-Qwen-7B` (GGUF Q4_K_M quantisation).
- **Justification:** This model delivers strong Chain-of-Thought (CoT) reasoning in a compact 7B footprint. It excels at STEM reasoning, code planning, and structured output generation, and runs comfortably on a single RTX 6000 Ada GPU.
- **Important caveat:** DeepSeek-R1-Distill-Qwen-7B is optimised for mathematical, logical, and STEM chain-of-thought tasks. For general language decomposition tasks (e.g., interpreting a brief like "build a cardiology clinic website"), the model should be paired with a system prompt that forces structured JSON output. For more complex natural language instructions, consider falling back to DeepSeek-V3 via API for the Task-Master role.
- **Integration:** The model is served locally via `Ollama` and called by CrewAI's `LLM` class.

Additional tasks come in when a website has to be updated. In order to prevent the SEO tanking upon regeneration, the Task-Master will keep track of the following:

- Map existing URLs.
- Maintain 301 redirects.
- Preserve existing high-performing metadata.

---

### 2.3 CrewAI Data Pipeline & State Management

CrewAI manages the data flow between agents, ensuring that the output of one agent becomes the input for the next. This is handled through a combination of **Crews** and **Flows**.

- **State Sharing:** CrewAI uses a flexible state system (Pydantic models or dictionaries) to pass data between tasks. When the `Task-Master` creates a plan, it is stored in the crew's `state`. The `Research Agent` reads the research topic from the state and writes back its findings.
- **Pipeline Example:**
    1. `Task-Master` (DeepSeek-R1-Distill-Qwen-7B) produces a `ResearchPlan`.
    2. CrewAI invokes `AutoResearchClaw` (Crew 1) and `DevAgent` (Crew 2) in parallel.
    3. Output from `AutoResearchClaw` is validated and passed via `state` to the `ContentGenerator`.
    4. Final output from `ContentGenerator` is passed to `DevAgent` for injection.
- **Reliability:** CrewAI's built-in guardrails — iteration limits and explicit task specifications — are essential to prevent agents from going off-rails.

---

### 2.4 Observer Model: Preventing Agentic Loops

To ensure the system is production-ready, an Observer Model is introduced as a non-intrusive middleware layer. This addresses the risk of AI agents entering infinite loops or using tools incorrectly.

- **Implementation:** The Observer is a Python class that wraps the execution of agent tools and LLM calls.
- **Loop Prevention:** It tracks the **hash of the agent's state**. If the state repeats more than 3 times without progress, the Observer injects a `"STOP"` command, logs the error, and triggers a fallback strategy (e.g., asking the operator for clarification).
- **Monitoring Dashboard:** The Observer feeds data into **Langfuse** (open-source) to track:
    - Token usage and cost per site update.
    - Agent step success/failure rate.
    - Context window saturation events (to prevent the model from losing its instructions mid-task).

---

### 2.5 Context Store: Multi-Project Memory

Managing 180 isolated client websites requires a memory architecture that doesn't cross-contaminate context between projects and doesn't incur vector database overhead at scale.

- **Approach:** A hierarchical key-value store, implemented in-house, that maps each site ID to a structured JSON state object (site type, last updated, content schema, last research report summary, brand guidelines).
- **This is a custom-built component** and must be scoped and engineered in Q3 of the development roadmap. It is not an off-the-shelf product.
- **Alternative:** For a faster initial implementation, a Redis-backed store with site-scoped namespacing is a viable substitute.

### 2.6 Human-In-Loop

Regardless of the state of the art technologies being used, it is important to have a human in loop always for approvals and to verify for hallucinations that might escape the system. While the idea of complete automation looks lucrative, if not monitored it will create very real legal risks very quickly.

### 2.7 Mitigation Risk

Moving approximately 180 existing sites to a new autonomous stack isn't just a technical challenge, it’s an uptime and SEO risk for the clinics. This is a risk that will have to be factored in when planning the implementation.

### 2.8 Medical Compliance Risk

Whenever a third party model is involved via APIs, mostly Path 2 and 3, any point that comes close to patient data, however trivial, has to be covered by Business Associate Agreements (BAA) or all Personally Identifiable Information (PII) has to be scrubbed before the data goes anywhere via API.

---

## 3. Implementation Paths & Financial Analysis (India Focus)

*All costs are in Indian Rupees (₹). Assumptions: 5-developer team, Indian market costs, exchange rate ₹84 = $1 USD. The client already manages ~180 medical websites, so revenue calculations are based on cost-savings from automation, not new client acquisition.*

---

### Path 1: The Fully Independent Sovereign Stack

*Maximum control. Zero third-party API reliance for core tasks. All models run on-premise.*

### 3.1 Hardware & Infrastructure Cost (One-Time CapEx)

> **Pricing note:** All GPU prices reflect current Indian retail market rates as of the time of writing. The RTX 6000 Ada retails between ₹6,94,499–₹9,40,999 depending on vendor. The L40S retails between ₹12,00,000–₹15,00,000 per card. Figures below use mid-range estimates.
> 

| Component | **PoC (Single GPU)** | **Production (180 Sites)** |
| --- | --- | --- |
| **GPU** | 1× NVIDIA RTX 6000 Ada (48GB) | 4× NVIDIA L40S (48GB each) |
| **GPU Cost (INR)** | ~₹8,00,000 (mid-range retail) | ~₹13,50,000/card (Total: **~₹54,00,000**) |
| **Server Chassis / CPU / RAM** | ₹2,00,000 | ₹8,00,000 |
| **Total Hardware (Approx.)** | **~₹10,00,000** | **~₹62,00,000** |

> **Note:** The prices are an average estimate and actual prices will inevitably vary.
> 

### 3.2 Operational Cost (OpEx — Annual)

| Resource | Cost (Annual) |
| --- | --- |
| **Power & Cooling (24/7)** | ₹1,50,000 (PoC) / ₹6,00,000 (Prod) |
| **Team (5 Devs @ ₹20 LPA avg)** | ₹1,00,00,000 |
| **Maintenance / Spares** | ₹50,000 |
| **Total Annual OpEx (Est.)** | **~₹1,07,00,000** |

### 3.3 Break-Even Analysis (Sovereign)

The 180 websites are **existing clients**. The system does not need to generate new revenue — it eliminates the manual labour cost currently required to serve them.

- **Current manual cost assumption:** 1 developer manages ~15–20 medical sites at ₹20 LPA. 180 sites ≈ 9–12 developers currently.
- **Labour savings from automation:** Replacing 9 developers at ₹20 LPA = **₹1,80,00,000/year saved.**
- **Year 1 Total Cost:** CapEx (~₹62L production) + OpEx (~₹1.07 Cr) = **~₹1.69 Cr**
- **Annual savings from Year 2 onwards:** ₹1.80 Cr (labour) − ₹1.07 Cr (new OpEx) = **~₹73L net annual saving.**
- **Break-Even:** ~18–24 months (front-loaded by CapEx). Faster if partial automation allows redeployment of existing developers rather than outright replacement.

---

### Path 2: The Hybrid Edge Stack *(Recommended)*

*Local compute for sensitive code generation. Cloud APIs for research and heavy content generation.*

### 3.4 Monthly Operational Cost (OpEx)

| Resource | Specification | Monthly Cost (₹) |
| --- | --- | --- |
| **Cloud GPU (RTX 6000 Ada equiv.)** | RunPod / AceCloud (on-demand, ~$0.77/hr × 730 hrs) | ₹47,000–₹53,000 |
| **Research API** | Perplexity Sonar Pro (token-based: ~$3/M input, $15/M output + $5/1k requests) | ₹20,000–₹30,000* |
| **Image API** | Flux Schnell via [fal.ai](http://fal.ai/) (~$0.003/image × 10,000 images) | ₹2,500 |
| **LLM API** | DeepSeek-V3 (~$0.14/M input, $0.28/M output × ~10M tokens) | ₹350–₹700 |
| **Hosting (Vercel / Cloudflare)** | 180 Sites | ₹2,500–₹3,500 |
| **Team (5 Devs)** | Salaries | ₹8,33,000 |
| **Total Monthly Burn** | **(Variable)** | **~₹9,10,000–₹9,25,000** |

> **\Perplexity Sonar Pro pricing correction:* The API is billed on tokens + per-request fees, not as a flat call-volume plan. At $5 per 1,000 requests, 50,000 research calls alone cost ~$250 (~₹21,000) in request fees, before any token costs are added. Budget ₹20,000–₹30,000/month for moderate usage (10,000–20,000 substantive research calls with average-length inputs/outputs). If the system performs one deep research per site update cycle per month across 180 sites, that is ~180 calls — which is relatively inexpensive. Scale projections carefully against actual call volume.
> 

> **DeepSeek-V3 note:** The official API (deepseek-chat) is priced at $0.14/M input tokens and $0.28/M output tokens (cache-miss rates). At 10M tokens/month, total API cost is under $5 (~₹420). This is extremely cost-efficient for content generation.
> 

### 3.5 Break-Even Analysis (Hybrid)

- **Annual OpEx:** ~₹1.10 Cr (including 5-developer team).
- **Labour savings from automation (same as Path 1):** ₹1.80 Cr/year saved by replacing 9 manual developers.
- **Net Annual Saving:** ₹1.80 Cr − ₹1.10 Cr = **~₹70L/year net saving.**
- **CapEx:** Near-zero (cloud GPU, no hardware purchase).
- **Break-Even: 1–2 months** from go-live. This is the most financially efficient path for an existing agency.
- **Additional upside:** As the system scales beyond 180 sites (new client acquisition), per-site marginal cost is minimal — each additional site costs only incremental API tokens and hosting.

---

### Path 3: Orchestration Layer (Wrapper Over Existing SaaS)

*Fastest time-to-market, lowest cost, highest fragility.*

### 3.6 Monthly Operational Cost

| Resource | Specification | Monthly Cost (₹) |
| --- | --- | --- |
| **SaaS Subscription** | Duda Agency Plan ($59/mo, verified) | ₹5,000 |
| **Orchestration Agent** | 1× DigitalOcean Droplet | ₹1,500 |
| **API Costs (DeepSeek-V3)** | Minimal orchestration-only usage | ₹500–₹1,000 |
| **Total Monthly Burn** |  | **~₹7,500–₹8,000** |

> **Duda pricing note:** The $59/month Agency plan is verified correct for monthly billing. The annual plan reduces this to $44/month. At 180 sites, Duda charges ₹17/month per additional site beyond the 4 included — a production deployment would require purchasing ~176 additional site subscriptions at $17/site/month, adding ~$2,992/month (~₹2,51,000/month). This makes the "Wrapper" path significantly more expensive at production scale than the summary line item suggests. The ₹7,500 figure reflects the account plan only, not the full site subscription cost.
> 

### 3.7 Break-Even Analysis (Wrapper — Corrected for Scale)

- **True monthly cost at 180 sites:** ₹5,000 (account) + ₹2,51,000 (site subs) + ₹1,500 (infra) + ₹1,000 (APIs) = **~₹2,58,500/month (~₹31L/year)**
- **Labour savings:** Same ₹1.80 Cr/year.
- **Net Annual Saving:** ₹1.80 Cr − ₹31L = **~₹1.49 Cr/year net saving.**
- **Break-Even: Immediate** (Month 1 cash-flow positive).
- **Risk:** **Extreme Vendor Lock-In & Fragility.** If Duda changes their API or UI, the entire automation pipeline breaks. Not viable as a scalable or white-labelled SaaS product. Viable only as an internal operations tool for the existing 180-site portfolio.

---

## 4. Development Timeline & Milestones

Given the complexity of agentic workflows and the critical need for an Observer layer in a medical web context (where incorrect content could have compliance implications), a **12-Month Roadmap** is required for a stable Beta.

| Quarter | Focus Area | Key Deliverable |
| --- | --- | --- |
| **Q1** | **PoC & Observer** | Task-Master (DeepSeek-R1-Distill) → Research → Static Site generation. **Critical:** Implement Loop Detection Observer. Medical content compliance flags. |
| **Q2** | **CrewAI Integration** | Formalise state pipeline between Research and Dev agents. Multi-tenant Headless CMS setup for 180 sites. |
| **Q3** | **Context Store & Analytics** | Implement custom hierarchical Context Store. Analytics parser and Auto-Update trigger for trend-shifted content. |
| **Q4** | **Scale & Hardening** | Load testing 180 concurrent update jobs. Semaphore locking for database writes. HIPAA/medical compliance review of AI-generated content pipeline. |

> **Medical-specific consideration:** Because the existing portfolio is medical websites, any AI-generated content (symptoms, treatments, clinic descriptions) must pass a human review gate or a domain-specific compliance check before publishing. This should be scoped into Q2 and Q4 deliverables and adds both development time and an ongoing operational review cost not reflected in the base OpEx figures above.
> 

---

## 5. Corrected Model & Technology Reference

The following table provides a verified snapshot of all AI models referenced in this PRD as of the document date.

| Role | Model | Status | Notes |
| --- | --- | --- | --- |
| Task-Master / Reasoning | DeepSeek-R1-Distill-Qwen-7B | ✅ Available (open-source, MIT) | Best for structured, STEM-style decomposition |
| Code Generation | Qwen2.5-Coder-32B | ✅ Available (Apache 2.0) | SOTA open-source code LLM, GPT-4o competitive |
| Text / Content Generation | DeepSeek-V3 | ✅ Available (API + open weights) | $0.14/M input, $0.28/M output. Latest version: deepseek-chat |
| Image Generation | Flux Schnell (FLUX.1 [schnell]) | ✅ Available (Apache 2.0) | ~$0.003/image via [fal.ai/Replicate](http://fal.ai/Replicate) |
| Research / Web Search | Perplexity Sonar Pro | ✅ Available (paid API) | Token-based pricing; $3/M input, $15/M output + per-request fee |
| Observability | Langfuse | ✅ Available (open-source) | Self-hostable |
| Agent Orchestration | CrewAI | ✅ Available (open-source) | Stable for multi-agent workflows |

---

## 6. Conclusion & Recommendation

For a medical web agency **already managing 180 client websites**, the business case for AGMS is strong regardless of path chosen, because the savings come from replacing existing labour — not from speculative new revenue.

**Recommended path for immediate deployment:** **Path 3 (SaaS Wrapper)** as a low-risk, zero-CapEx proof of concept for internal operations. Note the corrected full-scale Duda subscription cost (~₹2.58L/month at 180 sites) must be budgeted against the ₹15L/month in labour savings.

**Recommended path for strategic, long-term product:** **Path 2 (Hybrid Edge Stack)** — near-zero CapEx, fastest break-even (1–2 months), and the architecture most amenable to expanding beyond the existing 180-site portfolio into a sellable SaaS product for other medical web agencies.

**Path 1 (Sovereign Stack)** is recommended only if the agency has strong data-sovereignty requirements for patient-adjacent content, or if it intends to white-label the system and resell it to other agencies at scale (at which point the CapEx amortises across a much larger revenue base).

---

## 7. Success Metrics (KPIs)

- **Primary Metric:** **Reduction in Man-Hours per Site Update.** (Target: 85% reduction from baseline).
- **Technical Metric:** **Agent Autonomy Rate.** (% of tasks completed without the Observer model triggering a "Stop" or requiring human intervention).
- **Business Metric:** **SEO Rank Retention.** (95% of sites must maintain Page 1 rankings for their primary keywords 90 days post-migration).
- **Safety Metric:** **Zero Non-Compliant Content Incidents.** (Verified through the Q4 Human-in-the-loop review audit).
