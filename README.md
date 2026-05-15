# Company AI Assistant

A local-first company knowledge assistant built with Next.js, FastAPI, PostgreSQL + pgvector, and Ollama.

The app is designed as a readable private AI assistant architecture. Upload a PDF, extract and chunk its text, embed each chunk, store the vectors, retrieve relevant context, and stream an answer back to the browser.

## What It Does

- Upload and index PDF documents.
- Store document chunks and embeddings in PostgreSQL with pgvector.
- Ask questions against uploaded documents using RAG.
- Stream answers from a local Ollama chat model.
- Show source chunks used for each answer.
- List and delete uploaded documents.

## Architecture

```mermaid
flowchart LR
  Browser[Next.js frontend] -->|upload PDFs / ask questions| API[FastAPI backend]
  API -->|extract text| PDF[pypdf]
  API -->|split into chunks| Chunker[Chunking helper]
  API -->|embeddings + chat| Ollama[Ollama]
  API -->|documents, chunks, vectors| Postgres[(PostgreSQL + pgvector)]
  Postgres -->|nearest chunks| API
  API -->|NDJSON token stream| Browser
```

## Tech Stack

- Frontend: Next.js, React, TypeScript, Tailwind CSS
- Backend: FastAPI, Pydantic, psycopg
- Database: PostgreSQL 16 with pgvector
- Local AI runtime: Ollama
- PDF parsing: pypdf
- Local orchestration: Docker Compose

## Folder Structure

```txt
backend/
  app/
    api/          # FastAPI routes for documents and chat
    core/         # environment-backed settings
    db/           # connection pool and startup schema creation
    models/       # simple Python domain objects
    schemas/      # request and response models
    services/     # ingestion, vector search, Ollama, and chat logic
    utils/        # PDF extraction and chunking helpers

frontend/
  src/
    app/          # Next.js App Router pages
    components/   # upload, chat, layout, and UI components
    hooks/        # client-side state and streaming logic
    services/     # API client functions
    types/        # shared TypeScript API shapes
    lib/          # small utilities
```

## Quick Start

Start all services:

```bash
docker compose up --build
```

In a second terminal, pull the Ollama models:

```bash
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```

Open the app:

- Frontend: `http://localhost:3000`
- Backend API docs: `http://localhost:8000/docs`
- Backend health check: `http://localhost:8000/health`
- Ollama: `http://localhost:11434`
- PostgreSQL: `localhost:5432`

The home page redirects to `/upload`. Upload one or more PDFs, then go to `/chat` and ask questions about them.

## API Endpoints

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/health` | Check whether the backend is running. |
| `POST` | `/api/documents/upload` | Upload and index a PDF. |
| `GET` | `/api/documents` | List indexed documents. |
| `DELETE` | `/api/documents/{document_id}` | Delete a document and its chunks. |
| `POST` | `/api/chat/stream` | Stream a RAG answer as newline-delimited JSON. |

## Configuration

The backend reads settings from environment variables, with local defaults in `backend/app/core/config.py`.

| Variable | Default | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | `postgresql://postgres:postgres@localhost:5432/company_ai` | PostgreSQL connection string. |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama API base URL. |
| `OLLAMA_CHAT_MODEL` | `llama3` | Chat model used by the backend outside Docker. |
| `OLLAMA_EMBEDDING_MODEL` | `nomic-embed-text` | Embedding model. |
| `EMBEDDING_DIMENSION` | `768` | Vector size expected by pgvector. |
| `FRONTEND_URL` | `http://localhost:3000` | Allowed frontend origin for CORS. |
| `MAX_UPLOAD_MB` | `25` | Maximum PDF upload size. |
| `CHUNK_SIZE` | `900` | Approximate chunk size for extracted text. |
| `CHUNK_OVERLAP` | `150` | Overlap between adjacent chunks. |
| `SEARCH_RESULT_COUNT` | `4` | Number of chunks retrieved for each question. |

Docker Compose overrides the backend chat model to `llama3.2:3b` because it is small enough for typical Docker Desktop memory settings. If your machine has more memory, you can change `OLLAMA_CHAT_MODEL` in `docker-compose.yml`.

## How The App Works

### Upload Flow

1. The user uploads a PDF from the Next.js upload page.
2. The frontend sends `multipart/form-data` to `POST /api/documents/upload`.
3. FastAPI reads the file and extracts text with `pypdf`.
4. The backend splits the text into overlapping chunks.
5. Each chunk is embedded with Ollama.
6. PostgreSQL stores the document, chunk text, metadata, and vector embedding.

### Chat Flow

1. The user asks a question from the chat page.
2. The frontend calls `POST /api/chat/stream`.
3. The backend embeds the question with Ollama.
4. pgvector finds the nearest stored chunk vectors.
5. The best chunks are inserted into the prompt as source context.
6. Ollama streams an answer back to FastAPI.
7. FastAPI forwards newline-delimited JSON events to the browser.
8. The UI renders the answer as tokens arrive and displays the sources.

## RAG In Plain English

RAG means retrieval augmented generation.

Instead of asking the model to answer only from memory, the app first retrieves relevant text from uploaded documents. The model then answers using that retrieved context. This is useful for internal company systems because policies, playbooks, and procedures can change faster than model training data.

## Local Development Notes

Docker Compose puts all services on a private network:

- The backend reaches PostgreSQL at `postgres:5432`.
- The backend reaches Ollama at `ollama:11434`.
- The browser reaches the frontend at `localhost:3000`.
- The browser reaches the backend at `localhost:8000`.

That is why `DATABASE_URL` uses `postgres` inside Compose, while `NEXT_PUBLIC_API_BASE_URL` uses `localhost` for the browser.

Useful commands:

```bash
docker compose up --build
docker compose down
docker compose logs -f backend
docker compose logs -f frontend
docker compose logs -f ollama
```

To reset local indexed data:

```bash
docker compose down -v
```
