# IGNA Competitive Product Research Agent

An AI-powered product research agent that accepts a natural-language query and returns ranked, deduplicated listings from eBay, Best Buy, and Amazon — with an AI-generated summary and a concrete recommendation.

**Stack:** FastAPI · Azure OpenAI (gpt-4.1-mini) · Playwright · React/Vite · Pydantic

---

## How It Works

1. User submits a free-text query (e.g. _"best wireless headphones under $200"_)
2. Azure OpenAI parses it into structured criteria (brand, price, RAM, storage, condition, target sites)
3. A second LLM call checks whether the query is realistically fulfillable at that price
4. Playwright scrapes eBay, Best Buy, and Amazon sequentially
5. Results are deduplicated, strictly filtered, and scored — with a soft fallback if strict matches are too few
6. The top product is recommended and a short AI summary is generated
7. JSON and CSV reports are saved to `data/` and returned in the API response

---

## Codebase Overview

```
igna_agent_poc2/
├── api/
│   ├── app.py              # FastAPI bootstrap, CORS, Windows event loop policy
│   ├── search.py           # POST /search · POST /search/{id}/cancel
│   ├── health.py           # GET /health
│   └── report.py           # GET /report/{filename}
├── core/
│   ├── research_flow.py    # Main pipeline — wires every stage together
│   ├── query_parser.py     # NL → SearchCriteria via LLM (regex fallback)
│   ├── query_validator.py  # Feasibility check via LLM
│   ├── product_filter.py   # Dedup, strict filter, soft fallback, scoring
│   ├── product_recommender.py
│   ├── summary_generator.py
│   ├── report_writer.py    # CSV / JSON / Rich terminal table output
│   └── cancellation.py     # Thread-safe CancelContext + search registry
├── integrations/
│   ├── scraper_runner.py   # Runs the three scrapers sequentially
│   ├── ebay_scraper.py
│   ├── bestbuy_scraper.py
│   ├── amazon_scraper.py   # Uses persistent browser profile for bot avoidance
│   ├── browser.py          # Stealth Playwright helpers, human delays
│   ├── openai_client.py    # Azure OpenAI singleton
│   └── scraper_support.py
├── models/                 # Pydantic models: Product, SearchCriteria, SearchRequest, SearchResponse
├── scripts/
│   └── seed_amazon_profile.py  # One-time: seeds Amazon delivery ZIP into browser profile
├── frontend/               # React 18 + Vite UI
├── data/                   # Reports and browser profile (gitignored)
└── main.py                 # CLI entry point for interactive use
```

### Key design notes

- **Filtering is two-tier.** Strict filtering runs first (brand, model, price, condition, specs). If it returns fewer than 5 results, a soft pass by query-token relevance fills the gap.
- **Scoring weights:** brand match +20, exact model match +35, per-token match +5, rating bonus up to +10, accessory penalty −50.
- **Amazon uses a persistent browser profile** (`data/browser_profile/`) to survive session checks. It must be seeded once before first use.
- **Scrapers run sequentially**, not in parallel, to avoid browser session conflicts.
- **Playwright runs in `asyncio.to_thread()`** so it doesn't block the FastAPI event loop. Windows requires `ProactorEventLoop` (set in `api/app.py`).
- **Cancellation** is thread-safe via `CancelContext` (checked at 6 pipeline points). `POST /search/{id}/cancel` closes any open browser immediately and returns HTTP 499.
- **All LLM calls fail gracefully** — the query parser falls back to regex, the validator assumes feasible, and the summary generator uses a hardcoded fallback.

---

## Setup

**Prerequisites:** Python 3.9+, Azure OpenAI credentials

```bash
pip install -r requirements.txt
playwright install chromium
```

**`.env` in project root:**

```env
AZURE_OPENAI_API_KEY=your_key_here
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com
AZURE_OPENAI_API_VERSION=2024-05-01-preview
AZURE_OPENAI_DEPLOYMENT=gpt-4.1-mini
```

**`frontend/.env`:**

```env
VITE_API_BASE_URL=http://127.0.0.1:8000
```

**Seed the Amazon browser profile (first time only):**

```bash
python scripts/seed_amazon_profile.py
```

A Chromium window opens. Go to Amazon, click "Deliver to", enter your ZIP, apply, then close the window.

**Start the backend:**

```bash
uvicorn api.app:app --reload
```

**Start the frontend (optional):**

```bash
cd frontend && npm install && npm run dev
```

---

## API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/search` | Run a product search |
| `POST` | `/search/{id}/cancel` | Cancel an in-flight search |
| `GET` | `/report/{filename}` | Fetch a saved report |
| `GET` | `/health` | Azure OpenAI connectivity check |

**Example request:**

```bash
curl -X POST http://127.0.0.1:8000/search \
  -H "Content-Type: application/json" \
  -d '{"query": "travel camera under 1000", "max_results_per_site": 5}'
```

Each successful search writes `data/report_<timestamp>.csv` and `data/report_<timestamp>.json`.

---

## Troubleshooting

- **Amazon returns no results** — re-run `seed_amazon_profile.py` to reset the delivery ZIP.
- **Scrapers return 0 results** — check `data/debug_*.png` for CAPTCHA or blocked-page screenshots.
- **Azure OpenAI errors** — `GET /health` reports connectivity status and the configured deployment name.
