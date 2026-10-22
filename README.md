# CiteRAG

Document Q&A with page-level citations: answers come only from the uploaded
PDF, and every answer cites its source pages.

## How it works

1. **Per-page extraction**: PyMuPDF reads the PDF page by page, so every
   piece of text keeps its page number.
2. **Recursive chunking**: pages are split into ~1000-character chunks with
   100 characters of overlap, and each chunk inherits its page number.
3. **MiniLM embeddings in FAISS**: chunks are embedded locally with
   `all-MiniLM-L6-v2` and indexed in an in-memory FAISS store.
4. **Relevance threshold**: for each question, the top chunks are retrieved
   and anything beyond a distance cutoff is dropped. If nothing relevant is
   left, CiteRAG says so instead of calling the LLM.
5. **Gemini answer with citations**: the remaining chunks, tagged with their
   page numbers, are sent to Gemini with a strictly grounded prompt. The
   response includes the answer plus the source pages and excerpts.

```
upload → PyMuPDF per-page extraction → recursive chunking (1000 chars, 100 overlap)
       → sentence-transformer embeddings (all-MiniLM-L6-v2) → FAISS index
       → [chat] retrieval with a relevance threshold → Gemini → answer + page citations
```

Everything lives in `main.py`; `index.html` is a static frontend that talks
to it over HTTP.

## Setup

```bash
python -m venv venv
venv\Scripts\Activate.ps1        # Windows PowerShell
# source venv/bin/activate       # macOS/Linux

pip install -r requirements.txt

cp .env.example .env             # then fill in GOOGLE_API_KEY
uvicorn main:app --reload
```

Open `index.html` directly in a browser (no server needed for the frontend).

## Tech stack

- **API**: FastAPI + Uvicorn
- **PDF parsing**: PyMuPDF (`fitz`)
- **Chunking**: LangChain `RecursiveCharacterTextSplitter`
- **Embeddings**: sentence-transformers `all-MiniLM-L6-v2` (runs locally)
- **Vector store**: FAISS (`faiss-cpu`)
- **LLM**: Google Gemini (`google-generativeai`), model set via `GEMINI_MODEL`
- **Frontend**: a single static `index.html` (Tailwind CSS)

## Endpoints

### `POST /upload-pdf/`

Multipart form upload with a single `file` field (PDF only, ≤20MB).

Response:

```json
{
  "doc_id": "3f9a1c2b4e5d4f6a8b9c0d1e2f3a4b5c",
  "filename": "resume.pdf",
  "status": "Successfully processed and indexed.",
  "total_chunks": 12
}
```

### `POST /chat/`

```json
{
  "doc_id": "3f9a1c2b4e5d4f6a8b9c0d1e2f3a4b5c",
  "question": "What was the candidate's most recent job title?"
}
```

Response:

```json
{
  "answer": "The candidate's most recent job title was Senior Engineer (page 1).",
  "sources": [
    { "page": 1, "score": 0.62, "excerpt": "Senior Engineer, Acme Corp, 2022–present..." }
  ],
  "context_found": true
}
```

If nothing in the document is relevant to the question, Gemini is never
called, and the response instead looks like:

```json
{
  "answer": "I couldn't find any content in this document relevant to your question.",
  "sources": [],
  "context_found": false
}
```

## Design notes

- **all-MiniLM-L6-v2 embeddings**: small, runs locally, no per-call API cost.
  Limitation: it's trained on English text, so retrieval quality degrades on
  Indic-language documents. Swapping in a multilingual embedding model is
  the fix if that's a real need.
- **FAISS**: local, no infrastructure to run. Limitation: the index lives in
  this process's memory, so it isn't shared across multiple workers or
  processes.
- **chunk_size=1000, chunk_overlap=100**: these are common defaults, not
  values tuned for this project. Tuning them properly means measuring
  retrieval quality against an evaluation set of real questions, not
  guessing at round numbers.

## Known limitations

- Documents live in process memory (an `OrderedDict` capped at
  `MAX_DOCUMENTS`) and vanish on restart; they also don't work across
  multiple gunicorn workers, since an upload handled by one worker is
  invisible to the others. Persisting the FAISS index
  (`save_local`/`load_local`) or using a hosted vector store is the real
  fix, and is out of scope here.
- Chunking measures **characters**, not tokens. `chunk_size=1000` is roughly
  250 tokens for English, but far more tokens for Devanagari or Tamil text —
  so chunks are effectively much larger (and retrieval/context windows
  behave differently) in non-Latin scripts. A token-based length function is
  the correct fix.
- No OCR fallback: a scanned PDF with no text layer is rejected (422)
  rather than processed.
- No authentication: any caller who knows a `doc_id` can query that
  document.
