# Deliverable 1 — Healthcare Policy Information Collection Report

**Project:** HealthPolicy AI: Healthcare Policy Tracker · **Milestone:** M1 (Week 1)
**Tasks:** 1.1 Set up the environment · 1.2 Identify healthcare policy sources · 1.3 Build the Firecrawl collection pipeline

## 1. Problem context
Healthcare organizations must follow policy updates from many separate public bodies: payers and reimbursement
authorities, drug and device regulators, health ministries and global health agencies. Each one publishes on its
own website, in its own format, and at irregular times. Checking them by hand is slow, and it is easy to miss a
small but important edit, for example a revised package rate or a new deadline. M1 builds the automated
collection layer that every later stage depends on.

## 2. Environment setup (Task 1.1)
| Item | Choice |
|---|---|
| Language | Python 3.10+ (tested on 3.13) |
| Collection | `firecrawl-py` v4 (`Firecrawl.scrape`) |
| Config | `config/sources.yaml` (sources), `.env` (API keys, never committed) |
| Storage | SQLite (`data/memento.db`) via the Memento store |
| Version control | Git + GitHub; `.gitignore` excludes `.env`, the database and collected raw data |

```bash
python -m venv .venv && .venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env      # add FIRECRAWL_API_KEY
python run.py collect       # live collection
python run.py collect --offline cycle1   # offline snapshot collection (no key)
```

## 3. Policy sources (Task 1.2)
Selection criteria: (1) official government or regulator site, (2) publicly accessible without login,
(3) a page that lists policy updates over time (newsroom, notices, circulars), (4) relevant to hospital operations,
and (5) coverage of both India and the US plus one global authority.

| ID | Source | Authority | Region | Category |
|---|---|---|---|---|
| cms_newsroom | CMS Newsroom | CMS | US | Reimbursement |
| fda_press | FDA Press Announcements | FDA | US | Drug & device regulation |
| hhs_press | HHS Press Room | HHS | US | Government notification |
| mohfw_notices | MoHFW – What's New | MoHFW | IN | Government notification |
| nha_pmjay | National Health Authority – PM-JAY / ABDM | NHA | IN | Reimbursement |
| cdsco_notices | CDSCO – Public Notices | CDSCO | IN | Drug & device regulation |
| irdai_circulars | IRDAI – Circulars | IRDAI | IN | Reimbursement (health insurance) |
| who_news | WHO News Releases | WHO | Global | Public health guidance |

**Source categories:** `regulatory`, `reimbursement`, `government_notification`, `public_health_guidance`,
`drug_device_regulation`. To add or remove a source, edit the YAML file. No code changes are needed.

## 4. Firecrawl collection pipeline (Task 1.3)
`healthpolicy/collector.py` → `FirecrawlCollector.fetch(source)`:

```python
doc = Firecrawl(api_key=KEY).scrape(
    source.url,
    formats=["markdown"],          # LLM-ready text
    only_main_content=True,        # drop menus/sidebars
    exclude_tags=["nav", "footer", "header", "aside", "script", "style", "form", ".breadcrumb", ".cookie"],
    timeout=60000,
    max_age=0,                     # always fresh content, because we are tracking changes
)
```
* **Reliability:** each source is retried twice with backoff. If a source still fails, it is returned with
  `status="error"` and the error message, so the other sources keep running. Change detection never marks items
  as removed when their page failed to load.
* **Output:** a `RawPage` record (source metadata, markdown, fetch time, mode, status).
* **Offline mode:** `OfflineCollector` reads Firecrawl-style markdown snapshots from `data/fixtures/<cycle>/`.
  This makes the whole system testable without API credits. The snapshots are **synthetic** and clearly labelled.

## 5. Preprocessing (`healthpolicy/preprocess.py`)
1. **Clean:** remove images, banner quotes, horizontal rules and boilerplate lines (cookie notices,
   accessibility links, "skip to content", footer and copyright).
2. **Segment** each page into individual policy records:
   * newsroom pages → one record per `##`/`###` heading block (generic headings such as "Latest Updates" are skipped);
   * listing pages (typical of `.gov.in` sites) → one record per linked notice line;
   * single-document pages → one record for the whole page.
3. **Extract fields:** title, published date (ISO, `July 3, 2026`, `3 July 2026` and `dd/mm/yyyy` formats),
   document link, and plain text (up to 6,000 characters).
4. **Identify:** `item_id = sha1(source_id | normalised title)` stays the same across crawls.
   `content_hash = sha1(normalised title + date + text)` changes whenever the content changes.

**Structured record example** (`data/collected/nha_pmjay.json`):
```json
{
  "item_id": "68e4bf1d0b19660a", "source_id": "nha_pmjay", "authority": "NHA", "region": "IN",
  "category": "reimbursement", "title": "Revision of Health Benefit Package Rates under AB PM-JAY",
  "published": "2026-06-22",
  "text": "Package rates for 120 procedures in oncology and cardiology are revised. Empanelled hospitals must update pre-authorisation requests from the effective date.",
  "content_hash": "b3a4a385…", "fetched_at": "2026-11-22T11:26:38+00:00"
}
```

## 6. Collection results
**Offline baseline crawl (synthetic snapshots, cycle1)** — `outputs/collection_summary.json`

| Source | Category | Region | Status | Chars | Records |
|---|---|---|---|---|---|
| cms_newsroom | reimbursement | US | ok | 882 | 3 |
| fda_press | drug_device_regulation | US | ok | 881 | 3 |
| hhs_press | government_notification | US | ok | 625 | 2 |
| mohfw_notices | government_notification | IN | ok | 877 | 3 |
| nha_pmjay | reimbursement | IN | ok | 658 | 2 |
| cdsco_notices | drug_device_regulation | IN | ok | 613 | 2 |
| irdai_circulars | reimbursement | IN | ok | 595 | 2 |
| who_news | public_health_guidance | GLOBAL | ok | 620 | 2 |
| **Total** | | | **8/8** | | **19** |

Record extraction and date extraction were both 100% accurate against the labelled snapshots. Unit tests
also cover a realistic `.gov.in` listing page with `dd/mm/yyyy` dates and boilerplate.

**Live run:** run `python run.py collect` with a Firecrawl key, then paste the printed table here.
Live pages are longer and noisier, so record counts will vary by site.

## 7. Workflow summary
`sources.yaml → Firecrawl scrape (markdown) → RawPage → clean → segment → PolicyRecord[] → data/collected/*.json → Memento`
