# Choosing an LLM API for a Solo Build — A PM's Practical Guide

*Part of my AI Learning Journey | Last updated: May 2026*
*Updated with real-world findings from two projects: llm-eval-toolkit (Search Ad Copy Evaluator) and llm-issue-categorizer (Operational Ticket Categoriser)*

---

## The Problem This Solves

Most LLM API guides are written for engineering teams with cloud budgets. They recommend paid tiers, assume you have DevOps support, and treat cost as a variable to optimise rather than a hard constraint.

Solo builders face a different set of constraints:

- **Cost ceiling is often ₹0** — this is a portfolio project or a personal tool, not a funded product
- **No DevOps** — whatever you choose has to deploy with one command and stay up without maintenance
- **Public URL is required** — the whole point is a shareable demo, not something that only runs on your laptop
- **Time is limited** — you have evenings and weekends, not sprints

This guide maps the decision space for choosing an LLM API under these constraints, based on real experience building two tools across two months. The most important finding: **documented free tier limits and actual free tier limits are not the same thing.** Verify in AI Studio before committing to a model.

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

Several major providers offer genuinely free API access in 2026, no credit card required. The landscape as of May 2026:

| Provider | Free model | Documented RPD | Actual RPD (new projects) | RPM | Credit card |
|----------|-----------|---------------|--------------------------|-----|-------------|
| Google Gemini (Flash-Lite 2.5) | Gemini 2.5 Flash-Lite | 1,000 | **~20** ⚠️ | 15 | No |
| Google Gemini (Flash-Lite 3.1) | Gemini 3.1 Flash-Lite | 500 | **500** ✅ confirmed | 15 | No |
| Google Gemini (Flash 2.5) | Gemini 2.5 Flash | 500 | ~20 ⚠️ | 10 | No |
| Groq | Llama 4, Mixtral | ~14,400 | ~14,400 ✅ | varies | No |
| OpenAI | GPT-4o Mini | ~limited | limited | low | Yes (expiring credits) |
| Anthropic (Claude) | — | — | — | — | Yes (expiring $5 credits) |

**Key observations:**

Google is the most practical free-tier provider for solo builders. No credit card, no billing, and the API is OpenAI-compatible — meaning the code change from any other provider is minimal (swap base URL and key).

However, **the Gemini 2.5 series has a much tighter introductory quota than documented.** During llm-eval-toolkit batch testing, `gemini-2.5-flash-lite` returned a `429 RESOURCE_EXHAUSTED` error after 20 calls — not 1,000 as advertised. The same behaviour was observed for `gemini-2.5-flash`. This appears to be a new-project quota that applies before your account is established with Google.

`gemini-3.1-flash-lite` confirmed 500 RPD via the AI Studio rate limits dashboard. This is the recommended free-tier model as of May 2026.

Groq offers very high request volumes and exceptional speed (sub-100ms inference), but serves open-source models. For tasks requiring nuanced semantic reasoning — intent alignment, differentiation scoring, structured rubric output — open-source models at 7B–13B parameters may be less consistent than Gemini or Claude.

**⚠️ The critical rule: always verify actual limits in AI Studio before committing to a model.**

Go to [aistudio.google.com](https://aistudio.google.com) → your project → Rate limits. The dashboard shows your actual RPD and RPM limits, and your live usage over the past 28 days. Documented limits and actual limits differ — especially for newer models on new projects. Discovering a 20 RPD cap mid-build, after designing the tool around a 1,000 RPD assumption, costs time and requires rework.

**The free tier risk:** Google has reduced free quotas multiple times since 2024. Free tiers are not permanent entitlements — they can change. Build with the assumption that you may need to move to a paid tier if the project scales, and document the paid-tier fallback cost in your risk register.

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
- Under 50/day → most free tier options work
- 50–250/day → Gemini 3.1 Flash-Lite free tier (500 RPD confirmed)
- 250–500/day → Gemini 3.1 Flash-Lite free tier at the upper bound — monitor in AI Studio
- Over 500/day → paid tier required; at this volume you're beyond a solo portfolio project

**4a. Have you verified the actual limits for your account?**
- Go to AI Studio → Rate limits dashboard before finalising your model choice
- Do not rely on documented limits — verify your actual project quota
- If the model you planned to use shows a lower limit than expected, switch before building the tool around the wrong assumption

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

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Sign in with a personal Google account
3. Click "Get API key" — no credit card required
4. Check Rate limits dashboard — verify actual RPD for your target model before writing any code
5. Set the key as an environment variable or Codespaces Secret

**Python SDK — use `google-genai`, not `google-generativeai`:**

```bash
pip install google-genai
```

The older `google-generativeai` package is deprecated. It has a different import structure and API call format. Using it wastes debugging time.

**Basic call (current SDK):**
```python
from google import genai
import os

client = genai.Client(api_key=os.environ["GOOGLE_API_KEY"])
response = client.models.generate_content(
    model="gemini-3.1-flash-lite",
    contents="Your prompt here"
)
print(response.text)
```

**Recommended model as of May 2026:** `gemini-3.1-flash-lite` — 500 RPD confirmed on free tier, 15 RPM, no credit card required.

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

1. **Start with Gemini free tier across all projects** — one account, one key, zero cost
2. **Verify limits in AI Studio before designing each tool** — do not assume model limits carry over from previous projects or match documentation
3. **Use `gemini-3.1-flash-lite` as your default free-tier model** — 500 RPD confirmed as of May 2026; avoids the 20 RPD trap in the 2.5 series
4. **Add a paid tier only when a specific project demands it** — don't pre-empt this
5. **Document your fallback in each project's risk register** — if the free tier tightens, what's the paid-tier cost and what code change is required?

The goal is not to optimise every project independently. It's to have a stable, low-friction API layer that lets you focus on building, not on managing credentials and billing across five different providers.

---

## Further Reading

- [Google AI Studio](https://aistudio.google.com) — API key setup, rate limits dashboard (check actual limits here)
- [Gemini API pricing](https://ai.google.dev/pricing) — published free tier limits (verify against AI Studio)
- [Building with the Gemini API in Python](./building_with_gemini_api_python.md) — practical code patterns from two real builds
- [Ollama](https://ollama.com) — local model runner
- [LM Studio](https://lmstudio.ai) — GUI-based local model runner
- [Groq](https://groq.com) — free tier, high-speed open-source model inference
- [OpenRouter](https://openrouter.ai) — unified API for multiple providers, some free models

---

*Written as part of my public AI learning journey. I am a Senior TPM and Designated PM at Microsoft AI, building real AI products and documenting what I learn. Built and validated across two projects: [llm-eval-toolkit](https://github.com/saurabh-das7/llm-eval-toolkit) and [llm-issue-categorizer](https://github.com/saurabh-das7/llm-issue-categorizer).*
