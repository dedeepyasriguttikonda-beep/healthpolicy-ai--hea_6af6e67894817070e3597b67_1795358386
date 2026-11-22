# HealthPolicy AI: Healthcare Policy Tracker — Final Project Report

**Team:** Dedeepya Sri (Leader) · Tanav Sunkari · Sharuf Shaik · Sumanvitha Mandala
**Programme:** SkillWallet Group Project · **Duration:** 4 milestones

---

## Abstract
HealthPolicy AI is a Generative-AI system that monitors public healthcare policy sources and tells healthcare
organizations what changed. **Firecrawl** collects pages from eight official sources (CMS, FDA, HHS, MoHFW,
NHA/PM-JAY, CDSCO, IRDAI, WHO) as clean markdown. A preprocessing layer splits each page into structured policy
records. **Memento**, a persistent SQLite history of policies, versions, briefs and reviews, lets the system tell
genuinely new or changed content apart from content it has already reviewed. An **LLM** (Gemini, OpenAI or Groq)
then writes structured, source-grounded update briefs covering what changed, when, which policy is affected,
and the likely operational relevance. On a labelled evaluation scenario the system detected every new, changed and
removed policy (F1 = 1.0 per class), ignored a cosmetic-only edit, and raised no duplicate alerts on a re-run.

## 1. Introduction and problem statement
Hospitals, clinics, insurers and laboratories must keep up with reimbursement rules, regulatory notices, drug and
device alerts, and public-health guidance. This information is spread across many government websites, published
irregularly and in different formats. Tracking it by hand is slow, and the most important edits are often small:
a changed percentage, a new deadline, or a draft becoming final. These are easy to miss. The project builds a
tracker that (1) collects the information automatically, (2) remembers what has been seen, (3) finds meaningful
changes, and (4) explains them in a form an operations team can act on.

## 2. Objectives and how they were met
| Objective | Implementation |
|---|---|
| Understand healthcare policy tracking | Source and category analysis (§3), operational-relevance taxonomy (§7) |
| Retrieve policy information | Firecrawl `scrape` pipeline with retries and failure isolation (§4.1) |
| Process policy updates | Cleaning, segmentation, date and link extraction, stable IDs and content hashes (§4.2) |
| Generate policy update briefs | Grounded LLM prompt, JSON schema, validation (§4.4) |
| Implement persistent policy history | Memento store: runs, policies, versions, briefs, reviews (§4.3) |
| Build a tracking workflow | `run_cycle` pipeline, CLI and Streamlit dashboard (§5) |

## 3. Policy sources
Eight official, publicly accessible sources across India, the US and global health, in five categories:
reimbursement (CMS, NHA/PM-JAY, IRDAI), drug and device regulation (FDA, CDSCO), government notification (HHS,
MoHFW), and public-health guidance (WHO). The registry is `config/sources.yaml`.

## 4. System design
### 4.1 Collection — Firecrawl
`Firecrawl.scrape(url, formats=["markdown"], only_main_content=True, exclude_tags=[nav, footer, …], max_age=0)`.
Each source is retried with backoff. A failed source is logged with its error, and its items are never marked as
removed. An offline collector replays saved snapshots so the system can be tested repeatably.

### 4.2 Preprocessing
Boilerplate removal; segmentation into one record per heading block, per linked notice (listing pages), or per page;
date extraction in four formats; `item_id` = hash of source + normalised title, which stays stable across crawls;
`content_hash` = hash of normalised title + date + text, which detects change.

### 4.3 Persistent history — Memento
SQLite tables `runs`, `policies`, `versions` (append-only), `briefs` and `reviews`. Memento answers the question
"have we already reviewed this exact version?", which prevents duplicate alerts, and it supplies the previous
version for the diff. An optional mirror into the `memento-memory` knowledge graph adds semantic recall of
related past policies as extra LLM context.

### 4.4 Change detection and LLM briefs
Records are classified as new, changed (similarity < 0.97), cosmetic, unchanged or removed. Changed items also get
sentence-level and figure-level diffs. The LLM receives the previous and current versions, the diffs and rule-based
relevance hints, under strict grounding rules, and returns a JSON brief. Every brief is validated for schema
completeness, evidence grounding (quotes must appear in the source) and unsupported numbers (numbers that do not
appear in the source are flagged).

## 5. Implementation
| Module | Role |
|---|---|
| `collector.py` | Firecrawl + offline collectors |
| `preprocess.py` | Cleaning, segmentation, extraction, hashing |
| `memento_store.py` | Persistent policy history |
| `change_detector.py` | Change classification and diffs |
| `relevance.py` | Operational areas, stakeholders, urgency, deadlines |
| `llm.py`, `briefs.py` | LLM client, prompt, schema, validation, rendering |
| `pipeline.py` | End-to-end cycle and digest |
| `evaluate.py` | Evaluation suite |
| `app.py`, `run.py` | Streamlit dashboard and CLI |

Testing: `pytest` covers listing-page parsing, date formats, stable IDs, new/changed/removed detection, the LLM
prompt and validation (with a fake LLM that invents a number and a quote, both of which are caught), and an
end-to-end run with duplicate suppression. All tests pass.

## 6. Evaluation
### 6.1 Method
The evaluation uses a labelled two-crawl scenario. `cycle1` is the baseline: 19 policies from 8 sources. `cycle2`
is a later crawl containing **5 new** policies, **3 substantively changed** policies (a proposed fee schedule that
became final with a different rate, a package-rate revision with a new deadline, and a cashless-claim circular with
a new penalty), **1 removed** policy, and **1 cosmetic-only** edit. The snapshots are synthetic so that the ground
truth is known exactly (`data/fixtures/ground_truth.json`). Run `python run.py evaluate --llm <provider>`.

### 6.2 Results (measured, offline extractive mode)
| Area | Metric | Result |
|---|---|---|
| Retrieval | Sources collected | 8 / 8 |
| | Record extraction accuracy | 1.00 (23 / 23) |
| | Date extraction accuracy | 1.00 |
| Change detection | New — P / R / F1 | 1.00 / 1.00 / 1.00 (5/5) |
| | Changed — P / R / F1 | 1.00 / 1.00 / 1.00 (3/3) |
| | Removed — P / R / F1 | 1.00 / 1.00 / 1.00 (1/1) |
| | Cosmetic-edit false positives | 0 |
| Memento history | Items tracked / with multiple versions | 24 / 3 |
| | Duplicate alerts when the same crawl is re-run | 0 |
| Briefs | Generated / schema complete | 9 / 100% |
| | Evidence grounding / briefs with unsupported numbers | 1.00 / 0 |
| | Relevance distribution | 4 high · 5 medium · 0 low |

Note: in offline extractive mode the grounding and relevance-agreement scores are 1.0 by construction, because the
brief is copied from the source and the rule-based analyser. These scores become informative when an LLM is used.

### 6.3 LLM results
Run `python run.py evaluate --llm gemini` with a key set, then copy the `brief_quality` block from
`outputs/evaluation_gemini.json` into this table:

| Metric | Gemini | Notes |
|---|---|---|
| Schema complete rate | _fill in_ | |
| Avg evidence grounding | _fill in_ | share of quotes found verbatim in the source |
| Briefs with unsupported numbers | _fill in_ | possible hallucinated figures, check manually |
| Relevance agreement with rules | _fill in_ | LLM rating vs the independent rule-based rating |

Also hand-score 5 briefs from 1 to 5 on accuracy, completeness, clarity and usefulness of the operational analysis.

## 7. Operational-relevance analysis
Relevance is rated **high** for binding requirements with deadlines, penalties or payment effects; **medium** for
final guidance or notices that need awareness or planning; and **low** for drafts, consultations and reports.
Affected areas: Billing & Reimbursement, Regulatory Compliance, Clinical Operations, IT/Data & Privacy,
Pharmacy & Supply Chain, Workforce, Finance & Grants.

| Detected change (test scenario) | Type | Relevance | Main areas | Why it matters |
|---|---|---|---|---|
| Physician fee schedule: proposed → final, conversion factor 2.1% → 2.8%, telehealth permanent | changed | High | Billing, Compliance, IT | Revenue forecasts and telehealth billing workflows must be updated |
| PM-JAY package rates: dialysis packages added, claims at old rates rejected after 30 days | changed | High | Billing, Compliance | Pre-authorisation templates must change before the deadline, or claims are denied |
| IRDAI cashless circular: insurer bears cost of delays beyond 3 hours, penalty framework | changed | High | Billing, Compliance, Finance | Discharge workflows and turnaround times with insurers need monitoring |
| Mandatory notifiable-disease reporting on IHIP within 24 hours | new | High | Compliance, Clinical, IT | Lab and clinical teams need a reporting process |
| LDT oversight phase-in | new | Medium | Compliance, Supply chain | In-house lab test programmes need a compliance plan within 12 months |
| Information-blocking disincentives | new | Medium | IT/Data | Data-sharing practices and Medicare scores are affected |
| Revised Schedule M timeline (draft) | new | Medium | Compliance, Supply chain | Supplier compliance risk to watch |
| Avian influenza preparedness | new | Medium | Clinical, Supply chain | Isolation capacity and PPE stock should be reviewed |
| Drug shortage notice no longer listed | removed | Medium | Supply chain | Check whether the shortage was resolved or the notice moved |

## 8. Limitations
1. **Synthetic evaluation data:** perfect scores on a small, clean scenario are optimistic. Real pages are noisier.
   Their layouts vary, some content is rendered by JavaScript, and important details are often inside PDFs.
2. **Page-level scraping:** listing pages show titles and short excerpts. The full PDF circular is not read
   unless its link is scraped separately.
3. **Title-based identity:** if an authority renames a notice, it appears as one new item plus one removed item.
4. **LLM risk:** summaries may omit or misstate details. Grounding checks reduce this risk but do not remove it,
   so a human should review high-relevance briefs.
5. **Rule-based relevance** uses keyword heuristics and may misjudge uncommon wording.
6. **Cost and rate limits:** Firecrawl credits and LLM quotas limit how often and how widely the system can crawl.

## 9. Recommendations and future work
* Follow document links and parse PDFs (Firecrawl supports PDF parsing) to brief from the full circular.
* Use fuzzy title matching or embeddings to link renamed items.
* Add email or Slack alerts for high-relevance briefs, and a reviewer sign-off step in the dashboard.
* Use Firecrawl `map`/`crawl` to discover sub-pages, and add state health departments and more payers.
* Build a human-labelled evaluation set from live crawls collected over several weeks.
* Schedule runs with GitHub Actions and keep the Memento database as a versioned artefact.

## 10. Conclusion
HealthPolicy AI shows that Firecrawl collection, a persistent Memento history and grounded LLM summarisation
together form a practical policy-tracking workflow. Memento is what turns repeated crawling into *change tracking*:
the system raises alerts only for content it has not seen before, compares each update with the previous
version, and never alerts twice for the same change. The LLM turns those differences into concise briefs that say
what changed, when, and what a healthcare organization should do about it.

## Appendix A — Reproduce
```bash
pip install -r requirements.txt && copy .env.example .env
python run.py reset
python run.py track --offline cycle1 --baseline-only
python run.py track --offline cycle2 --llm gemini
python run.py evaluate --llm gemini
streamlit run app.py
pytest -q
```

## Appendix B — Disclaimer
Briefs summarise public information and are not legal or regulatory advice. Demo snapshots in `data/fixtures` are
fictional and exist only for testing.
