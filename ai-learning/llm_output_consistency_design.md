# LLM Output Consistency Design — Patterns That Actually Work

Lessons from building two LLM-powered products from scratch:
[llm-eval-toolkit](https://github.com/saurabh-das7/llm-eval-toolkit) (search ad copy evaluator)
and [llm-issue-categorizer](https://github.com/saurabh-das7/llm-issue-categorizer) (operational ticket categoriser).

Both tools required LLM output to be consistent, structured, and reliable across hundreds of
API calls. This playbook documents what broke, what fixed it, and the design patterns that held up.

---

## The Core Problem

LLMs are probabilistic. The same prompt, sent twice, can return different category names,
different score weightings, or differently structured JSON. For a one-off analysis this is
fine. For a tool that processes 100 rows in 10 batches and needs consistent category names
across all of them, it is not.

The solution is not to fight the probabilism — it is to design around it.

---

## Pattern 1 — Accumulated Context Across Batches

**The problem:** When categorising 90 tickets in 9 batches of 10, the LLM invented
"App Performance Issue" in batch 1 and "Application Stability Problem" in batch 5.
Same concept, different label. Downstream analysis was broken.

**The fix:** Pass the running list of categories already identified to every subsequent batch.

```python
accumulated_cats = []

for batch_num in range(total_batches):
    prompt = build_prompt(
        context=context,
        accumulated_categories=accumulated_cats,  # grows each batch
        batch=batch
    )
    results = call_llm(prompt)
    
    for res in results:
        cat = res.get("new_category", "")
        if cat and cat != "Uncategorised" and cat not in accumulated_cats:
            accumulated_cats.append(cat)
```

The prompt instructs the LLM: *"Use one of the existing categories if it fits. Only create
a new category if the ticket genuinely does not belong to any existing one."*

**Why it works:** The LLM is now constrained to a known vocabulary. It still exercises
judgment (the ticket may not fit any existing category) but the vocabulary is anchored.
Category drift across batches drops to near zero.

**What to watch:** The accumulated list grows. If it grows past ~15 categories, the LLM
starts force-fitting tickets to avoid creating more. Add a bucket threshold (see Pattern 3)
to handle genuine noise without inflating the taxonomy.

---

## Pattern 2 — Two-Pass Consolidation

**The problem:** Even with accumulated categories, near-duplicates slip through.
"UPI PIN Reset Issue" and "PIN Authentication Failure" are semantically the same
but the LLM created both across different batches.

**The fix:** After the full categorisation run, run a single consolidation pass.
This pass has full visibility of the final category distribution and can make
merge decisions with confidence that individual batch passes cannot.

```python
def build_consolidation_prompt(context, df):
    distribution = df["new_category"].value_counts().to_dict()
    
    return f"""You are reviewing the results of a ticket categorisation run.
Context: {context}
Category distribution: {distribution}

STAGE A — Write a one-line summary for each named category.
STAGE B — Identify merge opportunities:
  AUTO_MERGE: categories that are near-identical. Merge these.
  FLAG: categories that may overlap but you are uncertain. Flag only (max 3).
STAGE C — Write a 2-3 sentence narrative summarising the full results.

Return JSON only:
{{
  "narrative": "...",
  "category_summaries": {{"category": "summary", ...}},
  "auto_merges": [{{"merge_these": [...], "into": "..."}}],
  "merge_flags": ["..."],
  "uncategorised_analysis": ["..."]
}}"""
```

**Why it works:** The consolidation pass sees the full picture — distribution, counts,
and category names together. It can identify that "UPI PIN Reset Issue" (8 tickets)
and "PIN Authentication Failure" (3 tickets) are the same thing in a way that a
batch prompt processing 10 rows at a time cannot.

**The transparency principle:** Every auto-merge is logged with the source categories
and ticket count. Users can see exactly what was consolidated and why. High-confidence
merges happen automatically; uncertain cases are flagged for human review.

---

## Pattern 3 — Bucket Threshold

**The problem:** With 90 tickets across 10 batches, the LLM consistently created
1-2 row categories that were either genuine edge cases or misclassifications.
These categories cluttered the chart and made the distribution misleading.

**The fix:** After categorisation and before consolidation, collapse any category
with fewer than N rows into an "Uncategorised" bucket. Analyse that bucket separately.

```python
MIN_BUCKET_THRESHOLD = 5

def apply_bucket_threshold(df, threshold=MIN_BUCKET_THRESHOLD):
    cat_counts = df["new_category"].value_counts()
    small_cats = cat_counts[cat_counts < threshold].index.tolist()
    
    df = df.copy()
    df.loc[df["new_category"].isin(small_cats), "new_category"] = "Uncategorised"
    return df
```

**Why it works:** A chart with 6-8 meaningful categories plus an analysed Uncategorised
bucket is more useful to a PM than one with 20 categories where 14 have 1-2 rows each.
The Uncategorised bucket is not discarded — the consolidation pass analyses it separately
and surfaces the themes within it.

**Threshold calibration:** 5 rows works well for datasets of 80-100 rows. For larger
datasets, consider scaling to 3-5% of total rows.

---

## Pattern 4 — JSON Cleaning Before Parsing

**The problem:** Gemini (and most LLMs) wrap JSON responses in markdown fences
even when explicitly instructed not to. `json.loads()` fails on ` ```json ... ``` `.

**The fix:** Always strip before parsing. Never trust that the LLM followed the
"no markdown fences" instruction.

```python
def clean_json_response(raw_text):
    text = raw_text.strip()
    
    # Strip markdown fences
    if text.startswith("```"):
        lines = text.split("\n")
        # Remove first line (```json or ```) and last line (```)
        text = "\n".join(lines[1:-1]).strip()
    
    # Handle cases where the LLM added explanation before the JSON
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
parsed = json.loads(cleaned)
```

**Always use try/except:** Even after cleaning, parsing can fail. Graceful fallback
(mark the row as Uncategorised with Low confidence) is better than crashing the run.

```python
try:
    parsed = json.loads(cleaned)
except json.JSONDecodeError:
    # Fallback: mark all rows in batch as Uncategorised
    for i in range(len(batch)):
        result_df.at[start + i, "new_category"] = "Uncategorised"
        result_df.at[start + i, "confidence"] = "Low"
        result_df.at[start + i, "reasoning"] = "JSON parse error — batch failed"
```

---

## Pattern 5 — Never Trust LLM Arithmetic

**The problem:** In llm-eval-toolkit, the model was asked to return a weighted
average score across 5 dimensions. The returned scores were consistently off by
0.1-0.2. The LLM rounded intermediate calculations.

**The fix:** Never ask the LLM to calculate aggregates. Ask for the components
and calculate in Python.

```python
# Wrong — ask LLM for weighted average
# "Return the overall score as a weighted average of the five dimensions"

# Right — ask for raw scores, calculate yourself
weights = {
    "relevance": 0.20,
    "intent_alignment": 0.25,
    "differentiation": 0.25,
    "cta_strength": 0.20,
    "character_efficiency": 0.10
}

overall = sum(
    scores[dim] * weight
    for dim, weight in weights.items()
)
```

**The same principle applies to counts, percentages, and rankings.** If the output
requires arithmetic, do it in Python. Use the LLM for judgment, not calculation.

---

## Pattern 6 — Rate Limit Management

**The problem:** Gemini free tier is 15 RPM. A 90-row dataset in 9 batches of 10
fires 9 API calls. Without sleep, calls 5-9 hit rate limits and fail.

**The fix:** Sleep between batches. 4 seconds per batch stays within 15 RPM with
headroom for the consolidation pass.

```python
for batch_num in range(total_batches):
    # ... process batch ...
    
    if batch_num < total_batches - 1:
        time.sleep(4)  # 4 seconds = ~15 RPM max
```

**For multi-pass tools** (categorisation + consolidation + multi-tag): budget the
sleep across all passes. The consolidation pass is a single call — no sleep needed.
The multi-tag pass is another N-batch loop — same 4-second rule applies.

**Rate limit errors (429):** Catch and retry once after 8 seconds. If it fails again,
return a graceful fallback for that batch rather than crashing the full run.

```python
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
            return ""  # Graceful fallback
    raise
```

---

## Model Selection Decision

For both tools, `gemini-3.1-flash-lite` on the free tier was the right choice.

| Model | Free tier RPD | Suitable for |
|-------|--------------|--------------|
| `gemini-2.5-flash-lite` | ~20 RPD (actual, not documented) | Not suitable for demo tools |
| `gemini-3.1-flash-lite` | 500 RPD confirmed | Solo builds, portfolio demos |
| `gemini-2.5-flash` | 500 RPD | When output quality needs to be higher |

**Always verify limits in AI Studio dashboard** before committing to a model.
Documented limits and actual limits differ — especially for newer models.
Go to aistudio.google.com → your project → Rate limits.

---

## Prompt Design Principles

**Be explicit about output format.** Don't say "return JSON." Say:
*"Return a JSON object only — no markdown fences, no preamble, no explanation.
Start your response with `{` and end with `}`."*

**Use STAGE labels for multi-step reasoning.** When the prompt requires the LLM
to reason in sequence (generate summaries, then decide merges, then write narrative),
label each stage explicitly. The LLM follows the sequence more reliably.

**Pass context with every call.** A one-line description of the data type
("Support tickets for a logistics ops team") passed with every prompt dramatically
improves categorisation quality. "Slow response" means something different in
a payments app vs an AI assistant.

**Include the running category list in the prompt, not just the system context.**
Put it in the user message where the LLM's attention is highest.

---

## Summary

| Problem | Pattern | One-line fix |
|---------|---------|-------------|
| Category names drift across batches | Accumulated context | Pass running category list to every batch |
| Near-duplicate categories slip through | Two-pass consolidation | Full-picture merge pass after categorisation |
| Noise categories clutter output | Bucket threshold | Collapse <5 row categories to Uncategorised |
| JSON parse failures | Clean before parsing | Strip fences, find `{`, always try/except |
| LLM arithmetic is wrong | Calculate in Python | Ask for components, aggregate yourself |
| Rate limit errors | Sleep between batches | 4 seconds = ~15 RPM with headroom |

These patterns are transferable to any LLM-powered classification, evaluation,
or structured extraction tool. The specifics (batch size, threshold, sleep duration)
need calibration per use case — the patterns themselves are stable.
