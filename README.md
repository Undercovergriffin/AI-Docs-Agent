# AI Document Agent

Parses `.docx`, `.txt`, `.ppt`/`.pptx`, and `.xlsx` files, stores a
whitespace-stripped, base64-encoded copy of the extracted content in
Postgres, and returns an AI-generated summary using Groq's free API.

## How it works

```
file path
   │
   ▼
parsers/*  ──────────►  raw_text
   │                        │
   │                        ▼
   │                 summarizer.py ──► Groq API ──► summary
   ▼
processor.py
 (strip whitespace, base64-encode)
   │
   ▼
db.py ──► Postgres "documents" table
              (file_name, file_type, content_base64, summary, ...)
```

Note: the summary is generated from the **original** extracted text
(spaces intact), because an LLM can't read text with every space
removed. The whitespace-stripped + base64 version is what's stored in
the DB, per the spec — you can always reverse it with
`processor.from_base64()` if you need the cleaned text back.

## 1. Project setup

```bash
git clone <this-folder>   # or just copy the ai_doc_agent/ directory
cd ai_doc_agent
python -m venv venv
source venv/bin/activate     # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

Then edit `.env` and fill in your Postgres and Groq credentials (see
below).

## 2. Postgres setup (via pgAdmin)

1. Open **pgAdmin**, connect to your Postgres server.
2. Right-click **Databases → Create → Database**, name it e.g.
   `doc_agent_db` (matches `PG_DATABASE` in `.env`).
3. Open the **Query Tool** on that database and run `schema.sql` from
   this project (or just skip this — `main.py` calls `init_db()` on
   startup and will create the `documents` table automatically if it
   doesn't exist).
4. Fill in `.env`:
   ```
   PG_HOST=localhost
   PG_PORT=5432
   PG_DATABASE=doc_agent_db
   PG_USER=postgres
   PG_PASSWORD=<your postgres password>
   ```

## 3. Groq free API setup

Groq's developer tier is free with no credit card required, gated by
rate limits (as of writing: ~30 requests/min). Steps:

1. Go to **https://console.groq.com** and sign up (email, Google, or
   GitHub — no card needed).
2. In the left sidebar, click **API Keys → Create API Key**, give it a
   name, and copy the key immediately (it's only shown once). It will
   look like `gsk_...`.
3. Install the Groq Python SDK (already in `requirements.txt`, or on
   its own):
   ```bash
   pip install groq
   ```
4. Put the key in `.env`:
   ```
   GROQ_API_KEY=gsk_your_key_here
   GROQ_MODEL=llama-3.3-70b-versatile
   ```
5. Groq's available free models can change over time — check
   **console.groq.com → Playground** or the `/models` endpoint for the
   current list, and swap `GROQ_MODEL` if `llama-3.3-70b-versatile`
   is ever retired.

## 4. Run it

```bash
python main.py "/path/to/some/document.docx"
```

or interactively:

```bash
python main.py
File path> /path/to/report.xlsx
```

Each run:
- Extracts and prints a summary in the terminal
- Inserts one row into the `documents` table with the base64-encoded,
  whitespace-stripped content plus the summary

## 5. Inspect stored data (pgAdmin)

In pgAdmin's Query Tool:

```sql
SELECT id, file_name, file_type, created_at, summary
FROM documents
ORDER BY created_at DESC;
```

To decode a stored `content_base64` value back to text, use Python:

```python
from processor import from_base64
print(from_base64(row["content_base64"]))
```

(Remember: this returns the content with *no* whitespace — the
original spacing was intentionally discarded before storage.)

## File overview

| File                    | Purpose                                      |
|-------------------------|-----------------------------------------------|
| `config.py`              | Loads env vars, validates required keys       |
| `db.py`                  | Postgres connection, schema init, CRUD        |
| `parsers/docx_parser.py` | `.docx` text extraction (python-docx)         |
| `parsers/txt_parser.py`  | `.txt` reading                                |
| `parsers/pptx_parser.py` | `.ppt`/`.pptx` text extraction (python-pptx)  |
| `parsers/xlsx_parser.py` | `.xlsx` cell extraction (openpyxl)            |
| `processor.py`           | Whitespace stripping + base64 encode/decode   |
| `summarizer.py`          | Groq chat-completion call                     |
| `agent.py`               | Orchestrates parse → process → store → summarize |
| `main.py`                | CLI entry point                               |
| `schema.sql`             | Manual DDL for pgAdmin                        |

## Known limitations

- Legacy binary `.ppt` files aren't supported by `python-pptx` — only
  OOXML `.pptx`. Convert `.ppt` → `.pptx` first (PowerPoint/LibreOffice
  "Save As").
- Very large documents are truncated to ~60k characters before being
  sent to Groq (`summarizer.py: _MAX_INPUT_CHARS`) to stay within
  free-tier context/rate limits — raise this if you're on a paid tier
  with a larger-context model.
