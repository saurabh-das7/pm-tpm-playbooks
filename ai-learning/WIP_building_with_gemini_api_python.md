# Building with the Gemini API in Python — A Practical Guide

*Part of my AI Learning Journey | Last updated: —*
*🚧 WIP — template created April 2026, to be completed after Milestone 1 of the Search Ad Copy Evaluator build*

---

## Why This Document Is a WIP

A guide to building with the Gemini API is most useful when it contains working code from a real project — not sanitised examples from documentation. Code snippets that actually ran, error messages that were actually hit, and fixes that actually worked are more valuable than anything written before the build.

This document will be completed during and after Milestone 1 (evaluation engine) of the Search Ad Copy Evaluator. The goal is to capture the real integration experience — including what the documentation doesn't tell you.

---

## What This Will Cover

*(Sections to be written from real build experience)*

---

### 1. Setup and Authentication

What to capture:
- Exact pip install command and version used
- API key setup — Google AI Studio flow, where the key lives, how to verify it works
- The first working API call — minimal working example from the actual project
- Common authentication errors and fixes

---

### 2. Structuring the API Call for Evaluation Tasks

What to capture:
- Full API call structure used in the evaluator (with real parameters)
- Temperature and max_tokens settings chosen and why
- How system prompt and user message are structured in the SDK
- The OpenAI-compatible endpoint vs native SDK — which was used and why

---

### 3. Getting Reliable JSON Output

What to capture:
- How to prompt for JSON-only output reliably
- How to parse and validate the response
- What happens when the model doesn't return valid JSON and how to handle it
- Whether `response_mime_type="application/json"` (Gemini native feature) was used

---

### 4. Error Handling in Practice

What to capture:
- Rate limit errors — what they look like, how to handle them gracefully in Streamlit
- Timeout handling
- Empty or malformed responses
- The error handling pattern that ended up in the final code

---

### 5. Performance and Latency

What to capture:
- Real p50 and p95 latency numbers from the live app
- Whether latency met the ≤10 second target from the PRD
- What affected latency most (prompt length, model choice, etc.)

---

### 6. Free Tier Behaviour in Practice

What to capture:
- Whether the 1,000 req/day limit was hit during development
- How rate limit errors surface (HTTP status, error message format)
- Whether the limit reset behaviour was as documented (midnight Pacific)

---

## Code Snippets

*(Real, tested code to be added here during Milestone 1)*

```python
# Placeholder — replace with actual working code from the build
```

---

## Gotchas Not in the Documentation

*(To be filled in during build — the surprises that weren't obvious from docs)*

---

## Related Documents

- [LLM API Selection for Solo Builds](./llm_api_selection_for_solo_builds.md)
- [System Prompt Design for LLM Evaluation](./system_prompt_design_for_llm_evaluation.md)
- [Deploying AI Apps for Free](./deploying_ai_apps_for_free.md)
- [LLM Eval Toolkit — Build Log](https://github.com/saurabh-das7/llm-eval-toolkit/blob/main/docs/07_build_log.md)

---

*Written as part of my public AI learning journey. I am a Senior TPM and Designated PM at Microsoft AI, building real AI products and documenting what I learn.*
