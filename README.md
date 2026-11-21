# BigQuery Release Radar

**Automated RSS Feed Parser, Structured Cloud Release Monitor, and Digest Generator**

![Python](https://img.shields.io/badge/Python-3.x-blue?style=flat-square&logo=python)
![Flask](https://img.shields.io/badge/Flask-Web%20Framework-000000?style=flat-square&logo=flask)
![BeautifulSoup](https://img.shields.io/badge/BeautifulSoup-HTML%20Parser-darkgreen?style=flat-square)
![Feedparser](https://img.shields.io/badge/Feedparser-XML%20Ingestion-orange?style=flat-square)
![Vanilla JS](https://img.shields.io/badge/JavaScript-ES6+-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

---

## Overview

**BigQuery Release Radar** is a Python Flask web application and ingestion service that transforms unstructured Google Cloud BigQuery release notes into an interactive, categorized, and searchable developer dashboard.

### The Problem
Google Cloud publishes official BigQuery release notes through an Atom XML feed. However, each RSS entry groups an entire day's heterogeneous updates—spanning new feature launches, breaking changes, critical bug fixes, deprecations, and regional announcements—into a single composite HTML blob. This makes it challenging for data engineers, DevOps specialists, and cloud architects to quickly identify updates relevant to their workflows.

### The Solution
BigQuery Release Radar ingests the upstream XML feed, extracts the CDATA HTML payload, and executes an intelligent HTML parsing algorithm that segments monolithic daily entries into discrete, typed release items. Users can filter updates by semantic category, search release content in real time, and compile custom release digests for distribution.

---

## System Architecture

```text
 ┌──────────────────────────────────────────────────┐
 │ Official BigQuery Release Notes (Atom/XML Feed)   │
 └────────────────────────┬─────────────────────────┘
                          │ HTTP GET
                          ▼
 ┌──────────────────────────────────────────────────┐
 │ Python Flask Application (app.py)                │
 │                                                  │
 │  1. Ingestion: feedparser consumes XML feed      │
 │  2. Parsing: BeautifulSoup DOM traversal         │
 │     - Identifies <h3> / <h4> heading tags        │
 │     - Segments composite entries into updates    │
 │     - Normalizes category tags & clean text      │
 │  3. Endpoints:                                   │
 │     - GET /          -> Server-rendered UI       │
 │     - GET /api/feed  -> Structured JSON API      │
 └───────────┬──────────────────────────┬───────────┘
             │ JSON / HTML              │ JSON Payload
             ▼                          ▼
 ┌───────────────────────┐  ┌───────────────────────┐
 │ Interactive Dashboard │  │ REST API Consumers   │
 │ - Real-time Search    │  └───────────────────────┘
 │ - Category Badges     │
 │ - Multi-select Digest │
 │ - Twitter Composer    │
 └───────────────────────┘
```

---

## Key Features

- **Automated Feed Parsing & Normalization:** Employs `feedparser` to manage XML namespaces and character encodings, combined with `BeautifulSoup4` to segment composite daily entries into structured individual cards.
- **Categorization Engine:** Classifies release items into distinct categories (**Feature**, **Fix**, **Change**, **Announcement**, **Notice**) with dynamic count badges.
- **Real-Time Client-Side Filtering:** Instant search filtering across release titles, descriptions, and category tags with zero latency.
- **Interactive Social Digest Composer:** 
  - Generates single-update announcement drafts or multi-item bulleted digest summaries.
  - Features a live character counter adhering to Twitter/X constraints by accurately budgeting **23 characters** for `t.co` URL wrapping.
- **Light & Dark Theme Engine:** Modern responsive interface with persistent theme selection saved in browser `localStorage`.
- **Unit Test Coverage:** Automated test suite (`test_parser.py`) verifying feed segmenting, heading extraction, and fallback parsing.

---

## Project Structure

```text
BigQuery-Release-Radar/
├── app.py                 # Core Flask backend, feed ingestion & BeautifulSoup parser
├── test_parser.py         # Unit test suite verifying parsing logic and edge cases
├── requirements.txt       # Production dependencies
├── templates/
│   └── index.html         # Jinja2 dashboard template, modals, and filter controls
├── static/
│   ├── css/
│   │   └── style.css      # Custom dark/light mode styles, layout, and components
│   └── js/
│       └── app.js         # Client-side filtering, DOM updates, and tweet generator
├── .gitignore
└── README.md
```

---

## Technical Implementation Details

### HTML Feed Partitioning Algorithm
BigQuery release notes bundle multiple updates under one date entry using consecutive heading tags. The parser in `app.py` traverses child nodes sequentially:
```python
def parse_entry_content(content_html):
    soup = BeautifulSoup(content_html, 'html.parser')
    updates = []
    current_category = "General"
    current_blocks = []

    for element in soup.contents:
        if isinstance(element, str) and not element.strip():
            continue
        if element.name in ['h3', 'h4']:
            if current_blocks:
                updates.append({
                    "category": current_category,
                    "html_content": "".join(str(e) for e in current_blocks).strip(),
                    "text_content": BeautifulSoup("".join(str(e) for e in current_blocks), 'html.parser').get_text(separator=' ').strip()
                })
                current_blocks = []
            current_category = normalize_category(element.get_text(strip=True))
        else:
            current_blocks.append(element)
    ...
```

### URL Budgeting in Social Composer
Standard length calculations fail on Twitter/X because all HTTP/HTTPS links are automatically wrapped in 23-character `t.co` short links. The composer logic calculates available character limits dynamically:
$$\text{Effective Length} = \text{Text Characters} + (\text{URL Count} \times 23)$$

---

## API Endpoints

### 1. Web Dashboard
- **Route:** `GET /`
- **Description:** Renders the main dashboard with parsed releases, live filter counts, and tweet modal.

### 2. Structured Feed API
- **Route:** `GET /api/feed`
- **Description:** Returns JSON representation of the latest parsed release entries.
- **Response Sample:**
  ```json
  {
    "status": "success",
    "total_updates": 48,
    "categories": {
      "Feature": 22,
      "Fix": 14,
      "Change": 8,
      "Notice": 4
    },
    "entries": [
      {
        "date": "November 14, 2026",
        "category": "Feature",
        "title": "Continuous query export support",
        "content_html": "<p>BigQuery now supports continuous query exports...</p>",
        "link": "https://cloud.google.com/bigquery/docs/release-notes#November_14_2026"
      }
    ]
  }
  ```

---

## Getting Started

### Prerequisites
- Python 3.9 or higher
- Git

### 1. Clone the Repository
```bash
git clone https://github.com/Skand203/BigQuery-Release-Radar.git
cd BigQuery-Release-Radar
```

### 2. Set Up a Virtual Environment
```bash
# On Linux/macOS
python3 -m venv venv
source venv/bin/activate

# On Windows
python -m venv venv
venv\Scripts\activate
```

### 3. Install Dependencies
```bash
pip install -r requirements.txt
```

### 4. Run Automated Tests
Execute the unit test suite to verify parsing logic:
```bash
python -m unittest test_parser.py
```

### 5. Launch the Application
```bash
python app.py
```
Open [http://127.0.0.1:5000](http://127.0.0.1:5000) in your web browser.

---

## Future Enhancements

- [ ] Scheduled background feed polling with **Redis** caching to reduce upstream network calls.
- [ ] Automated Slack / Discord webhook notifications when high-priority breaking changes are detected.
- [ ] Export releases as periodic markdown or JSON digests for engineering changelogs.
- [ ] Docker containerization with multi-stage build for production deployment.

---

## License

This project is licensed under the [MIT License](LICENSE).
