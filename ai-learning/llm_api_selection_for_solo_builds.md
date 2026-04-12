# Choosing an LLM API for a Solo Build — A PM's Practical Guide

*Part of my AI Learning Journey | Last updated: April 2026*

---

## The Problem This Solves

Most LLM API guides are written for engineering teams with cloud budgets. They recommend paid tiers, assume you have DevOps support, and treat cost as a variable to optimise rather than a hard constraint.

Solo builders face a different set of constraints:

- **Cost ceiling is often ₹0** — this is a portfolio project or a personal tool, not a funded product
- **No DevOps** — whatever you choose has to deploy with one command and stay up without maintenance
- **Public URL is required** — the whole point is a shareable demo, not something that only runs on your laptop
- **Time is limited** — you have evenings and weekends, not sprints

This guide maps the decision space for choosing an LLM API under these constraints, based on real evaluation done while building an ad copy evaluation tool in April 2026.

---

## The Four Categories of Option

### Category 1 — Local LLM (Ollama, LM Studio)

**How it works:** Install a tool like Ollama on your personal machine. It downloads an open-source model (Llama 4, Qwen3, Gemma3, etc.) and runs a local API server — typically at `localhost:11434`. Your app calls this instead of a cloud API. Zero per-token cost.

**The tools:**
- **Ollama** — CLI-first, developer-focused, OpenAI-compatible API, strongest model library (100+ models). Best for building apps and pipelines.
- **LM Studio** — GUI-first, easier for non-terminal users, good for model evaluation and testing. Less suitable for programmatic integration.

**Hardware requirements:** The key number is VRAM. Plan for roughly 0.6–0.7 GB per billion parameters at Q4 quantization. A capable 8B model (Llama 4 8B, Gemma3 8B) needs ~6 GB VRAM and runs reasonably on a modern laptop GPU or Apple Silicon.

**The critical constraint for solo builders:**

If your app is hosted on a cloud platform (Streamlit Community Cloud, Hugging Face Spaces, Render), that server cannot reach your local Ollama instance. The cloud server and your laptop are on different networks.

Workarounds exist but all have problems:

| Workaround | Problem |
|------------|---------|
| ngrok tunnel | URL changes on every restart — not a persistent shareable link |
| Host Ollama on a cloud server | Adds monthly server cost + infrastructure complexity |
| Run app locally only | Eliminates the public URL — kills the demo purpose |

**When local LLM is the right choice:**
- Development and testing — use local LLM to iterate on prompts without burning API credits
- Apps that are intentionally private — internal tools, personal tools that don't need a public URL
- Cost is genuinely zero and you don't need public deployment

**When local LLM is the wrong choice:**
- Any app that needs a public, persistent URL
- Batch evaluation or scheduled jobs that run while your laptop is closed

---

### Category 2 — Free Cloud API Tiers

Several major providers offer genuinely free API access in 2026, no credit card required. The landscape as of April 2026:

| Provider | Free model | Req/day | Req/min | Credit card needed |
|----------|-----------|---------|---------|-------------------|
| Google Gemini (Flash-Lite) | Gemini 2.5 Flash-Lite | 1,000 | 15 | No |
| Google Gemini (Flash) | Gemini 2.5 Flash | 250 | 10 | No |
| Google Gemini (Pro) | Gemini 2.5 Pro | 100 | 5 | No |
| Groq | Llama 4, Mixtral | ~14,400 | varies | No |
| OpenAI | GPT-4o Mini | ~limited | low | Yes (with expiring credits) |
| Anthropic (Claude) | — | — | — | Yes (with expiring $5 credits) |

**Key observations:**

Google is the most practical free-tier provider for solo builders. No credit card, generous daily limits for low-volume apps, and the API is OpenAI-compatible — meaning the code change from any other provider is minimal (swap base URL and key).

Groq offers very high request volumes and exceptional speed (sub-100ms inference), but serves open-source models. For tasks requiring nuanced semantic reasoning — intent alignment, differentiation scoring, structured rubric output — open-source models at 7B–13B parameters may be less consistent than Gemini or Claude.

**The free tier risk:** Google reduced free quotas by 50–80% in December 2025. Free tiers are not permanent entitlements — they can change. Build with the assumption that you may need to move to a paid tier if the project scales, and document the paid-tier fallback cost in your risk register.

---

### Category 3 — Cheap Paid Cloud APIs (No Free Tier)

If free tier limits are insufficient, the cheapest paid options as of April 2026:

| Provider + Model | Input cost | Output cost | Notes |
|-----------------|------------|-------------|-------|
| Gemini 2.5 Flash-Lite (paid) | $0.10/M | $0.40/M | Cheapest major provider |
| Gemini 2.5 Flash (paid) | $0.30/M | $2.50/M | Better quality than Lite |
| Claude Haiku 4.5 | $0.80/M | $4.00/M | Cheapest Claude model |
| Claude Sonnet 4.6 | $3.00/M | $15.00/M | Mid-tier Claude |
| DeepSeek V3 | $0.28/M | $0.50/M | Strong open-weights model |

At the volume of a personal portfolio project (50–100 evaluations/day), the real monthly cost difference between these options is under ₹200. The decision is rarely about cost at this scale — it's about quality consistency and how the API fits your stack.

---

### Category 4 — Rule-Based Scoring (No LLM)

For tasks where evaluation criteria are entirely mechanical — character counts, keyword presence, format compliance — a rule-based approach costs nothing and requires no API.

**What rule-based scoring can do:**
- Check character limits (headline ≤ 30 chars, description ≤ 90 chars)
- Detect presence or absence of a CTA keyword
- Flag placeholder text or empty fields

**What rule-based scoring cannot do:**
- Assess whether a CTA matches the searcher's intent
- Evaluate whether a headline communicates a specific differentiator
- Reason about tone, specificity, or persuasiveness

For any evaluation task that requires semantic understanding — which is most of them — rule-based scoring is insufficient as a standalone approach. It can be used as a pre-filter (reject inputs that fail character limits before sending to the LLM) but not as the core evaluation engine.

---

## The Decision Framework

Answer these questions in order:

**1. Does the app need a public URL?**
- Yes → eliminate local LLM for production
- No → local LLM is viable; evaluate based on hardware and model quality

**2. Is cost a hard zero constraint?**
- Yes → evaluate free tiers (Google Gemini is the strongest option)
- No → evaluate quality and developer experience first, then cost

**3. What does the evaluation task actually require?**
- Mechanical checks only → rule-based (no LLM needed)
- Semantic reasoning → LLM required; question is which one

**4. How many requests per day do you actually need?**
- Under 250/day → Gemini Flash free tier is sufficient
- 250–1,000/day → Gemini Flash-Lite free tier covers this
- Over 1,000/day → you're beyond a solo portfolio project; evaluate paid tiers

**5. Is API consolidation a goal?**
- If you're building multiple projects and want one API key across all of them, pick the free-tier provider first and only deviate when quality or limits demand it.

---

## How to Test Quality Before Committing

Before locking in an API choice, run a quality spot-check:

1. Write 3 test prompts that represent your evaluation task
2. Include one clearly good input, one borderline input, one clearly bad input
3. Run each through your candidate API
4. Check: does the model's verdict match what a human expert would say?
5. Check: is the output consistently structured (JSON, specific fields, etc.)?
6. Run the same prompts 3 times — does the output vary significantly?

If the model fails on step 4, 5, or 6, move to a higher-capability model before building further. Prompt engineering can improve results but cannot fix a fundamental capability gap.

---

## Practical Setup Notes

### Google Gemini API (recommended for zero-cost solo builds)

1. Go to [ai.google.dev](https://ai.google.dev)
2. Sign in with a personal Google account
3. Click "Get API key" — no credit card required
4. Set the key as an environment variable: `GOOGLE_API_KEY=your_key`

**Python SDK:**
```bash
pip install google-generativeai
```

**Basic call:**
```python
import google.generativeai as genai
genai.configure(api_key=os.environ["GOOGLE_API_KEY"])
model = genai.GenerativeModel("gemini-2.5-flash-lite")
response = model.generate_content("Your prompt here")
print(response.text)
```

**OpenAI-compatible endpoint** (useful if you want to swap providers easily):
```python
from openai import OpenAI
client = OpenAI(
    api_key=os.environ["GOOGLE_API_KEY"],
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/"
)
```

### Streamlit secrets management (for deployed apps)

Store the key in `.streamlit/secrets.toml` locally (excluded from git):
```toml
GOOGLE_API_KEY = "your_key_here"
```

In code:
```python
api_key = st.secrets["GOOGLE_API_KEY"]
```

In Streamlit Community Cloud: add the key via the app's Secrets settings in the dashboard.

---

## The Consolidation Strategy

If you're building multiple AI projects and want to minimise API overhead:

1. **Start with Gemini free tier** across all projects — one account, one key, zero cost
2. **Add a paid tier only when a specific project demands it** — don't pre-empt this
3. **Use the OpenAI-compatible endpoint format** where possible — makes switching providers a one-line change
4. **Document your fallback** in each project's risk register — if the free tier tightens, what's the paid-tier cost?

The goal is not to optimise every project independently. It's to have a stable, low-friction API layer that lets you focus on building, not on managing credentials and billing across five different providers.

---

## Further Reading

- [Google AI Studio](https://aistudio.google.com) — API key setup, free tier
- [Gemini API pricing](https://ai.google.dev/pricing) — current free tier limits
- [Ollama](https://ollama.com) — local model runner
- [LM Studio](https://lmstudio.ai) — GUI-based local model runner
- [Groq](https://groq.com) — free tier, high-speed open-source model inference
- [OpenRouter](https://openrouter.ai) — unified API for multiple providers, some free models

---

*Written as part of my public AI learning journey. I am a Senior TPM and Designated PM at Microsoft AI, building real AI products and documenting what I learn. See the `llm-eval-toolkit` repo for a hands-on project applying these concepts.*
