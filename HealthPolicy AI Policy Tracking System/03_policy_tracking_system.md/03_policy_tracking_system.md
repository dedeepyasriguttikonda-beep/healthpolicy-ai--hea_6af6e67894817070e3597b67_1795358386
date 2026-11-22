# Deliverable 3 — HealthPolicy AI Policy Tracking System

**Milestones:** M2.4–2.5, M4 · **Tasks:** end-to-end workflow, tracking interface, monitoring pipeline

## 1. End-to-end workflow (`healthpolicy/pipeline.py → run_cycle`)
1. **Collect:** Firecrawl scrapes every enabled source (or the offline snapshots).
2. **Preprocess:** pages are cleaned and split into `PolicyRecord`s.
3. **Detect:** records are compared with Memento history (new / changed / cosmetic / unchanged / removed).
4. **Deduplicate:** a brief is skipped when Memento already holds a brief for that exact version
   (`is_reviewed(item_id, content_hash)`), so the same change never triggers two alerts.
5. **Generate:** the LLM writes a brief for each actionable change. Each brief is then validated and scored for grounding.
6. **Remember:** new versions are written to Memento, cosmetic items are touched, and removed items are marked removed.
7. **Report:** a digest (Markdown) and briefs (JSON) are written to `outputs/`, and the run is logged.

## 2. Memento — persistent policy history
`healthpolicy/memento_store.py` (SQLite, `data/memento.db`):

| Table | Holds |
|---|---|
| `runs` | each tracking cycle: mode, time, counts of new / changed / removed / unchanged |
| `policies` | latest known state per item: first_seen, last_seen, last_changed, status, version_count |
| `versions` | every distinct content version (append-only), giving the full history of a policy |
| `briefs` | each generated brief, linked to the version that triggered it, with provider and model |
| `reviews` | review status per item (brief_generated / reviewed / dismissed) |

API: `remember`, `recall`, `known_items`, `history`, `is_reviewed`, `save_brief`, `mark_removed`, `related_context`.

**Optional knowledge-graph mirror:** with `MEMENTO_BACKEND=memento-memory` (requires `pip install memento-memory`),
every policy fact and brief is also ingested into the `memento-memory` bitemporal knowledge graph.
`related_context()` then recalls related past policies and adds them to the LLM prompt as context.
SQLite stays the source of truth, so a graph error never stops the pipeline.

## 3. Interfaces
**CLI** (`run.py`): `track`, `collect`, `history`, `evaluate`, `reset`. Options: `--offline CYCLE`,
`--sources`, `--llm`, `--baseline-only`.

**Streamlit dashboard** (`streamlit run app.py`):
* **Sidebar:** choose live or offline mode, LLM provider and sources; run a cycle; reset history.
* **Update briefs:** KPI tiles (briefs, high relevance, new, changed); filters for relevance, region and change
  type; search; expandable brief cards with grounding scores; digest download.
* **Policy history (Memento):** every tracked policy with status and version count, plus a per-policy version timeline.
* **Runs:** the log of tracking cycles. **Sources:** the source registry. **Evaluation:** metric results.

## 4. Demonstration (offline, synthetic snapshots)
```
$ python run.py track --offline cycle1 --baseline-only
  pages 8 · records 19 · new 19            (history built, no briefs)
$ python run.py track --offline cycle2
  pages 8 · records 23 · new 5 · changed 3 · removed 1 · unchanged 15 · briefs 9
$ python run.py track --offline cycle2     (same data again)
  changed 0 · briefs 0                     (Memento suppresses duplicates)
$ python run.py history
  run 2 offline:cycle2 new=5 changed=3 removed=1 unchanged=15
  [active ] v2 CMS  Proposed Physician Fee Schedule Update for Next Calendar Year
```
The cosmetic-only edit (extra spaces in the WHO patient-safety item) was correctly classified as unchanged.

## 5. Running it on a schedule (monitoring, M4.1)
The live tracker is designed to run regularly. Each run reports only what changed since the previous run.
* Windows Task Scheduler: `python C:\path\healthpolicy-ai\run.py track` daily.
* Linux/macOS cron: `0 8 * * * cd /path/healthpolicy-ai && .venv/bin/python run.py track`.
* GitHub Actions: a scheduled workflow with the API keys stored as repository secrets.
