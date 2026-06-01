# Rubric-Based LLM Evaluation Design — A PM's Framework

*Part of my AI Learning Journey | Last updated: April 2026*
*v1 — will be updated as more projects are completed and real evaluation data is available*

---

## Why This Matters

Most teams evaluate LLM outputs informally. A few spot checks, a thumbs up from a stakeholder, an absence of complaints. This works until it doesn't — and when it stops working, you usually find out in production.

Rubric-based evaluation is the structured alternative. Instead of asking "is this response good?", you ask a set of specific, answerable questions about the response — each with a defined scoring scale — and derive an overall quality signal from the answers.

This document captures the design principles behind building a rubric-based LLM evaluator, drawn from real experience defining evaluation frameworks for production AI systems and from building a search ad copy evaluation tool in April 2026.

---

## The Five Design Decisions

Every rubric-based evaluator requires five decisions. Getting them right up front prevents the most common failure modes.

---

### Decision 1 — What Dimensions to Evaluate

The dimensions are the categories of quality your rubric measures. They should be:

- **Independently scoreable** — a dimension should not depend on another dimension's score. If you can't score one without knowing the other, they're not truly independent.
- **Exhaustive together** — the full set of dimensions should cover everything that could make a response good or bad for this use case.
- **Specific to the task** — generic dimensions (accuracy, clarity, relevance) are a starting point but rarely sufficient. The best rubrics have task-specific dimensions that experienced practitioners immediately recognise.

**Example — search ad copy evaluator dimensions:**

| Dimension | What it measures | Why it's task-specific |
|-----------|-----------------|----------------------|
| Relevance | Does the ad reflect the product? | Generic, but necessary baseline |
| Intent alignment | Does the CTA match the searcher's decision stage? | Specific to search advertising — this is the most common failure mode |
| Differentiation | Is there a specific USP? | Specific to competitive ad auctions |
| CTA strength | Is the action signal specific and urgent? | Specific to direct response advertising |
| Character efficiency | Is every character earning its place? | Specific to search ad format constraints |

**How to identify your dimensions:**
1. Think about the most common failure modes in your domain — each one maps to a dimension
2. Think about what an expert reviewer would check — that's your rubric
3. Validate that each dimension is independently scoreable before finalising

---

### Decision 2 — Scoring Scale

The choice of scoring scale affects interpretability, resolution, and how aggregation works.

| Scale | Pros | Cons | Best for |
|-------|------|------|----------|
| 1–5 integer | Easy to explain, enough resolution | Middle score (3) is ambiguous | Most rubric-based tools |
| 1–10 integer | More resolution | False precision — humans can't reliably distinguish 6 from 7 | Not recommended |
| Pass/Fail binary | Simplest to explain | No nuance — misses borderline cases | Hard constraints only |
| 0.0–1.0 float | Precise | Hard to explain to non-technical users | ML pipelines |

**Recommendation:** 1–5 integer for user-facing rubrics. Every score should have a defined anchor:

| Score | Meaning |
|-------|---------|
| 5 | Excellent — no improvement needed |
| 4 | Good — minor issues only |
| 3 | Acceptable — notable issues but not critical |
| 2 | Weak — significant issues that affect quality |
| 1 | Failing — critical failure on this dimension |

Define these anchors in writing before you start scoring. Without anchors, different reviewers (human or LLM) will interpret the scale differently.

---

### Decision 3 — Weight Allocation

Not all dimensions matter equally. Weight allocation is a product decision, not a data science one — it encodes your view of which dimensions most affect the outcome you care about.

**Static weights:** Fixed percentages assigned to each dimension. Simpler to implement and explain. Appropriate when the evaluation context doesn't change significantly across inputs.

**Dynamic weights:** Weights that shift based on inferred properties of the input. More powerful but requires a mapping from input context to weight profile.

**Example — dynamic weights for search ad copy:**

The keyword signals the searcher's intent. Intent determines which dimensions matter most:

| Intent | Why weights shift |
|--------|------------------|
| Purchase | User is ready to act → CTA Strength and Intent Alignment matter most |
| Consideration | User is comparing → Differentiation and Relevance matter most |
| Awareness | User is learning → Clarity and Relevance matter most |

**Weight allocation rules:**
- Weights must sum to 100%
- No single dimension should dominate (>40%) unless there's a strong domain reason
- The rationale for each weight should be documentable — "why does CTA Strength get 30% on purchase-intent queries?" should have a clear answer

**A useful test:** If a single dimension scores 1/5, should the overall verdict still be PASS? If not, add a verdict downgrade rule — regardless of weighted average, a catastrophic failure on one dimension caps the overall verdict.

---

### Decision 4 — Verdict Logic

The verdict is the action signal derived from the overall score. It should be unambiguous and actionable.

**Recommended three-tier verdict system:**

| Score range | Verdict | Action implied |
|------------|---------|----------------|
| High (e.g. 4.0–5.0) | ✅ READY / PASS | Use as-is |
| Medium (e.g. 2.5–3.9) | ⚠️ NEEDS REVISION | Fix before use |
| Low (e.g. below 2.5) | ❌ REJECT | Start over |

**The fourth verdict state — INSUFFICIENT INPUT:**

Every rubric-based evaluator needs a "cannot evaluate" state for inputs that don't provide enough context to score reliably. Without this, the evaluator will hallucinate confident verdicts on weak inputs — which is worse than returning no verdict at all.

Trigger this state when:
- Required fields are missing or empty
- The input is too vague to evaluate any dimension meaningfully (e.g. "good product")
- The input format is wrong (e.g. a URL pasted where copy is expected)

**Design rule:** Never force a confident verdict on an insufficient input. Graceful failure is a feature, not a limitation.

---

### Decision 5 — Reasoning Output

A score without explanation is noise. The evaluator must produce reasoning alongside every score — not generic commentary, but input-specific explanation.

**What good reasoning looks like:**
> Intent alignment: 2/5 — CTA says "Learn More" but the keyword "buy nike shoes online" signals a user ready to purchase. The mismatch will cost clicks.

**What bad reasoning looks like:**
> Intent alignment: 2/5 — The intent alignment score is low, indicating a problem with how well the response aligns with user intent.

**Design rule for reasoning:** The explanation must be specific enough that the user knows what to change. If you removed the score and showed only the reasoning, the user should still know whether this dimension passed or failed.

---

## The LLM-as-Judge Pattern

For scalable evaluation, the evaluator itself is an LLM (commonly called "LLM-as-judge"). The model reads the rubric, scores the input across each dimension, and returns structured output.

**Prompt engineering for rubric-based evaluation:**

The system prompt should include:
1. The rubric dimensions with scoring anchors
2. The weight profile to apply
3. The context needed to score (product description, keyword, etc.)
4. The output format (JSON with dimension scores and reasoning)
5. An instruction to never force a verdict on insufficient input

**Example system prompt structure:**
```
You are an expert evaluator of search ad copy. Score the following ad 
across five dimensions using the rubric below. Return your response as 
JSON only.

RUBRIC:
- Relevance (1-5): Does the ad accurately reflect the product described?
  5 = precise match, 1 = misleading or disconnected
- Intent Alignment (1-5): Does the CTA match the inferred searcher intent?
  5 = perfect match, 1 = completely mismatched or absent
[... other dimensions ...]

WEIGHT PROFILE (apply these weights for overall score):
- Relevance: 15%
- Intent Alignment: 30%
[... other weights ...]

OUTPUT FORMAT:
{
  "dimensions": {
    "relevance": {"score": int, "reasoning": "one sentence"},
    ...
  },
  "overall_score": float,
  "verdict": "READY_TO_SERVE" | "NEEDS_REVISION" | "REJECT" | "INSUFFICIENT_INPUT",
  "evaluator_note": "one sentence naming the primary issue or strength"
}

If the input is too vague or incomplete to score reliably, return verdict: 
"INSUFFICIENT_INPUT" and explain what is missing in the evaluator_note.
```

---

## Validating Your Rubric

A rubric is only as good as its agreement with human expert judgment. Before deploying, validate:

**1. Human alignment check**
Score 10–20 representative inputs with both the rubric and a human expert independently. Calculate agreement rate per dimension and overall verdict. Target: ≥75% agreement on overall verdict.

**2. Consistency check**
Send the same input 3 times with slight paraphrasing. Scores should not vary by more than 0.5 per dimension. High variance indicates the rubric dimensions are under-specified or the scoring anchors are ambiguous.

**3. False negative check**
Intentionally submit inputs that a human expert would reject. If the rubric passes them, the failure mode usually lies in the dimension that most directly captures the failure — tighten that dimension's scoring anchor.

**4. Edge case coverage**
Test the inputs that are designed to break the tool — empty fields, placeholder text, non-English copy, over-limit characters. Confirm the INSUFFICIENT_INPUT verdict fires correctly.

---

## Common Failure Modes

| Failure mode | Symptom | Fix |
|-------------|---------|-----|
| Dimensions overlap | Changing one dimension's input changes another's score | Redefine dimensions to be truly independent |
| Anchors are subjective | Two reviewers consistently disagree on a 3 vs 4 | Write more specific anchor descriptions with examples |
| Middle score dominates | Most outputs score 3 on most dimensions | Review whether the scale ends are calibrated correctly |
| Verdict doesn't match intuition | A clearly bad input gets NEEDS REVISION, not REJECT | Adjust weight profile or add downgrade rules |
| No reasoning specificity | Reasoning sounds the same for different inputs | Add an explicit instruction: "reasoning must reference specific words from the input" |

---

## Extending the Rubric

Rubrics are not static. As you accumulate evaluation data, patterns emerge:

- **Add dimensions** when you notice a failure mode that none of your existing dimensions capture
- **Adjust weights** when post-deployment data shows that certain dimensions are stronger predictors of real-world performance
- **Tighten anchors** when you see inconsistent scoring on specific score values
- **Add input validation rules** when you discover new edge cases that produce unreliable verdicts

Document every change with a reason. A rubric that evolves without a changelog becomes unauditable — which is exactly what good evaluation is supposed to prevent.

---

## Further Reading

- [LLM Evaluation — Problem Landscape for PMs](./llm_eval_problem_landscape.md)
- [RAGAS — Evaluation framework for RAG pipelines](https://github.com/explodinggradients/ragas)
- [Eugeneyan's writing on LLM evaluations](https://eugeneyan.com/writing/llm-evaluations/)
- [Anthropic's model evaluation research](https://www.anthropic.com/research)

---

*Written as part of my public AI learning journey. I am a Senior TPM and Designated PM at Microsoft AI, building real AI products and documenting what I learn. See the `llm-eval-toolkit` repo for a hands-on project applying these concepts.*
