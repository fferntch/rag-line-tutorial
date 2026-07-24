# rag-line-tutorial

# 📄 PDF Q&A — LangChain + Ollama + FAISS

A Retrieval-Augmented Generation (RAG) app that answers questions about a PDF
document — grounded in the document's actual text, with every answer
traceable back to the source chunks that produced it.

Runs **entirely on your own machine**: no API key, no account, no cloud call,
no per-token cost. The LLM is a local model served by [Ollama](https://ollama.com);
embeddings run locally too. The only thing that ever leaves your machine is
nothing.

Built as a learning project to understand every stage of a RAG pipeline
individually: loading, splitting, embedding, storing, retrieving, and
generating — rather than treating LangChain as a black box.

**[📚 Read the interactive learning path →](docs/learning/index.html)** — five
stop-by-stop pages (open `docs/learning/index.html` in a browser) covering the
concept, architecture, real code, and a quiz for every commit in this repo.

## Overview

Point the app at a PDF (by default, *"Advances in Artificial Intelligence
for Energy Forecasting"*) and ask it questions in plain English. The app:

1. Loads and splits the PDF into small, overlapping text chunks.
2. Embeds each chunk into a vector using a local, free embedding model.
3. Stores those vectors in a FAISS index for fast similarity search.
4. On each question, retrieves the most relevant chunks and asks a local
   Qwen3 model (via Ollama) to answer **using only that retrieved context**.
5. Shows the answer next to the exact source chunks it came from, so you
   can verify the model isn't making things up.

## Architecture

```
                     ┌─────────────┐
                     │   PDF file  │
                     └──────┬──────┘
                            │  loader.py
                            │  PyPDFLoader → RecursiveCharacterTextSplitter
                            ▼
                  ┌───────────────────┐
                  │  Document chunks   │
                  └─────────┬─────────┘
                            │  rag.py: get_embeddings()
                            │  HuggingFace all-MiniLM-L6-v2 (local, free)
                            ▼
                  ┌───────────────────┐
                  │   FAISS vector     │◄── cached on disk in vectorstore/
                  │       index        │    (rebuilt only if missing)
                  └─────────┬─────────┘
                            │  rag.py: build_rag_chain()
                            │  vectorstore.as_retriever(k=4)
                            ▼
                  ┌───────────────────┐        ┌──────────────────┐
     question ───▶│  Retrieval chain   │───────▶│  prompts.py       │
                  │  (LangChain LCEL)  │        │  QA_PROMPT        │
                  └─────────┬─────────┘        └──────────────────┘
                            │  create_stuff_documents_chain
                            ▼
                  ┌───────────────────┐
                  │  Qwen3 via Ollama  │◄── runs locally, no network call
                  │  (ChatOllama)      │
                  └─────────┬─────────┘
                            ▼
                answer + source chunks
                            │
                            ▼
                     ┌─────────────┐
                     │   app.py    │
                     │  (Streamlit)│
                     └─────────────┘
```

**Module responsibilities:**

| File          | Responsibility                                                        |
| ------------- | ----------------------------------------------------------------------|
| `config.py`   | All settings — paths, model names, chunk sizes — in one place         |
| `loader.py`   | PDF → page Documents → split chunks                                   |
| `rag.py`      | Embeddings, FAISS vector store, LLM, and the retrieval chain          |
| `prompts.py`  | The prompt template that shapes how the model uses retrieved context  |
| `app.py`      | Streamlit UI — the only file that knows about the frontend            |

## Installation

### 1. Install Ollama

Download and run the installer from **<https://ollama.com/download/windows>**,
or install via `winget`:

```powershell
winget install Ollama.Ollama
```

Ollama installs as a background app and starts automatically. Verify it's
running:

```powershell
ollama --version
```

If it's ever not running, start it manually with:

```powershell
ollama serve
```

### 2. Pull the model

```powershell
ollama pull qwen3
```

This downloads Qwen3's default size (a few GB — exact size depends on the
tag). If your machine is memory-constrained, pull a smaller tag instead
(e.g. `ollama pull qwen3:4b` or `ollama pull qwen3:1.7b`) and set
`OLLAMA_MODEL_NAME` to match (see Configuration below). If Qwen3 isn't
available in your Ollama version, `ollama pull llama3` is the documented
fallback — set `OLLAMA_MODEL_NAME=llama3` to match.

### 3. Set up the Python project

```powershell
# Clone and enter the project
cd LangChain

# Create and activate a virtual environment
python -m venv .venv
.venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

No API key, no `.env` file, no account signup required for any of this.

### Configuration (optional)

Everything has a sensible default (`config.py`), but any of these can be
overridden per-session as environment variables if you want a different
model size or a non-default Ollama host:

```powershell
$env:OLLAMA_MODEL_NAME = "qwen3:4b"
$env:OLLAMA_BASE_URL = "http://localhost:11434"
```

## Usage

```powershell
.venv\Scripts\streamlit.exe run app.py
```

This opens the app in your browser. The first question triggers a one-time
build of the FAISS index (loads the PDF, splits it, embeds every chunk) —
subsequent runs load the cached index from `vectorstore/` instantly. If
Ollama isn't running or the model hasn't been pulled yet, the app shows a
clear error instead of a crash.

To use a different PDF, replace the file in `data/` and update
`DEFAULT_PDF_PATH` in `config.py`, then delete the `vectorstore/` contents
so it rebuilds against the new document.

### Command-line smoke test (no UI)

```python
from rag import answer_question, build_rag_chain, get_or_build_vectorstore

vectorstore = get_or_build_vectorstore()
chain = build_rag_chain(vectorstore)
answer, sources = answer_question(chain, "What forecasting models does the paper compare?")
print(answer)
```

## Folder Structure

```
LangChain/
├── app.py              # Streamlit UI
├── rag.py               # Embeddings, vector store, retrieval chain, local LLM
├── loader.py             # PDF loading + text splitting
├── prompts.py             # Prompt templates
├── config.py               # Central configuration
├── requirements.txt         # Python dependencies
├── README.md                 # This file
├── .gitignore                 # Excludes vectorstore/, caches
├── data/                        # Source PDF(s)
│   └── Advances in Artificial Intelligence for Energy Forecasting.pdf
├── vectorstore/                   # FAISS index (generated, gitignored)
└── docs/
    └── learning/                     # Interactive learning path (this repo's own tutorial)
        ├── index.html                     # Line map — links to all 5 stops
        ├── 01-project-setup.html
        ├── 02-orchestration-framework.html
        ├── 03-embedding-vector-store.html
        ├── 04-retrieval-augmented-generation.html
        └── 05-ui-framework.html
```

## Screenshots

<!-- Add screenshots of the running app here, e.g.: -->
<!-- ![App screenshot](docs/screenshot-question.png) -->
<!-- ![Source chunks expanded](docs/screenshot-sources.png) -->

*(placeholder — add screenshots once the app is running locally)*

## Future Improvements

- **Conversation memory** — chain-aware follow-up questions instead of
  single-shot Q&A.
- **Multi-document support** — upload and query across several PDFs at once.
- **Hybrid search** — combine FAISS similarity search with keyword (BM25)
  search for better recall on exact terms/acronyms.
- **Streaming responses** — stream the model's answer token-by-token instead
  of waiting for the full response.
- **Evaluation harness** — a small set of question/expected-answer pairs
  to catch retrieval or prompt regressions.
- **Swap FAISS for a managed vector DB** (e.g. Pinecone, Chroma Cloud) to
  demonstrate the vector-store abstraction is swappable.
- **Host the learning path on GitHub Pages** — the `docs/learning/` folder is
  already structured for it: point Pages at `docs/` and the tutorial becomes
  a public, shareable link.
- **Package for easy distribution** — since there's no cloud dependency
  anymore, this could ship as a desktop app (e.g. via PyInstaller) instead of
  a web deploy; Streamlit Community Cloud won't work for this version since
  it has no way to run Ollama alongside it.

## Tech Stack

- [LangChain](https://python.langchain.com/) — orchestration
- [Ollama](https://ollama.com/) + Qwen3 — local LLM for answer generation, no API key
- [sentence-transformers](https://www.sbert.net/) — local embeddings (`all-MiniLM-L6-v2`)
- [FAISS](https://github.com/facebookresearch/faiss) — vector similarity search
- [Streamlit](https://streamlit.io/) — UI
