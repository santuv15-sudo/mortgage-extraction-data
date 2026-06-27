# mortgage-extraction-data

Source PDFs and parsed JSON for the Fannie Mae **Selling Guide** extraction pipeline used by [`mortgage-rag`](../mortgage-rag/) (backend: [mortgage-rag-backend](https://github.com/santuv15-sudo/mortgage-rag-backend)).

This repo holds **data only** — no application code. Parsing, chunking, and embedding live in `mortgage-rag`.

## Layout

```text
sources/pdf/                    # Source PDFs (gitignored — add locally)
parsed/
  selling-guide-2026-06-03/     # Full guide parse (pages 20–1188)
    batches/                      # One JSON file per 100-page segment
      batch_020-120.json
      batch_121-221.json
      …
      batch_1131-1188.json
    guide.json                    # Topic-indexed gold JSON (390 topics)
    chunks.jsonl                  # Embed-ready chunks (body only, 1,467 rows)
    validation_report.json        # Quality gates (TOC noise, topic coverage)
    progress.json                 # Incremental parse segment tracker
    manifest.json                 # Parse metadata
    document.md                   # Human-readable mirror (optional)
manifests/                        # Reserved for cross-document manifests
```

## Current dataset

| Field | Value |
|-------|--------|
| Document | `Selling-Guide_06-03-2026_highlighted.pdf` |
| PDF pages | 1,188 (body starts at **page 20**) |
| Parse segments | 12 × ~100 pages (`20-120`, `121-221`, … `1131-1188`) |
| Topics | 390 |
| Chunks | 1,467 |
| TOC lines in chunks | 0 |

Pages 1–19 (copyright, TOC, Roman preface) are **never embedded**.

## Clone and use with mortgage-rag

```bash
# Sibling layout (recommended)
git clone <this-repo> mortgage-extraction-data
git clone https://github.com/santuv15-sudo/mortgage-rag-backend.git mortgage-rag

cd mortgage-rag
cp .env.example .env
# Set PARSED_DATA_DIR=../mortgage-extraction-data/parsed/selling-guide-2026-06-03
# Set INGEST_FROM_PARSED=true

uv sync
uv run python -c "
from config.settings import get_settings
from rag.ingest import ingest_from_parsed
ingest_from_parsed(get_settings().parsed_data_dir, get_settings(), force=True)
"
```

Or via API: `POST /reindex` with `INGEST_FROM_PARSED=true`.

## Parse commands (from mortgage-rag)

**Full book (12 segments, ~6 min on Apple Silicon):**

```bash
cd ../mortgage-rag
uv run python scripts/parse_guide.py \
  --pdf ../mortgage-extraction-data/sources/pdf/Selling-Guide_06-03-2026_highlighted.pdf \
  --skip-pages 1-19 \
  --output ../mortgage-extraction-data/parsed/selling-guide-2026-06-03/ \
  --all-segments --batch-size 100 --docling-batch-size 20
```

**Resume after interruption:**

```bash
uv run python scripts/parse_guide.py ... --all-segments --batch-size 100 --resume
```

**Rebuild chunks/guide from existing batches (no re-parse):**

```bash
uv run python scripts/parse_guide.py --merge-only \
  --pdf ../mortgage-extraction-data/sources/pdf/Selling-Guide_06-03-2026_highlighted.pdf \
  --output ../mortgage-extraction-data/parsed/selling-guide-2026-06-03/ \
  --skip-pages 1-19
```

Place the source PDF at `sources/pdf/Selling-Guide_06-03-2026_highlighted.pdf` (not committed — add locally).

## Output file reference

| File | Purpose |
|------|---------|
| `batches/batch_NNN-NNN.json` | Raw Docling elements per 100-page segment (`page_range`, `elements[]`) |
| `guide.json` | Topic-indexed structure with blocks, index, and filtered noise |
| `chunks.jsonl` | One JSON object per embeddable chunk (`topic_id`, `context`, `text`, …) |
| `validation_report.json` | Automated quality checks (`passed`, `toc_lines_in_chunks`, …) |
| `progress.json` | Completed segment keys and batch filenames |

## Batch JSON shape

```json
{
  "page_range": [20, 120],
  "elements": [
    {"index": 0, "text": "Subpart A1, Approval Qualification", "page": 20, "label": "section_header"}
  ]
}
```

## What is gitignored

- `sources/pdf/*.pdf` — add the Fannie Mae PDF locally (~3 MB)
- `*.pdf` anywhere

Parsed JSON (`batches/`, `chunks.jsonl`, `guide.json`, etc.) **is committed** so teams can ingest without re-parsing.

## Related docs

See [mortgage-rag/docs/EXTRACTION-DATA.md](../mortgage-rag/docs/EXTRACTION-DATA.md) for pipeline architecture, quality filters, and ingest wiring.
