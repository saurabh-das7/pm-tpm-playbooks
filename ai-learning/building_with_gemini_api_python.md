# Building with the Gemini API in Python — A Practical Guide

Real patterns from building two production Streamlit tools using Gemini's free tier:
[llm-eval-toolkit](https://github.com/saurabh-das7/llm-eval-toolkit) and
[llm-issue-categorizer](https://github.com/saurabh-das7/llm-issue-categorizer).

This is not a tutorial. It is a reference for the things that are not obvious from
the documentation — the SDK quirks, the rate limit realities, the model selection
decisions, and the patterns that make Gemini reliable in a production tool.

---

## SDK: Use `google-genai`, Not `google-generativeai`

The older SDK (`google-generativeai`) is deprecated. The new one is `google-genai`.
They have completely different import structures and API call formats. Using the wrong
one wastes hours.

```bash
pip install google-genai
```

```python
# Wrong — deprecated
import google.generativeai as genai
genai.configure(api_key=os.environ["GOOGLE_API_KEY"])
model = genai.GenerativeModel("gemini-pro")
response = model.generate_content("Hello")

# Right — current SDK
from google import genai

client = genai.Client(api_key=os.environ["GOOGLE_API_KEY_V2"])
response = client.models.generate_content(
    model="gemini-3.1-flash-lite",
    contents="Hello"
)
print(response.text)
```

---

## API Key Management

**In GitHub Codespaces:** Store the key as a Codespaces Secret, not in a `.env` file.
Go to github.com/settings/codespaces → Secrets → New secret.
The secret auto-injects as an environment variable — never needs to be typed again.

```python
import os
from google import genai

def get_client():
    api_key = os.environ.get("GOOGLE_API_KEY_V2")
    if not api_key:
        raise ValueError("GOOGLE_API_KEY_V2 not set in environment")
    return genai.Client(api_key=api_key)
```

**In Streamlit Community Cloud:** Add the key in the Secrets section of the app dashboard.
Format in `secrets.toml` (local only, never committed):

```toml
GOOGLE_API_KEY_V2 = "your_key_here"
```

Access in the app:

```python
import streamlit as st
import os

# Streamlit Cloud reads from secrets.toml
api_key = st.secrets.get("GOOGLE_API_KEY_V2") or os.environ.get("GOOGLE_API_KEY_V2")
```

**Always add `secrets.toml` to `.gitignore`:**
```
.streamlit/secrets.toml
```

---

## Model Selection

The right model depends on your use case and free tier limits. Always verify
actual limits in AI Studio dashboard — documented limits and actual limits differ.

Go to [aistudio.google.com](https://aistudio.google.com) → your project → Rate limits.

| Model | Free tier RPD | Free tier RPM | When to use |
|-------|--------------|--------------|-------------|
| `gemini-2.5-flash-lite` | ~20 (actual) | 15 | Don't — too restrictive for demo tools |
| `gemini-3.1-flash-lite` | 500 | 15 | Portfolio tools, solo builds |
| `gemini-2.5-flash` | 500 | 10 | When output quality needs to be higher |
| `gemini-2.5-pro` | 25 | 5 | Complex reasoning tasks, not batch tools |

**For batch tools (processing 50-100 rows):** `gemini-3.1-flash-lite` is the right
choice. 500 RPD is sufficient for multiple full runs per day. 15 RPM requires
4-second sleeps between batches of 10 — manageable.

**The `gemini-2.5-flash-lite` trap:** The documentation advertised 1,000 RPD for
new projects. The actual limit was 20 RPD — discovered during batch testing when
call 21 returned a 429. Confirmed in AI Studio dashboard. Always verify before
committing to a model in a tool design.

```python
MODEL = "gemini-3.1-flash-lite"
```

---

## Making API Calls

The basic pattern for a text generation call:

```python
from google import genai

client = genai.Client(api_key=api_key)

response = client.models.generate_content(
    model="gemini-3.1-flash-lite",
    contents=prompt  # str or list of parts
)

text = response.text  # The generated text
```

**Always wrap in try/except:**

```python
def call_gemini(client, prompt):
    try:
        response = client.models.generate_content(
            model="gemini-3.1-flash-lite",
            contents=prompt
        )
        return response.text
    except Exception as e:
        if "429" in str(e):
            # Rate limit — retry once after 8 seconds
            import time
            time.sleep(8)
            try:
                response = client.models.generate_content(
                    model="gemini-3.1-flash-lite",
                    contents=prompt
                )
                return response.text
            except Exception:
                return ""  # Graceful fallback
        raise
```

---

## Rate Limit Management

The free tier is 15 RPM (requests per minute). For a batch tool processing
10 rows per call, this means:

- 9 batches for a 90-row dataset
- At 15 RPM maximum: one call every 4 seconds
- 4-second sleep between batches stays within limit with headroom

```python
import time

BATCH_SIZE = 10

for batch_num in range(total_batches):
    start = batch_num * BATCH_SIZE
    end = min(start + BATCH_SIZE, len(rows))
    batch = rows[start:end]
    
    result = call_gemini(client, build_prompt(batch))
    process_result(result)
    
    # Sleep between batches — not after the last one
    if batch_num < total_batches - 1:
        time.sleep(4)
```

**For multi-pass tools:** If your tool runs multiple LLM passes (categorisation →
consolidation → multi-tag), budget the sleeps across all passes. A consolidation
pass that is a single call needs no sleep. The next batch loop needs the 4-second rule.

**503 transient errors:** Retry once after 2 seconds. These are temporary
server-side issues, not rate limits.

---

## Getting Structured JSON Output

Gemini does not reliably return clean JSON even when instructed to. Always clean
the response before parsing.

**Prompt instruction:**
```
Return a JSON object only — no markdown fences, no preamble, no explanation.
Start your response with { and end with }.
```

**Cleaning function:**
```python
import json

def clean_json_response(raw_text):
    text = raw_text.strip()
    
    # Strip markdown fences (```json ... ``` or ``` ... ```)
    if text.startswith("```"):
        lines = text.split("\n")
        text = "\n".join(lines[1:-1]).strip()
    
    # Find the start of the JSON object
    start = text.find("{")
    if start > 0:
        text = text[start:]
    
    # Handle array responses
    if not text.startswith("{"):
        start = text.find("[")
        if start >= 0:
            text = text[start:]
    
    return text

# Usage
raw = call_gemini(client, prompt)
cleaned = clean_json_response(raw)

try:
    parsed = json.loads(cleaned)
except json.JSONDecodeError as e:
    # Log and return fallback
    print(f"JSON parse error: {e}")
    parsed = {}
```

**For batch responses** where each row needs a structured result, design the
prompt to return a JSON array — one object per row:

```python
# Prompt instructs:
# Return a JSON array with one object per ticket:
# [{"ticket_id": "...", "new_category": "...", "confidence": "High/Medium/Low", "reasoning": "..."}, ...]

def parse_batch_response(raw, batch):
    cleaned = clean_json_response(raw)
    try:
        results = json.loads(cleaned)
        if isinstance(results, list):
            return results
    except json.JSONDecodeError:
        pass
    
    # Fallback: return Uncategorised for all rows in batch
    return [
        {"ticket_id": row.get("ticket_id", ""), "new_category": "Uncategorised",
         "confidence": "Low", "reasoning": "Parse error"}
        for row in batch
    ]
```

---

## Prompt Engineering for Reliable Output

**Be explicit about format.** Vague: *"Return JSON."*
Clear: *"Return a JSON array only — no markdown fences, no text before or after.
Start with [ and end with ]."*

**Use STAGE labels for multi-step reasoning.**
When the task requires sequential reasoning (generate, then evaluate, then summarise),
label each stage:

```
STAGE A — Write a one-line summary for each category.
STAGE B — Using the Stage A summaries, identify merge opportunities.
STAGE C — Write a 2-3 sentence narrative summarising the full results.
```

The LLM follows labelled stages more reliably than implicit sequencing.

**Pass context with every call.**
A one-line description of the data domain dramatically improves output quality:

```python
prompt = f"""Context: {context}

Categorise the following tickets...
"""
```

"Slow response" means something different in a payments app vs an AI assistant.
The context anchors the LLM's interpretation.

**Constrain vocabulary for consistency.**
Pass the list of already-identified categories with every batch prompt:

```python
if accumulated_categories:
    cat_list = "\n".join(f"- {c}" for c in accumulated_categories)
    cats_section = f"\nExisting categories identified so far:\n{cat_list}\n\nUse one of these if it fits."
else:
    cats_section = "\nNo categories identified yet — generate your own."
```

---

## Full Working Example

A minimal but production-ready categorisation engine:

```python
import os
import json
import time
from google import genai

MODEL = "gemini-3.1-flash-lite"
BATCH_SIZE = 10

def get_client():
    return genai.Client(api_key=os.environ["GOOGLE_API_KEY_V2"])

def call_gemini(client, prompt):
    try:
        response = client.models.generate_content(model=MODEL, contents=prompt)
        return response.text
    except Exception as e:
        if "429" in str(e):
            time.sleep(8)
            try:
                response = client.models.generate_content(model=MODEL, contents=prompt)
                return response.text
            except Exception:
                return ""
        raise

def clean_json(raw):
    text = raw.strip()
    if text.startswith("```"):
        text = "\n".join(text.split("\n")[1:-1]).strip()
    start = text.find("[")
    return text[start:] if start >= 0 else text

def categorise(rows, context):
    client = get_client()
    accumulated = []
    results = []

    batches = [rows[i:i+BATCH_SIZE] for i in range(0, len(rows), BATCH_SIZE)]

    for idx, batch in enumerate(batches):
        cat_section = (
            "Existing categories:\n" + "\n".join(f"- {c}" for c in accumulated)
            if accumulated else "No categories yet — generate your own."
        )

        prompt = f"""Context: {context}
{cat_section}

Categorise each ticket. Return JSON array only:
[{{"id": "...", "category": "...", "confidence": "High/Medium/Low", "reasoning": "..."}}]

Tickets:
{json.dumps(batch, indent=2)}"""

        raw = call_gemini(client, prompt)

        try:
            batch_results = json.loads(clean_json(raw))
            results.extend(batch_results)
            for r in batch_results:
                cat = r.get("category", "")
                if cat and cat != "Uncategorised" and cat not in accumulated:
                    accumulated.append(cat)
        except json.JSONDecodeError:
            results.extend([
                {"id": row.get("id", ""), "category": "Uncategorised",
                 "confidence": "Low", "reasoning": "Parse error"}
                for row in batch
            ])

        if idx < len(batches) - 1:
            time.sleep(4)

    return results
```

---

## Deployment Checklist

- [ ] Using `google-genai` (not `google-generativeai`)
- [ ] API key stored as environment variable or Streamlit secret — not hardcoded
- [ ] `secrets.toml` in `.gitignore`
- [ ] Model confirmed in AI Studio dashboard (actual RPD, not documented RPD)
- [ ] 4-second sleep between batches
- [ ] JSON cleaning before `json.loads()`
- [ ] Try/except on every API call with graceful fallback
- [ ] Rate limit retry (429) with 8-second wait
- [ ] Never asking the LLM to calculate aggregates — do it in Python
