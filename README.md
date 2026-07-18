# ATS Resume Scorer

A web app that scores how well a resume matches a job description and returns actionable feedback. Built with FastAPI + Streamlit, using spaCy and Sentence Transformers for NLP and the Groq API for LLM-generated suggestions.

## What it does

1. Upload a resume (PDF / DOC / DOCX) and paste a job description.
2. The backend parses the resume, extracts skills and experience, and compares them to the JD using semantic similarity.
3. You get an ATS score, a breakdown by category (formatting, keywords, content, skill validation, ATS compatibility), and LLM-written suggestions for what to improve.
4. Past analyses are saved to your account so you can revisit them.

## Tech stack

- **Frontend:** Streamlit
- **Backend:** FastAPI (Python)
- **NLP:** spaCy (`en_core_web_md`), Sentence Transformers (`all-MiniLM-L6-v2`)
- **LLM:** Groq API (Llama 3)
- **Auth + Database:** Supabase (email/password and Google OAuth)
- **PDF report export:** WeasyPrint + Jinja2

## Project structure

```
ATS_SCORER/
├── backend/                              FastAPI app, NLP services, API routes
├── frontend/                             Streamlit app, views, components
│   └── .streamlit/secrets.toml.example   Template for frontend secrets
├── jupyter notebooks/                    Research and dataset prep (not used at runtime)
├── requirements.txt                      Combined backend + frontend dependencies
└── .env.example                          Template for backend environment variables
```

## Setup

### 1. Clone and create a virtual environment

```bash
git clone <repo-url>
cd ATS_SCORER
python -m venv venv
source venv/bin/activate         # Windows: venv\Scripts\activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
python -m spacy download en_core_web_md
```

WeasyPrint needs system libraries on Linux:

```bash
# Fedora
sudo dnf install -y cairo pango gdk-pixbuf2 libffi

# Debian / Ubuntu
sudo apt install -y libcairo2 libpango-1.0-0 libpangoft2-1.0-0 libffi-dev
```

### 3. Configure environment variables

Copy the template and fill in your keys:

```bash
cp .env.example .env
```

You need:

- A **Supabase** project — grab `SUPABASE_URL`, `SUPABASE_KEY` (service role), and `SUPABASE_ANON_KEY` from Project Settings → API.
- A **Groq** API key from [console.groq.com](https://console.groq.com) — **required**, see note below.
- (Optional) Google OAuth set up in the Supabase dashboard if you want Google sign-in.
- Set `ALLOWED_ORIGINS` in `.env` to a comma-separated list of frontend origins allowed to call the backend (defaults to `http://localhost:8501` for local dev). Add your deployed Streamlit URL here when going to production.

The Streamlit frontend also reads Supabase config from `frontend/.streamlit/secrets.toml`. Copy `secrets.toml.example` to `secrets.toml` and fill it in.

### 4. Run the backend

From the project root:

```bash
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
```

The API is now at `http://localhost:8000`.

### 5. Run the frontend

In a new terminal (with the venv activated):

```bash
streamlit run frontend/streamlit_app.py
```

The app opens at `http://localhost:8501`.

## Deployment

Backend and frontend are two separate services — there is intentionally no single "deploy everything" step.

- **Backend (FastAPI):** deploy using the included `DockerFile` to any container platform (Render, Railway, Fly.io, etc). It only packages `backend/`. Set `GROQ_API_KEY`, `SUPABASE_URL`, `SUPABASE_KEY`, `SUPABASE_ANON_KEY`, `SUPABASE_JWT_SECRET`, and `ALLOWED_ORIGINS` (comma-separated, including your frontend's deployed URL) as environment variables on the platform. The container reads `$PORT` if set, falling back to 8000.
- **Frontend (Streamlit):** deploy separately. Either:
  - **Streamlit Community Cloud** — point it at `frontend/streamlit_app.py`, no Docker needed; or
  - **Any container platform** — use `frontend/Dockerfile`, which packages `frontend/` standalone (it has no dependency on `backend/`). It also reads `$PORT`, falling back to 8501.

  Either way, set `backend.url` in that platform's secrets (or `frontend/.streamlit/secrets.toml`) to your deployed backend's URL, plus the `[supabase]` and `[google_oauth]` values from `secrets.toml.example`.
- Update the backend's `ALLOWED_ORIGINS` whenever the frontend's URL changes, or requests from the frontend will fail CORS.
- **No rate limiting yet.** `/api/v1/analyze-resume` calls the paid Groq API on every request with no throttling. Before a public deploy, add rate limiting at the edge (e.g. your platform's built-in limits, or a reverse proxy) rather than an in-process limiter — an in-memory limiter only works with a single worker process and silently stops protecting you the moment you scale to multiple uvicorn workers or instances.

## Notes for students

- **Never commit `.env` or `secrets.toml`** — they hold API keys. Both are in `.gitignore`; check before you push.
- The first run downloads the Sentence Transformer model (~80 MB). It's cached afterwards.
- **`GROQ_API_KEY` is required, not optional.** Resume parsing (skills, experience, keywords) runs through the Groq LLM before any scoring happens — without a key, `/api/v1/analyze-resume` will fail with a 500 error, not just a missing "suggestions" section.
- `jupyter notebooks/` is for experimentation and isn't required to run the app.

- 
