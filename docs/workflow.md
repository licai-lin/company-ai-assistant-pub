# Project Workflow

This project is a local-first RAG application. Users upload company PDFs, the backend turns those PDFs into searchable vector chunks, and the chat screen streams answers grounded in the uploaded documents.

## System Overview

```mermaid
flowchart LR
  user[User]
  frontend[Next.js frontend<br/>Upload and chat UI]
  backend[FastAPI backend<br/>Routes and services]
  postgres[(PostgreSQL + pgvector<br/>Documents, chunks, embeddings)]
  ollama[Ollama<br/>Embedding and chat models]

  user -->|Uploads PDFs<br/>asks questions| frontend
  frontend -->|HTTP requests| backend
  backend -->|Store and retrieve data| postgres
  backend -->|Create embeddings<br/>stream chat completions| ollama
  backend -->|JSON responses<br/>NDJSON token stream| frontend
  frontend -->|Rendered answer<br/>source chunks| user
```

## Upload And Index Workflow

```mermaid
sequenceDiagram
  actor User
  participant UI as Next.js Upload Page
  participant API as FastAPI /api/documents/upload
  participant Doc as DocumentService
  participant PDF as PDF Extractor
  participant Chunk as Chunking Helper
  participant Ollama as Ollama Embeddings
  participant DB as PostgreSQL + pgvector

  User->>UI: Select PDF
  UI->>API: POST multipart/form-data
  API->>Doc: upload_pdf(file)
  Doc->>Doc: Validate content type and file size
  Doc->>PDF: Extract text from PDF bytes
  PDF-->>Doc: Plain text
  Doc->>Chunk: Split text into overlapping chunks
  Chunk-->>Doc: Chunk list
  Doc->>DB: Insert document record

  loop For each chunk
    Doc->>Ollama: Generate embedding
    Ollama-->>Doc: Vector
    Doc->>DB: Insert chunk, metadata, and embedding
  end

  DB-->>Doc: Commit indexed document
  Doc-->>API: UploadResponse
  API-->>UI: Document summary and chunk count
  UI-->>User: Show uploaded document
```

## Chat Retrieval Workflow

```mermaid
sequenceDiagram
  actor User
  participant UI as Next.js Chat Page
  participant API as FastAPI /api/chat/stream
  participant Chat as ChatService
  participant Search as VectorSearchService
  participant Ollama as Ollama
  participant DB as PostgreSQL + pgvector

  User->>UI: Ask a question
  UI->>API: POST JSON question
  API->>Chat: stream_answer(question)
  Chat->>Search: search(question)
  Search->>Ollama: Generate question embedding
  Ollama-->>Search: Query vector
  Search->>DB: Nearest-neighbor vector query
  DB-->>Search: Most relevant document chunks
  Search-->>Chat: Ranked chunks
  Chat-->>API: Stream sources event
  Chat->>Chat: Build prompt from question and chunks
  Chat->>Ollama: Stream chat completion

  loop For each generated token
    Ollama-->>Chat: Token
    Chat-->>API: NDJSON token event
    API-->>UI: Forward token event
    UI-->>User: Append token to answer
  end

  Chat-->>API: NDJSON done event
  API-->>UI: Finish stream
  UI-->>User: Show final answer and sources
```

## Backend Service Responsibilities

```mermaid
flowchart TB
  routes[API routes<br/>documents.py and chat.py]
  documentService[DocumentService<br/>PDF validation, extraction, chunking, indexing]
  chatService[ChatService<br/>RAG prompt creation and response streaming]
  vectorSearch[VectorSearchService<br/>Question embedding and pgvector search]
  ollamaService[OllamaService<br/>Embedding and chat model calls]
  database[Database pool<br/>Startup schema and PostgreSQL access]

  routes --> documentService
  routes --> chatService
  documentService --> ollamaService
  documentService --> database
  chatService --> vectorSearch
  chatService --> ollamaService
  vectorSearch --> ollamaService
  vectorSearch --> database
```

## Data Flow In Plain English

1. A PDF is uploaded from the frontend.
2. The backend extracts readable text, splits it into overlapping chunks, and asks Ollama for an embedding for each chunk.
3. PostgreSQL stores the document metadata, chunk text, and pgvector embeddings.
4. A user asks a question in the chat UI.
5. The backend embeds the question, searches pgvector for similar chunks, and builds a prompt from those chunks.
6. Ollama streams the answer back through FastAPI as newline-delimited JSON.
7. The frontend renders tokens as they arrive and shows the source chunks used for the answer.

## Whole Project Text Diagram

```txt
COMPANY AI ASSISTANT

User
  |
  | 1. Upload PDF / ask question
  v
Next.js Frontend
  - Upload page
  - Chat page
  - API client
  - Streaming chat hook
  |
  | 2. HTTP request
  |    - POST /api/documents/upload
  |    - GET /api/documents
  |    - DELETE /api/documents/{document_id}
  |    - POST /api/chat/stream
  v
FastAPI Backend
  - documents.py handles upload, list, delete
  - chat.py handles streamed answers
  |
  +-----------------------------+
  |                             |
  | Upload path                 | Chat path
  |                             |
  v                             v
DocumentService               ChatService
  - validate PDF                - receive question
  - extract PDF text            - request relevant chunks
  - split into chunks           - build RAG prompt
  - embed chunks                - stream answer events
  - store document
  |                             |
  v                             v
OllamaService                 VectorSearchService
  - chunk embeddings             - embed question
  |                              - search similar chunks
  v                             |
PostgreSQL + pgvector <---------+
  - documents table
  - document_chunks table
  - vector embeddings
  |
  | Relevant source chunks
  v
ChatService
  |
  | Prompt with retrieved context
  v
OllamaService
  |
  | Stream generated tokens
  v
FastAPI StreamingResponse
  |
  | Newline-delimited JSON events
  | - sources
  | - token
  | - error
  | - done
  v
Next.js Frontend
  |
  | Render answer and source chunks
  v
User
```
