# Debugging LLM Output Inconsistency — A PM's Field Guide

*Part of my AI Learning Journey | Last updated: —*
*🚧 WIP — template created April 2026, to be completed after Milestone 1 and 3 of the Search Ad Copy Evaluator build*

---

## Why This Document Is a WIP

Debugging LLM output inconsistency is best documented with real examples — specific prompts, actual outputs, and the changes that fixed them. Generic advice without concrete cases is not useful.

This document will be completed after the evaluation engine is built and tested (Milestone 1) and after the Compare tab is live (Milestone 3). By then there will be real inconsistency cases to document honestly.

---

## What This Will Cover

*(Sections to be written from real build experience)*

---

### 1. Types of Inconsistency

What to capture:
- Verdict inconsistency — same input, different verdict across runs
- Score variance — same input, score differs by more than 1 point
- Reasoning drift — verdict is consistent but reasoning changes significantly
- Format inconsistency — JSON structure varies between calls

---

### 2. Diagnosing the Root Cause

What to capture:
- How to isolate whether the issue is in the prompt, the temperature setting, or the model
- How to tell if the problem is in a specific dimension vs the overall scoring
- What logging to add during development to surface inconsistency

---

### 3. Fixes That Actually Worked

What to capture:
- Temperature adjustments and their effect on consistency
- Prompt changes that reduced variance (with before/after examples)
- Anchor wording changes that made scoring more predictable
- Output format changes that reduced JSON errors

---

### 4. Fixes That Didn't Work

What to capture:
- Approaches tried that made things worse or had no effect
- Why they didn't work (in hindsight)

---

### 5. Validation Framework

What to capture:
- The consistency test protocol developed for this project
- Pass/fail thresholds that worked in practice
- How often re-validation is needed after prompt changes

---

## Placeholder for Real Cases

*(Add real cases here during and after Milestone 1 and 3)*

| Case | Input | Expected output | Actual output | Root cause | Fix applied |
|------|-------|----------------|--------------|------------|-------------|
| | | | | | |

---

## Related Documents

- [System Prompt Design for LLM Evaluation](./system_prompt_design_for_llm_evaluation.md)
- [Rubric-Based LLM Evaluation Design](./rubric_based_llm_evaluation_design.md)
- [LLM Eval Toolkit — Build Log](https://github.com/saurabh-das7/llm-eval-toolkit/blob/main/docs/07_build_log.md)

---

*Written as part of my public AI learning journey. I am a Senior TPM and Designated PM at Microsoft AI, building real AI products and documenting what I learn.*
