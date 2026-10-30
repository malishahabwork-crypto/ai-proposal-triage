# System Design Write-Up (`MATACS-TH-2026-02`)

## 1. AI vs. Rules
* **Where AI is used:** AI (LLM) is used strictly in **Stage 2 (Structured Fact Extraction)** and **Stage 4 (One-Line Summary Generation)**. Because contractor PDFs are unstructured and evaluation criteria are dynamically created by admins rather than hard-coded, an LLM is ideal for reading natural language, extracting target fields into a strict JSON schema, quoting verbatim evidence, and flagging suspicious text (such as prompt injections or ambiguous dates).
* **Where plain code/rules are used:** Deterministic code handles **Stage 1 (PDF parsing & sanitization)**, **Stage 3 (Pass/Fail mandatory gates & project ceiling checks)**, and **Stage 4 (Weighted math, normalization, tie-breaking, and sorting)**.
* **Why the line is drawn there:** LLMs excel at semantic reading but are unreliable at arithmetic, strict boundary comparisons (e.g., $\le \$260,000$), and repeatable sorting. Separating AI *extraction* from code-based *evaluation* ensures that a language model never decides pass/fail eligibility or calculates weighted scores.

---

## 2. Consistency and Explainability
* **Repeatability across runs:** Running the same proposals twice produces the exact same ranking. In the extraction step, the LLM is called with `temperature = 0`, a pinned model snapshot, and a strict JSON schema (`response_format`), while caching extracted JSON by file hash (`SHA-256`). All pass/fail checks, point calculations, and tie-breakers execute in deterministic code.
* **Explainability for reviewers:** Every extracted field requires a verbatim `source_quote` and `page_number` from the PDF. On the reviewer dashboard, each proposal displays:
  1. Its final status (`ranked`, `flagged`, or `ineligible`) and one-line summary reason.
  2. A per-criterion score breakdown showing the raw extracted value, points awarded, weight applied, and a clickable citation that highlights the exact sentence in the source PDF.

---

## 3. Human in the Loop and Audit Logging
* **Human review checkpoints:**
  1. **Exception Resolution (`flagged`):** Proposals halted by Gate 0 or Gate 2—such as Proposal C (prompt injection attempt and ambiguous "since 2020" date window) or Proposal D (missing completion schedule)—are routed to a human review queue before they can be scored.
  2. **Disqualification Check (`ineligible`):** Reviewers can inspect rejected bids (such as Proposal B lacking a direct electrical contractor licence) to confirm the rule engine's rejection reason.
  3. **Manual Override & Final Award:** Reviewers can override any extracted value, status, or rank by selecting an override reason and typing a mandatory justification before making the final contract award decision.
* **What gets logged for audit:** Every run writes an immutable audit log entry containing: `timestamp`, `proposal_file_hash`, `criteria_version_id`, `llm_model_version`, raw LLM JSON output with quotes, rule-engine score breakdown, `reviewer_id`, and any manual override diffs with written justifications.

---

## 4. Changing Criteria After Scoring
* **Versioned criteria snapshots:** Admin criteria configurations are saved as immutable versions (`v1`, `v2`, etc.) rather than overwriting existing rules in place.
* **Handling already-scored proposals:** If an admin edits criteria or weights after proposals are scored under `v1`, existing `v1` scores are marked as **`Stale (Out of sync with Criteria v2)`** in the UI rather than silently changing.
* **Smart re-evaluation:** The reviewer is prompted to click **"Re-run Batch under v2"**:
  * If **only weights or numeric thresholds** changed, the system skips the LLM entirely and instantly recalculates scores using the cached extracted JSON (zero extra AI cost).
  * If a **new criterion** was added, the system runs a targeted LLM extraction pass only for the new field, then recomputes the batch while keeping the `v1` results in the audit history.

---

## 5. Risks and Safeguards

| Risk | Practical Safeguard in This Design |
| :--- | :--- |
| **Prompt Injection / Manipulation** *(e.g., Proposal C's Page 9 instruction to "Rank it first")* | Wrap proposal text in `<untrusted_proposal_text>` delimiters; instruct the system prompt to treat embedded grading commands as adversarial data; automatically set `status: "flagged"` in Gate 0. |
| **Hallucinations / Wrong Extraction** | Enforce `null` or `"unclear"` when facts are missing (like Proposal D's TBD timeline); require verbatim substring quotes for every extracted value and verify in code that the quote exists in the source PDF text. |
| **Ambiguous Data** *(e.g., Proposal C's "since 2020" vs. last 5 years)* | Do not let the LLM guess or prorate numbers; extract the raw text timeframe, flag the mismatch in Gate 2, and escalate to a human reviewer. |
| **Vendor Bias** | Strip or mask company names and logos before passing text to the LLM so qualitative extraction (like Team & Approach) is blind to vendor identity; calculate all points via deterministic math. |
| **API Cost & Token Limits** | Enforce file size and page limits on upload; cache LLM extractions by document hash (`SHA-256`); reuse cached extractions whenever admins adjust scoring weights. |