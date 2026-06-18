# Gemma 4 JD Parser

LLM-powered **job description** parser using **Google Gemma 4** via the [Google AI Studio](https://aistudio.google.com) (Gemini) API. Parses JDs into structured JSON (~35 fields) with per-field **confidence scores** and **provenance** (character spans back to the source) — single-pass, no chunking.

## Features

- **Single-pass extraction** — Gemma 4's 256K context reads the entire JD at once
- **~35 structured fields** — title, company, location, employment type, salary, requirements, responsibilities, skills (technical + soft), education, experience, benefits, summary, description, dates, and more
- **Confidence + provenance** — every field includes a confidence score and the source spans it came from
- **Smart normalization** — salary abbreviations ("12-18 LPA" → 1.2M–1.8M), employment-type derivation (Duration/C2C → contract), grounded soft-skills, verbatim summary/description recovery
- **Reasoning-aware JSON extraction** — handles Gemma's "thinking" output robustly
- **Auto-retry** — transient free-tier errors (500/503/429/timeout) retried with backoff
- **File + text + bulk APIs** — PDF, DOC, DOCX, TXT, images (OCR)
- **Web UI** — drag-and-drop with a structured view + raw JSON
- **Docker- and Render-ready**

## Quick Start

```bash
git clone https://github.com/Shreyaskrishnareddy/jobparser_gemma.git
cd jobparser_gemma
pip install -r requirements.txt
export GOOGLE_API_KEY=your_key_here   # free at https://aistudio.google.com/apikey
python app.py
```

Open http://localhost:8001.

### Docker

```bash
docker build -t jobparser-gemma .
docker run -p 8000:8000 -e GOOGLE_API_KEY=your_key_here jobparser-gemma
```

## API

- `POST /parse` — parse an uploaded JD file (`-F "file=@jd.pdf"`)
- `POST /parse/text` — parse raw JD text (`{"text": "..."}`)
- `POST /parse/bulk` — parse up to 50 files (`-F "files=@..."`)
- `GET /health` — health + active model

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `GOOGLE_API_KEY` | *(required)* | Google AI Studio key |
| `GOOGLE_MODEL` | `gemma-4-31b-it` | Gemma model |
| `GOOGLE_TIMEOUT` | `300` | Per-request timeout (s) |
| `LLM_MAX_RETRIES` | `4` | Retries on transient errors |
| `PORT` | `8001` | Server port |

> **Free-tier note:** Gemma 4 31B is a large reasoning model — a full JD takes ~2–4 min on the free tier and may intermittently return 500s (handled by retry). A paid tier removes the latency/variance.

## Deployment (Render)

Push to GitHub → **New → Blueprint** → select the repo (uses `render.yaml`) → set `GOOGLE_API_KEY` → Apply.

See [DOCUMENTATION.md](DOCUMENTATION.md) for architecture, schema, and the post-processing pipeline.

## License

MIT
