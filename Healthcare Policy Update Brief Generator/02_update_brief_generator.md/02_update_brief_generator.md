# Deliverable 2 — Healthcare Policy Update Brief Generator

**Milestone:** M2 (Week 2) · **Tasks:** 2.1 Prepare and compare policy information · 2.2 Design the brief · 2.3 Integrate the LLM API

## 1. Comparing new information with history (Task 2.1)
`healthpolicy/change_detector.py` compares each collected record with the Memento store:

| Class | Rule |
|---|---|
| **new** | `item_id` not seen before for this source |
| **changed** | same `item_id`, different `content_hash`, and text similarity below the threshold (0.97) |
| cosmetic | hash differs but similarity ≥ 0.97 (whitespace or punctuation edits). Treated as unchanged, so it raises no alert |
| unchanged | identical `content_hash` |
| **removed** | previously active item missing from a page that was crawled successfully |

For changed items the detector also computes:
* a **sentence diff** (sentences added and removed);
* a **figure diff**, which pairs old and new numbers, percentages, amounts and time limits (for example `10 days → 30 days`).
  Numeric changes matter most for operations.

## 2. Brief design (Task 2.2)
Each brief is a JSON object (the `BRIEF_FIELDS` schema in `healthpolicy/briefs.py`):

| Field | Purpose |
|---|---|
| policy_name, issuing_authority, source_name, source_url, region, category | identification and traceability |
| change_type | new / changed / removed (set by the detector, not by the LLM) |
| published_or_effective_date | when the change occurred |
| summary | 2–3 plain-language sentences |
| what_changed | exactly what is different from the previous version |
| key_points | 3–5 bullets |
| affected_stakeholders | hospitals, insurers, labs, manufacturers, clinicians… |
| operational_relevance | `{level: high/medium/low, areas: [...], explanation}` |
| recommended_actions | concrete next steps |
| deadlines | time limits stated in the source |
| source_evidence | exact sentences copied from the source (grounding) |
| confidence, caveats | uncertainty, draft status, missing details |

Briefs are rendered to Markdown (`render_markdown`). Each run produces a digest (`outputs/digest_run*.md`) sorted
by operational relevance, plus a JSON file for further processing.

## 3. LLM integration (Task 2.3)
`healthpolicy/llm.py` gives one interface over several providers:

| Provider | Default model | Key |
|---|---|---|
| Google Gemini (default, free tier) | `gemini-2.5-flash` | `GEMINI_API_KEY` |
| OpenAI | `gpt-4o-mini` | `OPENAI_API_KEY` |
| Groq (OpenAI-compatible) | `llama-3.3-70b-versatile` | `GROQ_API_KEY` |
| offline | extractive fallback | none |

All providers use temperature 0.2 and JSON output mode, with retries and tolerant JSON parsing. If the LLM fails,
the pipeline falls back to the extractive brief and records the reason in `caveats`.

### Prompt design (grounding)
* **System prompt:** use only the provided source text; write "Not stated in source" when something is missing;
  evidence must be copied exactly; rate operational relevance high, medium or low; informational, not legal advice.
* **User prompt** for a changed policy: title and metadata, then the **previous version (from Memento)**, the
  **current version**, the detected added and removed sentences, the changed figures, rule-based relevance hints,
  and the JSON schema.
* For new items it sends the source text. For removed items it sends the last known text and explains that the
  notice is no longer listed.

### Validation (hallucination checks)
`validate()` runs on every brief:
* fills in any missing schema keys and records them in `schema_issues`;
* **evidence grounding** = share of `source_evidence` sentences that appear in the source text;
* **unsupported numbers** = numbers in the summary, changes, key points or deadlines that do not occur in the source.
  The unit test confirms that an invented "99 days" is caught.

## 4. Example output (changed policy, offline extractive mode, synthetic data)
> **Revision of Health Benefit Package Rates under AB PM-JAY** · CHANGED · NHA · IN · reimbursement · 🔴 HIGH
> **What changed:** + rates for 35 dialysis packages are added · + hospitals must update pre-authorisation requests
> from 1 August, and claims at old rates will be rejected after 30 days · − "…from the effective date."
> **Deadlines:** from 1 August, after 30 days

With an LLM provider enabled, the same input produces a fuller summary, an explanation of operational impact, and
specific recommended actions. Run `python run.py track --offline cycle2 --llm gemini` and include the generated
digest in your submission as the LLM example.

## 5. How to run
```bash
python run.py track --offline cycle1 --baseline-only      # build history
python run.py track --offline cycle2 --llm gemini         # briefs for 5 new, 3 changed, 1 removed
```
