---
name: paper-downloader
description: >
  Bulk academic paper collection and PDF downloading system.
  Searches OpenAlex for papers matching your keywords, filters by citation count,
  downloads PDFs from 8+ sources with automatic fallback, and monitors all processes
  with a supervisor that auto-restarts crashed workers and sends email progress reports.
  Use this skill when setting up, operating, debugging, or extending the paper download pipeline.
---

# Paper Downloader Skill

## Overview

This skill helps you build and operate a bulk academic paper collection system that:
1. **Searches** OpenAlex (250M papers, free API) for papers matching your keywords
2. **Filters** by citation count to keep only high-quality papers
3. **Downloads** PDFs from 8+ sources with automatic fallback (open access, preprints, institutional proxies)
4. **Monitors** all processes with a supervisor that auto-restarts crashed workers and emails progress reports

---

## System Architecture

```
corpus_search/
├── config.py       Keywords, citation thresholds, DB paths
├── search.py       OpenAlex API queries → SQLite metadata
├── db.py           Database schema, upsert, stats
└── run.py          CLI entry: search | download | status | topics

retry_download.py   Multi-source PDF downloader (8 sources, circuit breakers)
supervisor.py       Process watchdog + email reports every 1-2h
collect_topic.py    Topic-specific collection from OpenAlex
download_topic.py   Topic-specific download + Excel export
```

### Two Databases

| Database | Purpose | Location (configurable) |
|----------|---------|----------|
| `corpus` | High-quality papers filtered by citation count | `/your/path/corpus/_corpus.db` |
| `topic` | Topic-specific papers across research domains | `/your/path/topic_watcher/papers.db` |

---

## Database Schema

```sql
CREATE TABLE papers (
    doi         TEXT PRIMARY KEY,
    s2_id       TEXT DEFAULT '',
    title       TEXT,
    year        INTEGER,
    venue       TEXT DEFAULT '',
    citations   INTEGER DEFAULT 0,
    abstract    TEXT DEFAULT '',
    pdf_url     TEXT DEFAULT '',
    passes_filter INTEGER DEFAULT 0,   -- citation threshold gate
    downloaded  INTEGER DEFAULT 0,
    pdf_path    TEXT DEFAULT '',
    found_by    TEXT DEFAULT '',        -- which keyword found this paper
    added_date  TEXT
);

CREATE TABLE search_log (
    query TEXT PRIMARY KEY,
    fetched_at TEXT
);
```

**Citation filter thresholds** (configurable in `config.py`):
- Papers from 2020+: ≥10 citations
- Papers from 2010-2019: ≥20 citations
- Papers before 2010: ≥30 citations

---

## Download Source Priority

`retry_download.py` tries sources in this order per paper:

1. **OpenAlex OA locations** — free, legal, high success rate
2. **Semantic Scholar** — open access links from S2 API
3. **arXiv / ChemRxiv** — preprints (DOI prefix: 10.48550, 10.26434)
4. **Unpaywall** — legal OA PDF finder (requires `mailto` email)
5. **Europe PMC / PubMed Central** — biomedical OA
6. **CORE** — aggregator for institutional repositories
7. **Publisher direct** — curl_cffi with Chrome fingerprint spoofing
8. **Institutional proxy (EZproxy)** — university library access (requires session cookie)
9. **Sci-Hub mirrors** — ⚠️ CAPTCHA-blocked as of 2026, treat as permanently failed
10. **LibGen** — unstable DNS, useful for older papers only

### PDF Validation

Every download is validated before saving:
```python
def _ok(content):
    return (isinstance(content, bytes) 
            and len(content) > 8000 
            and b"%PDF" in content[:8])
```

---

## Circuit Breakers

Each source has an independent circuit breaker to prevent hammering a broken endpoint:

| Source | Failure threshold | Cooldown |
|--------|------------------|----------|
| Sci-Hub | 3 consecutive failures | 1800 s |
| EZproxy (institutional) | 3 | 3600 s |
| LibGen | 5 | 900 s |
| Publisher direct | 10 | 600 s |
| OA sources (arXiv etc.) | 40 | 180 s |

**Global circuit breaker**: if ≥90% of last 20 downloads fail → pause 60 s (VPN disconnect detection).

---

## Publisher-Specific Notes

| Publisher | DOI prefix | Strategy | Notes |
|-----------|-----------|---------|-------|
| ACS | 10.1021/ | Direct + proxy | High success rate |
| Springer/Nature | 10.1007/, 10.1038/ | Direct + proxy | Most papers accessible |
| RSC | 10.1039/ | **Custom URL** | DOI suffix encodes year + journal — must reverse-engineer (see below) |
| Frontiers, PLOS, MDPI | 10.3389/, 10.1371/, 10.3390/ | Direct | Open access, no auth needed |
| AIP/APS, IOP, Taylor & Francis, Oxford | 10.1063/, 10.1088/, 10.1080/, 10.1093/ | EZproxy only | VPN direct connect returns HTML, not PDF |
| **Elsevier** | 10.1016/ | **Browser cookie required** | IP auth insufficient; need JSESSIONID cookie from ScienceDirect |
| **Wiley** | 10.1002/, 10.1111/ | **Browser cookie required** | Same as Elsevier |
| Science/AAAS | 10.1126/ | EZproxy | Requires institutional access |

### RSC URL Reverse Engineering

RSC PDF URLs are **not** derived directly from DOI. The format is:
```
https://pubs.rsc.org/en/content/articlepdf/{year}/{journal}/{article_id}
```

The DOI suffix encodes year and journal:
- First character: decade (`a`=1990s, `b`=2000s, `c`=2010s, `d`=2020s, `e`=2030s)
- Second character: year within decade (digit)
- Next letters: journal abbreviation (e.g., `ee` = Energy & Environmental Science)

```python
def _rsc_pdf_url(doi):
    suffix = doi.split('/')[-1].lower()
    decade_map = {'a': 1990, 'b': 2000, 'c': 2010, 'd': 2020, 'e': 2030}
    if not suffix or suffix[0] not in decade_map:
        return None
    try:
        year = decade_map[suffix[0]] + int(suffix[1])
    except (ValueError, IndexError):
        return None
    m = re.match(r'[a-z]\d([a-z]+)', suffix)
    if not m:
        return None
    return f"https://pubs.rsc.org/en/content/articlepdf/{year}/{m.group(1)}/{suffix}"

# Example: 10.1039/d2ee01234a → .../2022/ee/d2ee01234a
```

### Elsevier / Wiley Cookie Injection

These publishers require browser session cookies even with institutional IP access:

1. Connect to institutional VPN
2. Open browser, navigate to ScienceDirect / Wiley and confirm access
3. Export cookies for `sciencedirect.com` or `onlinelibrary.wiley.com` using Cookie Editor extension
4. Add to `config.json`:
```json
{
  "elsevier_cookies": {
    "JSESSIONID": "...",
    "ScienceDirectUser": "...",
    "MAID": "..."
  }
}
```
5. **Cookie lifetime**: 24–48 hours. Must refresh regularly.

### EZproxy (Institutional Proxy)

For publishers that require IP authentication through a university proxy:

1. Connect to university VPN (triggers IP-based auth on EZproxy)
2. Export cookies from your institution's EZproxy domain (e.g. `ezproxy1.lib.youruniversity.ac.uk`) — **not** the main university domain
3. Save as `session.json`:
```json
{
  "cookies": [
    {"name": "ezproxy", "value": "SESSION_TOKEN", "domain": "ezproxy1.lib.youruniversity.ac.uk", "path": "/"}
  ]
}
```

**Common failure**: account suspended after high-frequency automated requests → wait 24h, then re-export cookies.

---

## Setup Guide

### 1. Install dependencies

```bash
pip install requests openpyxl curl_cffi
```

`curl_cffi` enables Chrome browser fingerprint spoofing (needed for some publishers). Install separately if compilation fails.

### 2. Configure `config.py`

```python
# corpus_search/config.py

BASE_DIR = Path("/your/path/to/papers/corpus")   # change to your path
DB_PATH  = BASE_DIR / "_corpus.db"
OA_EMAIL = "your@email.com"                       # OpenAlex polite pool: 10 req/s

def citation_threshold(year):
    if year >= 2020: return 10
    if year >= 2010: return 20
    return 30

SEARCH_QUERIES = [
    "your keyword A",
    "your keyword B",
    # ... add your research keywords
]
```

### 3. Configure `config.json` (supervisor / email)

```json
{
  "email": {
    "sender": "your@gmail.com",
    "recipient": "your@gmail.com",
    "smtp_server": "smtp.gmail.com",
    "smtp_port": 587,
    "app_password": "xxxx xxxx xxxx xxxx"
  }
}
```

**Gmail app password**: Google Account → Security → 2-Step Verification → App passwords

### 4. Run metadata search

```powershell
# Windows (set UTF-8 encoding first)
$env:PYTHONUTF8 = '1'
Set-Location corpus_search
python run.py search
```

This queries OpenAlex for all keywords in `SEARCH_QUERIES`, storing metadata in `_corpus.db`. Takes 30–120 minutes depending on keyword count.

### 5. Start the supervisor

```powershell
$env:PYTHONUTF8 = '1'
Set-Location ..    # back to paper_downloader/
python supervisor.py
```

The supervisor auto-launches:
- `retry_download.py --db corpus --threads 8`
- `retry_download.py --db topic --threads 10`  
- `topic_watcher.py --full`

And sends email progress reports every 2 hours.

---

## Common Operations

### Check current status

```powershell
$env:PYTHONUTF8='1'; python _check_status.py
```

Or query SQLite directly:
```python
import sqlite3
conn = sqlite3.connect('path/to/_corpus.db')
total     = conn.execute('SELECT COUNT(*) FROM papers').fetchone()[0]
passes    = conn.execute('SELECT COUNT(*) FROM papers WHERE passes_filter=1').fetchone()[0]
downloaded = conn.execute('SELECT COUNT(*) FROM papers WHERE downloaded=1').fetchone()[0]
pending   = conn.execute('SELECT COUNT(*) FROM papers WHERE passes_filter=1 AND downloaded=0 AND doi IS NOT NULL').fetchone()[0]
```

### Add a new research topic

```powershell
$env:PYTHONUTF8='1'; python collect_topic.py --topic "Your_Topic_Name"
```

If a topic was already collected and you want to re-collect:
```python
import sqlite3
c = sqlite3.connect('/your/path/topic_watcher/papers.db')
c.execute("DELETE FROM topic_log WHERE topic=?", ('Your_Topic_Name',))
c.commit()
```

### Download topic papers with citation filter

```powershell
$env:PYTHONUTF8='1'; python download_topic.py --topic Your_Topic --min_cited 10 --threads 8
```

### Force immediate status email

```powershell
$env:PYTHONUTF8='1'; python _send_now.py
```

### Stop all workers gracefully

```powershell
Get-Process python | ForEach-Object {
    $cmd = (Get-CimInstance Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine
    if ($cmd -match "supervisor|retry_download|topic_watcher") {
        Stop-Process -Id $_.Id -Force
    }
}
```

### Restart everything

```powershell
$env:PYTHONUTF8='1'
Set-Location "C:\path\to\paper_downloader"
Start-Process python "supervisor.py" -WorkingDirectory (Get-Location) -WindowStyle Minimized
```

---

## Supervisor Configuration

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `CHECK_S` | 1800 s | Health check interval (30 min) |
| `EMAIL_S` | 3600 s | Email report interval (1 h) |
| `SYNC_S` | 10800 s | PDF sync interval (3 h) |
| `MAX_PROCS` | 6 | Max worker processes |
| `MAX_MEM_MB` | 3000 | Per-process memory limit; exceeded → kill + restart |

**Note**: Always pass `local_hostname="localhost"` to `smtplib.SMTP()` if your machine hostname contains non-ASCII characters (common on Chinese/Japanese Windows), otherwise SMTP EHLO fails.

---

## Known Issues & Fixes

### SQLite "database is locked"

Workers hold a WAL write lock continuously. Direct `sqlite3.connect()` from another script will timeout.

**Fix**: Stop the worker first (`Stop-Process`), do your operation, then supervisor restarts it within 30 min.

**Prevention**: All connections use:
```python
conn = sqlite3.connect(db_path, timeout=60)
conn.execute("PRAGMA journal_mode=WAL")
conn.execute("PRAGMA busy_timeout=60000")
```

### Sci-Hub permanently blocked

As of 2025, all Sci-Hub mirrors return CAPTCHA pages for automated requests. Remove from your sources list to avoid wasted retries.

### LibGen DNS failures

LibGen mirrors have frequent DNS resolution failures. Only useful for pre-2020 papers. Set failure threshold to 5 with 900s cooldown.

### IOP / Oxford / T&F require EZproxy

These publishers reject direct requests even with institutional IP. Must route through EZproxy:
```python
proxy_url = f"https://ezproxy1.lib.youruniversity.ac.uk/login?url={original_url}"
```

### `topic_log` blocks re-collection

`collect_topic.py --all` skips topics already in `topic_log`. Delete the row to force re-collection.

### OpenAlex abstract reconstruction

OpenAlex stores abstracts as inverted index (word → list of positions). Reconstruct with:
```python
def _reconstruct_abstract(inv_index):
    if not inv_index:
        return ""
    max_pos = max(pos for positions in inv_index.values() for pos in positions)
    words = [''] * (max_pos + 1)
    for word, positions in inv_index.items():
        for pos in positions:
            words[pos] = word
    return ' '.join(words)
```

---

## OpenAlex API Notes

- **Free, no API key required** — add `mailto=your@email.com` to get polite pool (10 req/s, 100k req/day)
- Rate limit 429 → back off 60×attempt seconds, retry up to 4 times
- `per-page` max: 200 results per request; use `cursor=*` for pagination
- Filter syntax: `filter=title.search:keyword,publication_year:>2009,cited_by_count:>9`
- OA location field: `open_access.oa_locations[].pdf_url` — direct PDF links when available

```python
params = {
    "search": keyword,
    "filter": "cited_by_count:>9",
    "per-page": 200,
    "cursor": "*",
    "select": "id,doi,title,publication_year,cited_by_count,abstract_inverted_index,open_access,primary_location",
    "mailto": OA_EMAIL,
}
```

---

## Performance Benchmarks (real-world)

| Database | Papers collected | PDFs downloaded | Time |
|----------|-----------------|-----------------|------|
| Domain A (broad field) | ~70,000 total, ~48,000 pass filter | ~40,000 (83%) | ~3 weeks, 8 threads |
| Domain B (narrow topic) | ~9,000 | ~4,300 (48%) | ~1 week, 3 threads |

**Bottlenecks**:
- Elsevier (10.1016/): largest single blocker — requires browser session cookie
- Wiley (10.1002/): same as Elsevier
- High-throughput open access: ~300–500 PDFs/hour with 8 threads

---

## Skill Usage

When this skill is active, Claude can help you with:

- **Setting up** the system from scratch (choosing keywords, configuring paths)
- **Monitoring** download progress and interpreting status reports
- **Debugging** when downloads stall (check circuit breakers, VPN status, cookie expiry)
- **Expanding** the corpus with new keywords or research topics
- **Querying** the database to find papers on specific topics
- **Troubleshooting** publisher-specific access issues

### Example prompts after loading this skill:

```
"Check the download status of my corpus database"
"Add a new keyword to my search queries"
"Why are Elsevier papers not downloading?"
"How do I query the database for papers on a specific topic?"
"How do I re-collect a topic that was already marked done?"
```
