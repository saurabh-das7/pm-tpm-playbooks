# PM & TPM Playbooks

A living collection of frameworks, templates, and structured thinking
built from real product and program work across Ad Tech, AI/ML, and
Marketplace products.

Written from the perspective of someone who has shipped LLM-powered
products, measurement frameworks, and large-scale ad format launches
at a top-tier tech company.

---

## What's inside

### 🤖 api-and-infra
Practical guides for choosing, setting up, and managing the API and
infrastructure layer for AI products — built from real experience, not
documentation.

| Doc | What it covers |
|-----|---------------|
| [llm_api_selection_for_solo_builds.md](./api-and-infra/llm_api_selection_for_solo_builds.md) | How to choose an LLM API at ₹0 cost — free tier realities, model selection, consolidation strategy |
| [building_with_gemini_api_python.md](./api-and-infra/building_with_gemini_api_python.md) | Practical Gemini API patterns from two real builds — SDK quirks, rate limits, JSON cleaning, error handling |
| [deploying_ai_apps_for_free.md](./api-and-infra/deploying_ai_apps_for_free.md) | Streamlit Community Cloud deployment guide — secrets, cold start, continuous deployment |

---

### 🧪 eval-and-prompting
Frameworks for designing LLM evaluation systems, writing reliable prompts,
and managing output consistency at scale.

| Doc | What it covers |
|-----|---------------|
| [llm_eval_problem_landscape.md](./eval-and-prompting/llm_eval_problem_landscape.md) | The six core LLM evaluation problems PMs face — a landscape map |
| [rubric_based_llm_evaluation_design.md](./eval-and-prompting/rubric_based_llm_evaluation_design.md) | How to design multi-dimensional scoring rubrics — dimensions, weights, verdict logic |
| [system_prompt_design_for_llm_evaluation.md](./eval-and-prompting/system_prompt_design_for_llm_evaluation.md) | Writing prompts that produce reliable, structured evaluation output — including batch and multi-pass patterns |
| [llm_output_consistency_design.md](./eval-and-prompting/llm_output_consistency_design.md) | Patterns for consistent LLM output across batches — accumulated context, two-pass consolidation, bucket thresholds |

---

### 🎯 product-and-ux
PM frameworks for designing AI products and documenting them with the
rigour of a real product team.

| Doc | What it covers |
|-----|---------------|
| [zero_friction_ux_for_ai_tools.md](./product-and-ux/zero_friction_ux_for_ai_tools.md) | Seven UX design patterns for AI tools — sample pre-population, cascading dropdowns, live previews, verdict-first output |
| [pm_documentation_for_solo_ai_builds.md](./product-and-ux/pm_documentation_for_solo_ai_builds.md) | Stage-by-stage PM documentation guide — problem statement through retrospective |

---

## How to use this

Each document is standalone — read what's relevant to your current problem.
The eval-and-prompting and api-and-infra folders are the most immediately
practical for anyone building LLM-powered tools. The product-and-ux folder
is most useful when starting a new project or preparing a portfolio piece.

## Built from

These frameworks were developed and validated across two public projects:

| Project | What it is |
|---------|-----------|
| [llm-eval-toolkit](https://github.com/saurabh-das7/llm-eval-toolkit) | Search Ad Copy Evaluator — Streamlit app evaluating LLM-generated ad copy against a structured rubric · [Live demo](https://llm-eval-toolkit-uwvrvxbgvcgwmk9rpbpjun.streamlit.app/) |
| [llm-issue-categorizer](https://github.com/saurabh-das7/llm-issue-categorizer) | Operational Ticket Categoriser — LLM-powered tool to re-categorise vendor tickets and surface product patterns |

## Status

🚧 Actively building — frameworks are added as they are developed and
validated across real projects.

---

*By Saurabh Das · [LinkedIn](https://linkedin.com/in/saurabhdas7)*
