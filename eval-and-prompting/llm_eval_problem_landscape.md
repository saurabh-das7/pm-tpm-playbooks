# LLM Evaluation — Problem Landscape for Product Managers

*Part of my AI Learning Journey | Last updated: April 2026*

---

## Why This Matters

As LLMs move from demos into production, one question becomes unavoidable: **how do you know if your model is actually doing its job well?**

This is not a question most teams answer rigorously. Output quality is often assessed informally — a few spot checks, a thumbs up from a stakeholder, an absence of complaints. That works until it doesn't. And when it stops working, you usually find out in production.

LLM evaluation (or "LLM eval") is the discipline of systematically assessing model output quality. It sits at the intersection of product management, data science, and quality engineering — and it is increasingly a core PM skill for anyone building AI products.

This document maps the six core problems that PMs encounter when trying to evaluate LLM outputs, drawn from hands-on experience building and shipping LLM-powered products.

---

## The Six Core Problems

### Problem 1 — Output Quality Assessment
**"Is this response actually good?"**

This is the most fundamental problem in LLM eval, and the hardest to answer cleanly. A response can be:
- Technically accurate but incomplete
- Fluent and well-written but hallucinated
- Relevant to the question but unhelpful in practice
- Correct for one user's context and wrong for another's

The core challenge is that **"good" is multi-dimensional**. No single metric captures it. Teams that reduce quality to a single score (accuracy, BLEU, ROUGE) tend to miss real failure modes. Teams that rely purely on human spot-checks can't scale.

The solution is structured, rubric-based evaluation — breaking quality into explicit dimensions (relevance, accuracy, completeness, clarity) and scoring each independently. This is common practice in academic NLP but surprisingly rare in production PM workflows.

**Where this shows up in practice:**
- QA-ing a new LLM feature before launch
- Comparing two model versions to decide which to ship
- Reviewing model outputs flagged by users for quality issues

---

### Problem 2 — Human-LLM Alignment
**"Does the model agree with what a human expert would say?"**

When LLMs are used to automate tasks previously done by humans — triaging issues, writing summaries, generating recommendations — the natural question is: how closely does the model's output match what a human expert would have produced?

This is the **model-human parity** problem. It sounds simple but is deceptively hard:

- **Human judgement is itself variable.** Two experts often disagree, especially on ambiguous cases. Your baseline is noisy before you even evaluate the model.
- **Labelled data is expensive and often outdated.** Ground truth datasets take time to build and go stale as the task definition evolves.
- **Complex tasks require decomposition.** For multi-step reasoning tasks (like root cause analysis), you need to evaluate each sub-step independently — and subjectivity compounds at each stage.
- **Parity is context-dependent.** 80% model-human parity may be acceptable for a low-stakes summarisation task and completely unacceptable for a medical diagnosis tool.

Defining what level of parity is "good enough" for a given use case is a product decision, not a data science one — which is why this is a core PM problem.

**Where this shows up in practice:**
- Automated RCA or diagnostic tools benchmarked against analyst judgement
- AI-generated summaries compared to human-written ones
- Customer support copilots evaluated against expert agent responses

---

### Problem 3 — Classification Accuracy and Explainability
**"Did the model label this correctly — and can it tell me why it got it wrong?"**

A growing class of LLM use cases involves classification: tagging content as eligible or ineligible, routing issues to the right team, categorising feedback into buckets. In these cases, the model is not generating free text — it is making a structured decision.

Two things matter in classification tasks, and you need both:

1. **The label** — is the classification correct?
2. **The reasoning** — is the explanation for the classification coherent and auditable?

Getting the label right but providing a wrong or circular explanation is a serious problem in regulated or high-stakes environments. Getting the explanation right but the label wrong means the model understands the task but is miscalibrated.

At high volumes, manual review of incorrect classifications is not sustainable. The common patch — a human override queue — is a symptom of an eval problem, not a solution to it.

**Where this shows up in practice:**
- Content moderation and policy enforcement (eligible / not eligible)
- Issue or ticket categorisation from unstructured text
- Ad quality and brand safety classification

---

### Problem 4 — Quality Drift Over Time
**"Is the model getting worse?"**

LLM quality is not static. It can degrade silently due to:
- Prompt changes made without systematic testing
- Fine-tuning on new data that introduces regressions
- Upstream model updates from the model provider
- Shifts in input distribution (users asking different kinds of questions)

Most teams have no systematic way to detect quality drift until something breaks visibly in production — a spike in user complaints, a failed audit, a business metric drop. By then, the degradation has often been accumulating for weeks.

Solving this requires three things: a **baseline** (what good looked like at launch), a **periodic re-evaluation process** (running the same eval set regularly), and a **regression detection mechanism** (surfacing when current quality falls below the baseline threshold).

This is an area where product and data science need to co-own the solution — and where PMs who understand eval can add disproportionate value.

**Where this shows up in practice:**
- Post-launch quality monitoring for any LLM feature
- Detecting regression after a model version upgrade
- Ensuring SLA compliance for AI-powered workflows

---

### Problem 5 — Scale and Automation
**"I can't manually review 10,000 outputs a day."**

Even when a team has a clear rubric and trained reviewers, manual evaluation doesn't scale. A model generating thousands of outputs per day cannot be reviewed by humans in anything close to real time.

The most common solution is **LLM-as-judge** — using a second LLM to evaluate the outputs of the first. This approach scales well and can be highly consistent, but it introduces its own reliability questions:

- Does the judge model have the same biases as the model being evaluated?
- Is the judge model's scoring consistent across runs?
- How do you validate that the judge model is itself reliable?

The answer is a calibration layer — periodically comparing LLM-as-judge scores against human scores to ensure the automated eval remains trustworthy. This is itself a product and process design problem.

**Where this shows up in practice:**
- Any high-volume LLM application where manual review is the bottleneck
- Automated eval pipelines for continuous quality monitoring
- A/B testing LLM variants at scale

---

### Problem 6 — Root Cause Isolation
**"Quality dropped — but where exactly, and why?"**

When overall quality degrades, an aggregate score tells you something is wrong but not where to look. Is it a prompt formulation issue? A specific input type the model handles poorly? A data pipeline problem affecting context quality? A model capability gap?

Without structured dimensional scoring, root cause isolation is guesswork. With it, patterns become visible: if accuracy scores drop while relevance and clarity hold, the problem is likely factual — which points toward retrieval quality or model knowledge gaps. If completeness drops across a specific category of inputs, the prompt may be under-specifying requirements for that category.

Dimensional scoring transforms eval from a quality gate into a **diagnostic tool** — which is where it becomes genuinely useful for product iteration.

**Where this shows up in practice:**
- Post-incident analysis after a quality regression
- Prioritising which aspect of a model to improve next
- Communicating quality issues to engineering with actionable specificity

---

## How These Problems Connect

These six problems are not independent. In practice, they compound:

- You cannot solve **Problem 4 (drift)** without first solving **Problem 1 (what good looks like)**
- You cannot scale **Problem 2 (human alignment)** without solving **Problem 5 (automation)**
- You cannot do **Problem 6 (root cause)** without the dimensional structure from **Problem 1**

A mature LLM eval practice addresses all six. Most teams are still working on Problem 1.

---

## A Framework for Where to Start

| Your situation | Start here |
|----------------|------------|
| Building a new LLM feature, no eval in place | Problem 1 — define your quality dimensions |
| Have a rubric but reviewing manually, volume growing | Problem 5 — build LLM-as-judge pipeline |
| Model replaced humans on a task, need to validate | Problem 2 — establish model-human parity baseline |
| LLM used as classifier, too many overrides | Problem 3 — audit classification + reasoning quality |
| Launched successfully but no ongoing monitoring | Problem 4 — set up drift detection |
| Quality is "fine" but you can't explain why it dips | Problem 6 — move to dimensional scoring |

---

## Further Reading

- [Anthropic Model Evaluation](https://www.anthropic.com/research)
- [RAGAS — Evaluation framework for RAG pipelines](https://github.com/explodinggradients/ragas)
- [LangChain Evals](https://docs.smith.langchain.com/)
- [Eugeneyan's writing on LLM evaluation](https://eugeneyan.com/writing/llm-evaluations/)

---

*Written as part of my public AI learning journey. I am a Senior TPM at Microsoft AI, documenting what I learn while building real AI products. See the `llm-eval-toolkit` repo for a hands-on project applying these concepts.*
