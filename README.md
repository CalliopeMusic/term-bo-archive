# Terminal Boredom Archive

A two-step toolkit for preserving and searching the Terminal Boredom forum
from the Wayback Machine.

---

## How it works

1. **scraper.py** — discovers every archived thread via the Wayback Machine CDX
   API, then crawls and parses each one, storing posts in a local SQLite
   database (`archive.db`).

2. **serve.py** — starts a tiny local web server that serves the search
   interface and talks to the database. Open `http://localhost:8765` in your
   browser and search away.

---

## Requirements

- Python 3.10+
- Two small libraries:

```
pip install requests beautifulsoup4
```

or

```
pip install -r requirements.txt
```

---

## Step 1 — Run the scraper

```bash
python scraper.py
```

This will:
1. Ask the Wayback Machine CDX API for every archived `terminal-boredom.com`
   topic URL (takes a few seconds).
2. Work through each topic, downloading and parsing posts, saving them to
   `archive.db`.
3. Print progress every 50 topics and write a `scraper.log` file.

**The scraper is resumable.** If you stop it (Ctrl-C), just re-run with:

```bash
python scraper.py --skip-discovery
```

It will pick up exactly where it left off without re-scraping anything.

### Options

| Flag | Default | Description |
|------|---------|-------------|
| `--db PATH` | `archive.db` | Output database path |
| `--limit N` | `0` (all) | Only scrape first N topics (good for testing) |
| `--max-pages N` | `0` (all) | Max pages per topic |
| `--skip-discovery` | off | Skip CDX step and use existing DB topics |

### Quick test run

```bash
python scraper.py --limit 50 --max-pages 2
```

This scrapes just 50 topics (2 pages each) so you can verify everything is
working before committing to the full run.

### How long will it take?

The scraper waits ~1.5 seconds between requests to be polite to archive.org.
A rough estimate:

- Terminal Boredom had ~5,000–15,000 topics
- Average ~1.5 pages per topic
- ~2–3 requests per topic at 1.5s each = **~4–9 hours total**

Running it overnight is the easiest approach. You can check progress in
`scraper.log` or by looking at the database:

```bash
python -c "
import sqlite3
conn = sqlite3.connect('archive.db')
print('Posts:',  conn.execute('SELECT COUNT(*) FROM posts').fetchone()[0])
print('Topics done:', conn.execute('SELECT COUNT(*) FROM topics WHERE fully_scraped=1').fetchone()[0])
print('Topics pending:', conn.execute('SELECT COUNT(*) FROM topics WHERE fully_scraped=0').fetchone()[0])
"
```

---

## Step 2 — Search the archive

Once you have some (or all) posts in the database:

```bash
python serve.py
```

Then open **http://localhost:8765** in your browser.

### Features

- **Full-text search** — powered by SQLite FTS5 (fast even with 100k+ posts),
  with keyword highlighting in snippets
- **Thread view** — click "view thread →" on any result to read the full
  conversation
- **Boards tab** — see post/topic counts per board
- **Authors tab** — browse all authors, click one to search their posts
- **Pagination** — page through large result sets

### Options

```bash
python serve.py --db my_archive.db --port 9000
```

---

## File structure

```
terminal-boredom-archive/
├── scraper.py          # Wayback Machine crawler
├── serve.py            # Local search server
├── static/
│   └── index.html      # Search interface
├── requirements.txt
├── README.md
└── archive.db          # Created by scraper (not included)
```

---

## Notes

- The scraper targets the **February 2023** Wayback Machine snapshot by
  default (the most complete one available). You can change `DEFAULT_TS` in
  `scraper.py` if you want a different date.
- Only posts that were successfully archived will be recovered — some threads
  or pages may be missing from the Wayback Machine entirely.
- This is for **personal, non-commercial archival/research** use only.
