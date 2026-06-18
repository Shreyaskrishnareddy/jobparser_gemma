# Gemma 4 JD Parser — Technical Documentation

> LLM-powered job description parser using **Google Gemma 4** via the Google AI Studio (Gemini) API for structured data extraction.
> **Repository**: [github.com/Shreyaskrishnareddy/jobparser_gemma](https://github.com/Shreyaskrishnareddy/jobparser_gemma)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Tech Stack](#3-tech-stack)
4. [API Reference](#4-api-reference)
5. [Data Schema](#5-data-schema)
6. [Post-Processing Pipeline](#6-post-processing-pipeline)
7. [Model & Free-Tier Behavior](#7-model--free-tier-behavior)
8. [Deployment](#8-deployment)
9. [Known Limitations](#9-known-limitations)
10. [Configuration Reference](#10-configuration-reference)
11. [Project Structure](#11-project-structure)

---

## 1. Overview

This application parses job descriptions (PDF, DOCX, DOC, TXT, images) into structured JSON using **Gemma 4 31B** served through the Google AI Studio (Gemini) API. It extracts ~35 fields across job details, company info, location, compensation, requirements, skills, and metadata. Each extracted field carries a **confidence score** and **provenance** (character spans tracing back to the source text).

**Why an LLM instead of regex/keyword rules:** traditional JD parsers break on non-standard formatting and varying section headers. An LLM understands context, so it handles arbitrary JD layouts. Gemma 4's 256K context lets the entire JD be parsed in a single pass.

**Accuracy model:** the LLM extracts; a deterministic post-processing layer then grounds and corrects the output (employment-type derivation, salary normalization, soft-skill grounding, verbatim summary/description recovery from the source).

---

## 2. Architecture

```
                Browser / API client
                        |
                index.html  (drag-drop UI + JSON view)
                        |
          +---------------------------------+
          |        app.py (Flask)           |
          |  /parse  /parse/text /parse/bulk|
          +---------------------------------+
                        |
          +---------------------------------+
          |       gemma_jd_parser.py        |
          |  extract_text_from_file()       |
          |  parse_jd()  -> Gemma call      |
          |  _extract_json() (reasoning-aware)|
          |  _normalize + _post_process     |
          |  _build_output() (confidence +  |
          |     provenance wrapper)         |
          +---------------------------------+
                        |
          +---------------------------------+
          |  Google AI Studio (Gemini API)  |
          |  Model: gemma-4-31b-it          |
          +---------------------------------+
```

**Request flow:**
1. File uploaded to `/parse`; text extracted (PyMuPDF / docx2txt / antiword / Tesseract).
2. Full text injected into the extraction prompt.
3. Single call to Gemma 4 (auto-retry on transient errors; a second concise pass if the first is truncated).
4. Reasoning-aware JSON extraction — Gemma emits "thinking" before the JSON, so the extractor selects the richest balanced JSON object rather than a stray skeleton.
5. Normalization + ~25 deterministic post-processing fixes.
6. Output wrapped with per-field confidence + provenance and a `_metadata` block.

---

## 3. Tech Stack

| Layer | Choice | Role |
|-------|--------|------|
| **API** | Flask + flask-cors | HTTP server, routing |
| **WSGI** | Gunicorn (1 worker, 8 threads, 600s timeout) | Keeps /health responsive during long parses |
| **LLM** | Gemma 4 31B (`gemma-4-31b-it`) | JD text -> structured JSON |
| **Inference** | Google AI Studio (Gemini API) | Hosted Gemma; 256K context |
| **PDF / DOCX / DOC / OCR** | PyMuPDF, docx2txt, antiword, Tesseract | Text extraction |

---

## 4. API Reference

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/parse` | POST | Parse one uploaded JD file |
| `/parse/text` | POST | Parse raw JD text (JSON body with `text`) |
| `/parse/bulk` | POST | Parse multiple JD files (<=50) |
| `/health` | GET | Health + active model |

Error codes: `400` bad request, `502` Gemma API error after retries.

---

## 5. Data Schema

The output is a wrapper object:

```json
{
  "id": "<uuid>",
  "source": { "type": "file", "filename": "...", "uploaded_at": "..." },
  "detected_language": "en",
  "global_confidence": 0.94,
  "fields": { "<field>": { "value": ..., "confidence": 0.0-1.0, "provenance": {...}, "status": "ok" }, ... },
  "_metadata": { "parser": "gemma_jd_parser", "model": "gemma-4-31b-it", "processing_time_ms": ..., "..." }
}
```

Fields include: `title` (text/seniority/domain), `company`, `location`, `employment_type`, `contract_type`, `contract_duration`, `salary`, `requirements`, `responsibilities`, `skills`, `technical_skills`, `soft_skills`, `education`, `experience_years`, `benefits`, `work_authorization`, `job_domain`, `job_summary`, `description`, `job_id`, `work_mode`, `job_posted_date`, `job_expiry_date`, `reporting_to`, `team_size`, `travel_requirement`, `application_link`, `equal_opportunity_statement`, `company_website`, `industry`, `company_size`, `company_overview`, `preferred_experience`, `preferred_technologies`, `certifications`.

Convention: a field whose value is not stated in the JD is `null` (lists are `[]`). Null is the correct answer for absent data — values are never fabricated.

---

## 6. Post-Processing Pipeline

After the LLM returns JSON, `_post_process()` applies deterministic fixes (each isolated so one failure can't break the result). Highlights:

- **employment_type** — derived correctly: a stated Duration ("12 Months"), "C2C", "W2", or staffing client implies `contract`; never defaults to `full_time`.
- **salary** — normalizes abbreviations ("12-18 LPA" → 1.2M-1.8M INR; "$80K-120K" → 80000-120000); does not multiply plain numbers.
- **soft_skills** — keyword-grounded: included only when the skill (or a close variant) actually appears in the JD text; de-duplicated to canonical single-keyword form.
- **summary / description / company_overview** — recovered VERBATIM from the source section when the LLM truncated to one line.
- **industry** — inferred from company overview / context when not explicit.
- **location** — city/region/country normalized (country = nation, region = state).
- **skills** — de-duplicated; alternatives in "X or Y" split into both; technical vs soft split.
- **benefits / reporting_to** — hallucination guards (only kept when grounded in text).

A confidence score and provenance span are attached to every field in `_build_output()`.

---

## 7. Model & Free-Tier Behavior

**Default model:** `gemma-4-31b-it` (31B dense) — a strong, accurate extractor and a reasoning model (it "thinks" before emitting JSON; the extractor handles this).

| Aspect | Behavior |
|--------|----------|
| **Latency** | ~2-4 min per JD on the free tier (large model, best-effort capacity) |
| **Context** | 256K — no token-limit failures even on long JDs |
| **Rate limits** | 15 RPM / 1,500 RPD |
| **Transient errors** | Occasional 500s/timeouts; auto-retried with backoff (`LLM_MAX_RETRIES`) |

A paid Google AI Studio tier removes the latency variance and the 500s.

---

## 8. Deployment

**Render (blueprint):** push to GitHub -> **New -> Blueprint** -> select repo (uses `render.yaml`) -> set `GOOGLE_API_KEY` (`sync: false`) -> Apply. Sets `GOOGLE_MODEL`, `GOOGLE_TIMEOUT=300`, `LLM_MAX_RETRIES=4`, and a 1-worker/8-thread/600s gunicorn config.

**Docker:** `docker build -t jobparser-gemma . && docker run -p 8000:8000 -e GOOGLE_API_KEY=... jobparser-gemma` (the Dockerfile installs antiword + tesseract).

---

## 9. Known Limitations

| Limitation | Detail / Mitigation |
|------------|---------------------|
| **Free-tier latency** | 2-4 min/JD; a paid tier brings it to seconds |
| **Free-tier 500s** | Intermittent; mitigated by auto-retry |
| **Legacy `.doc`** | Needs `antiword` (in the Dockerfile) for clean extraction |

---

## 10. Configuration Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GOOGLE_API_KEY` | Yes | — | Google AI Studio key ([aistudio.google.com/apikey](https://aistudio.google.com/apikey)) |
| `GOOGLE_MODEL` | No | `gemma-4-31b-it` | Gemma model ID |
| `GOOGLE_TIMEOUT` | No | `300` | Per-request timeout (seconds) |
| `LLM_MAX_RETRIES` | No | `4` | Retries on transient errors |
| `MAX_OUTPUT_TOKENS` | No | `32768` | Output token cap |
| `PORT` | No | `8001` | Local server port |

---

## 11. Project Structure

| File | Role |
|------|------|
| `app.py` | Flask API server (single, text, bulk endpoints) |
| `gemma_jd_parser.py` | Core: text extraction, Gemma call, reasoning-aware JSON extraction, post-processing, confidence/provenance |
| `index.html` | Web UI (structured view + JSON) |
| `requirements.txt` | Dependencies |
| `Dockerfile` | Production container (antiword + tesseract) |
| `render.yaml` | Render blueprint |
| `.env.example` | Environment template |
