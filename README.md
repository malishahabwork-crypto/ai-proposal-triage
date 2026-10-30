# AI-Assisted Proposal Triage Tool Design (`MATACS-TH-2026-02`)

* **Candidate Name:** Muhammad Ali Shahab
* **Assessment Reference:** `MATACS-TH-2026-02`
* **Role:** Associate Software Developer
* **Time Spent:** ~60 minutes

---

## Repository Contents

* [**Process Flow Diagram (PNG)**](diagram/flow_diagram.png) · [**PDF Version**](diagram/flow_diagram.pdf) · [**Interactive HTML**](diagram/proposal_triage_flow_diagram.html) — End-to-end visual architecture illustrating PDF ingestion, schema-constrained AI extraction, sequential deterministic rule gates, mathematical scoring/tie-breaking, and human-in-the-loop checkpoints.
* [**System Design Write-Up (`WRITEUP.md`)**](WRITEUP.md) — Concise responses to the five core architectural questions (AI vs. rules boundary, repeatability & explainability, human oversight & audit logs, criteria versioning, and risk mitigation).
* [**AI Prompts (`PROMPTS.md`)**](PROMPTS.md) — Production system and user prompt templates for Stage 2 semantic fact extraction and Stage 4 one-line summary generation.
* [**Sample Set Ranking Output (`ranking.json`)**](ranking.json) — Final triage evaluation and ranking of sample Proposals A–E following the required JSON schema.

---

## Process Flow Architecture Preview

![Process Flow Diagram](diagram/flow_diagram.png)

---

## AI Tool Usage & Critical Refinement Disclosure

I used AI assistants (Claude and Gemini) to brainstorm QA edge cases across the sample proposals, structure the visual flowchart layout, and format the documentation templates. 

**Where I disagreed with and corrected the AI's output:**
During the initial analysis, the AI suggested either letting the LLM calculate the weighted rubric scores directly inside the prompt or automatically disqualifying Proposal D (`status: "ineligible"`) for lacking a numeric completion timeline. I rejected both suggestions:
1. **No Math in the LLM:** I moved all pass/fail checks, boundary comparisons, and weighted calculations into deterministic code so that scores are 100% repeatable and immune to LLM arithmetic drift.
2. **Flagging vs. Disqualifying Proposal D:** Rather than marking Proposal D `ineligible` (which is reserved for explicit mandatory compliance failures like Proposal B's unconfirmed subcontractor), I designed Gate 2 to route Proposal D to `status: "flagged"` (`rank: null, score: null`). Because Proposal D passes all mandatory licensing/insurance checks and offers the lowest valid price ($231,000 CAD) with 11 qualifying installations, flagging it allows a human reviewer to request schedule clarification rather than silently discarding a strong bid.
