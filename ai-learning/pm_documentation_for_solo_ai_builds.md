# PM Documentation for Solo AI Builds — A Stage-by-Stage Guide

*Part of my AI Learning Journey | Last updated: April 2026*
*v1 — will be updated after the first project reaches launch*

---

## Why Document a Solo Build?

Two reasons, and they're different.

**Reason 1 — Portfolio signal.** A GitHub repo with working code tells a hiring manager you can build. A GitHub repo with working code and PM-quality documentation tells them you think like a product person, not just a builder. The documentation is the differentiation.

**Reason 2 — Better decisions.** Writing forces clarity. A problem statement you can't write clearly is a problem you don't fully understand. A PRD with no non-goals is a PRD you haven't thought through. The process of documenting forces you to make decisions you would otherwise defer — and deferred decisions become scope creep.

This guide captures the documentation stages for a solo AI build, what each document should contain, and the principles that make PM documentation different from engineering documentation.

---

## The Documentation Philosophy

**Document decisions, not just outcomes.** The most valuable thing about a PM document is not what was decided — it's why. A hiring manager reading your PRD doesn't need to know that you chose Streamlit. They want to know that you evaluated alternatives, understood the trade-offs, and made a reasoned choice.

**Write for two audiences simultaneously.** The primary audience is your future self — the person who picks up this project after two weeks away and needs to remember why things are the way they are. The secondary audience is anyone who lands on your GitHub. Write with the authenticity of the first audience and the clarity of the second.

**Documents are living artifacts.** A problem statement written before you build will be approximately right. A problem statement written after you've built and shipped will be more precisely right. Version your documents — update them as you learn, and note what changed and why.

**Decisions have context; documents should capture that context.** "We chose X" ages poorly. "We chose X because of Y constraint, evaluating A, B, and C alternatives" ages well and demonstrates thinking.

---

## The Eight Stages

### Stage 1 — Problem Statement

**What it is:** The document that answers "why does this exist and who is it for?"

**What it must contain:**
- The problem in concrete, specific terms — not "LLMs produce low quality output" but "LLMs generate search ads that pass grammar checks and fail real auctions"
- Who feels this problem acutely — one specific user archetype, not a broad persona
- The cost of the problem — what actually breaks when this goes unsolved
- Why now — what has changed that makes this problem more urgent today

**What makes it PM-quality:**
- The problem is separated from the solution — the problem statement contains no product decisions
- The "who" is specific enough to make a product decision against — "performance marketers at growth-stage startups" not "marketers"
- The cost section is concrete — money wasted, time lost, decisions delayed — not vague statements about inefficiency

**Common mistake:** Writing a problem statement that's really a product description dressed up as a problem. If your problem statement contains the word "tool", "app", or "platform", you've drifted into solution space.

---

### Stage 2a — Product Requirements Document (PRD)

**What it is:** The contract between the problem and the solution — what the product does, for whom, under what constraints, and how success is measured.

**What it must contain:**
- Product overview and core job to be done (one sentence)
- Primary user (one specific archetype, not a list)
- Goals — what v1 accomplishes
- Non-goals — what v1 explicitly does not do, with rationale for each exclusion
- The feature set, specified as user flows not engineering tasks
- Evaluation rubric or scoring design (for AI products)
- Success metrics — how you know it's working
- Design principles — the values that govern trade-off decisions
- Open questions — decisions not yet made, with resolution timelines

**What makes it PM-quality:**
- Non-goals have rationale — "we're not doing X because Y" not just a list of exclusions
- Success metrics are measurable and specific — "≥75% agreement with expert reviewer verdict" not "users find the output useful"
- Open questions are honest — a PRD with no open questions is either very late-stage or dishonest

**Common mistake:** A PRD that's really a feature list. If your PRD has no non-goals, no success metrics, and no open questions, it's a spec document, not a PRD.

---

### Stage 2b — Sample / Test Data Bank

**What it is:** For AI evaluation tools, a curated set of inputs that power the demo experience and anchor the rubric.

**What it must contain:**
- Sample inputs across multiple scenarios
- Deliberate quality variation — at least one strong, one mixed, one weak sample per scenario
- Expected verdict for each sample (what the tool should return)
- Rationale for why each sample produces its verdict — connecting to specific rubric dimensions

**What makes it PM-quality:**
- Samples are realistic — they could have been produced by a real user doing a real task
- Variant B (mixed quality) is genuinely non-obvious — it should require the rubric to catch, not gut feel
- The bank is extensible — new samples can be added following a documented format

---

### Stage 3 — UX Flow and Wireframe

**What it is:** The document that defines how users move through the product — what they see, what they can do, and in what order.

**What it must contain:**
- Interface structure (tabs, panels, navigation)
- One flow per major user journey, written as a sequence of steps
- Edge case and error handling for each flow
- Design principles that governed layout and interaction decisions
- Wireframe screenshots or ASCII diagrams of key screens

**What makes it PM-quality:**
- Flows include the zero-friction entry point — how does a first-time user experience the tool without prior knowledge?
- Edge cases are documented — what happens on empty input, invalid format, or ambiguous data?
- Design decisions are explained — "we used mode toggles instead of separate pages because X"

**Common mistake:** UX docs that describe what the screen looks like but not how the user gets there or what happens next. A wireframe without a flow is a picture, not a design.

---

### Stage 4 — Tech Stack Decisions

**What it is:** A record of what was built with, what alternatives were evaluated, and why each choice was made.

**What it must contain:**
- One section per major decision area (framework, API, hosting, data storage, etc.)
- For each: the question being answered, the options evaluated, the decision, and the rationale
- Explicit "why not" for the alternatives that were considered but rejected
- A stack summary table

**What makes it PM-quality:**
- Decisions have business/product rationale, not just technical rationale — "we chose X because it fits a ₹0 API budget and the portfolio consolidation goal" not just "X has better performance"
- Constraints are stated — the decision is made in the context of real constraints, not hypothetical best-case scenarios
- Limitations are acknowledged — every choice has a downside; name it

**Common mistake:** Tech stack docs that list what was used without explaining why. Any developer can read a requirements.txt. The value is in the reasoning.

---

### Stage 5 — Risk and Cost Plan

**What it is:** The document that names what could go wrong and what it costs to run.

**What it must contain:**
- Risk register: each risk with probability, impact, and mitigation
- Cost breakdown: all ongoing costs (API, hosting, tooling) with monthly estimates
- Spend controls: what prevents runaway costs
- Escalation paths: what happens if a free tier is discontinued or a risk materialises

**What makes it PM-quality:**
- Risks are specific — "the free API tier may be restricted" not "API issues"
- Mitigations are actionable — "fallback to paid tier at $0.10/M tokens, estimated ₹50/month at current volume"
- Cost estimates are grounded in real pricing, not approximations

---

### Stage 6 — Roadmap

**What it is:** The timeline from zero to launched, broken into milestones with honest time estimates.

**What it must contain:**
- Milestones (not tasks) — outcomes, not activities
- Time estimates per milestone, calibrated to available hours per week
- The MVP definition — what is the minimum bar for "launched"
- Post-MVP backlog — features deferred to v2, in priority order

**What makes it PM-quality:**
- Time estimates account for the solo builder reality — weekend evenings, not 8-hour days
- The MVP is genuinely minimal — the temptation is to keep adding scope; the roadmap holds the line
- Dependencies are explicit — which milestone must complete before the next can start

---

### Stage 7 — Build Log

**What it is:** A running journal of what was built, what was learned, and what changed along the way.

**What it must contain:**
- Weekly entries (or per-session entries for intensive builds)
- What was accomplished
- What was harder than expected and why
- What changed from the plan and why
- What's next

**What makes it PM-quality:**
- Honest about difficulty — a build log that only records wins is a press release, not a journal
- Connects changes back to earlier decisions — "changed X because the assumption in the PRD turned out to be wrong"
- Shows learning — the best build logs demonstrate how the builder's thinking evolved

---

### Stage 8 — Launch and Retrospective

**What it is:** Pre-launch quality checklist and post-launch reflection, combined.

**What it must contain:**
- Launch checklist: everything that must be true before the URL is shared
- Post-launch reflection: what worked, what didn't, what would be done differently
- Next bets: the two or three highest-value improvements based on what was learned

**What makes it PM-quality:**
- The checklist is specific enough to actually verify — "evaluation output returns valid JSON" not "app works"
- The retrospective is candid — name what went wrong without hedging
- Next bets are prioritised — not a wishlist, but a ranked set of the highest-leverage next moves

---

## The Framing Principle

One framing decision affects everything: are you documenting a learning journey or building a portfolio?

The answer is both — but the framing should be learning journey.

Portfolio-framed documentation sounds like a job application: polished, selective, designed to impress. It omits the wrong turns, the reconsidered decisions, the honest difficulty. Hiring managers who read enough of these can smell the polish.

Learning journey-framed documentation sounds like a person thinking in public: curious, candid, specific about what was hard and why. It includes the reconsidered decisions and names them honestly. This is harder to write but more distinctive to read.

The practical difference: in your PRD, document the features you considered and decided not to build — and explain why. In your build log, document the moment you realised your initial approach was wrong — and what you did instead. In your retrospective, name the one decision you'd make differently — and what you'd do instead.

That candour is what separates a strong PM portfolio from a polished one.

---

*Written as part of my public AI learning journey. I am a Senior TPM and Designated PM at Microsoft AI, building real AI products and documenting what I learn. See the `llm-eval-toolkit` repo for a hands-on project applying these concepts.*
