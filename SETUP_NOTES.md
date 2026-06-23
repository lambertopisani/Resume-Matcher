## How to Run Resume Matcher

Every time you come back, run these steps in order:

### 1. Start Ollama (if not already running)
Ollama runs as a background service and usually auto-starts with Windows. Verify:
```powershell
ollama list
```
If you get "not recognized", launch it from Start Menu or run:
```powershell
& "$env:LOCALAPPDATA\Programs\Ollama\ollama app.exe"
```

### 2. Start the Backend (Terminal 1)
```powershell
cd C:\Users\Lambe\OneDrive\Documents\Claude_Projects\ResumeMatcher\apps\backend
uv run app
```
Backend runs on **http://127.0.0.1:8000**. Wait until you see the Uvicorn startup message before proceeding.

### 3. Start the Frontend (Terminal 2)
```powershell
cd C:\Users\Lambe\OneDrive\Documents\Claude_Projects\ResumeMatcher\apps\frontend
npm run dev
```
Frontend runs on **http://localhost:3000**. Open this URL in your browser.

### 4. Use the App
- Upload a master resume (PDF or DOCX)
- Add job descriptions
- Tailor resumes, generate cover letters, etc.
- LLM provider is Ollama with `qwen2.5:3b` (fully local, no API key needed)

---

## Project Details

- **Repo:** https://github.com/srbhr/Resume-Matcher (cloned 2026-06-23)
- **Branch:** main
- **Stack:** FastAPI backend (Python 3.13) + Next.js 16 frontend (Node.js 24)
- **LLM:** Ollama with `qwen2.5:3b` — configured in `apps/backend/.env`
- **PDF export:** Requires both frontend + backend running (Playwright renders frontend pages)
- **Config:** `apps/backend/.env` controls LLM provider, model, timeouts, CORS

---

## Lessons Learned — Ollama Model Selection

### What works: `qwen2.5:3b`
- ~2GB, fits entirely in the RTX 500 Ada 4GB VRAM
- Parses resumes to structured JSON in ~28 seconds
- Produces complete, schema-compliant JSON output

### What does NOT work: `gemma3:4b`
- Only 1.2GB loaded into VRAM, rest ran on CPU → extremely slow (~147s for a simple resume)
- Default Ollama context window is 4096 tokens — the resume parsing prompt (schema example + full resume text) exceeds this, causing silent failures
- Even with a custom 16k-context variant (`gemma3:4b-16k`), it was too slow and produced incomplete JSON (missing required fields like `name`, `id`)
- The app's 240-second request timeout caused uploads to appear stuck at "STATUS: PROCESSING" forever

### Key rules for Ollama model selection with this app
1. **Model must fit in VRAM** — if it spills to CPU, inference is too slow for the app's timeouts
2. **Model must handle structured JSON well** — resume parsing requires complete, schema-compliant output with specific fields (`id`, `name`, `title`, etc.)
3. **Context window matters** — the parsing prompt is large; models with <8k context will silently truncate
4. **RTX 500 Ada (4GB VRAM) budget:** stick to models ≤3B parameters (qwen2.5:3b, phi4-mini, gemma3:1b)
5. **Don't assume bigger = better** — a 4B model that spills to CPU is slower than a 3B model fully in VRAM

### Other viable models (not tested but should work)
- `phi4-mini` (~2.5GB) — good at structured output
- `qwen2.5:7b` — would partially spill to RAM but still faster than gemma3:4b due to better architecture

---

## What NOT to Do

- Don't use `gemma3:4b` or larger models that won't fit in 4GB VRAM
- Don't forget to start Ollama before the backend — the health check passes but parsing hangs
- Don't change `REQUEST_TIMEOUT_SECONDS` in `.env` without also changing `NEXT_PUBLIC_REQUEST_TIMEOUT_MS` on the frontend (they must stay in sync)
- Don't create `app/api/` routes in the frontend — they shadow the backend proxy
- `ollama` CLI won't be on PATH in a new terminal after fresh install until you restart the terminal
