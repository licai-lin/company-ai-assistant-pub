# Neon Postgres Guide

This guide explains how this project connects to Neon Postgres from the Python
FastAPI backend.

It is written for someone new to backend deployment, Python environment
variables, and managed Postgres.

## What Neon Does In This Project

Neon provides the Postgres database for the app.

The app stores:

```txt
documents
  filename
  title
  created_at

document_chunks
  extracted text
  metadata
  embedding vector
```

The original uploaded PDF file is not stored in Neon. The backend reads the PDF,
extracts text, splits the text into chunks, creates embeddings with Ollama, then
stores the chunks and embeddings in Neon.

## Required Neon Feature

This project needs the `pgvector` extension because embeddings are stored in a
Postgres vector column.

In Neon, open the SQL Editor and run:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Then verify:

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'vector';
```

Expected result:

```txt
vector
```

Important: the extension name is `vector`, not `pgvector`.

## DATABASE_URL

The backend connects to Neon using the `DATABASE_URL` environment variable.

The value looks like this:

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST/DB_NAME?sslmode=require&channel_binding=require"
```

For Neon, keep these query parameters if Neon gives them to you:

```txt
sslmode=require
channel_binding=require
```

They tell the Postgres client to connect securely over SSL.

## Where To Put DATABASE_URL

For local testing, create a backend env file:

```txt
backend/.env
```

Example:

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST/DB_NAME?sslmode=require&channel_binding=require"
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_CHAT_MODEL=llama3.2:3b
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
OLLAMA_NUM_PREDICT=96
EMBEDDING_DIMENSION=768
FRONTEND_URL=http://localhost:3000
```

For AWS production, create a `.env` file beside `docker-compose.prod.yml`:

```txt
.env
```

Example:

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST/DB_NAME?sslmode=require&channel_binding=require"
FRONTEND_URL=https://your-vercel-app.vercel.app
OLLAMA_CHAT_MODEL=llama3.2:3b
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
OLLAMA_NUM_PREDICT=96
EMBEDDING_DIMENSION=768
```

Do not commit real `.env` files. They contain passwords.

## How Python Reads DATABASE_URL

The backend uses Pydantic `BaseSettings` in:

```txt
backend/app/core/config.py
```

In Python, the setting is written in lowercase:

```python
database_url: str = "postgresql://postgres:postgres@localhost:5432/company_ai"
```

In `.env`, Docker Compose, and cloud deployment settings, the same setting is
uppercase:

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST/DB_NAME?sslmode=require"
```

Pydantic maps them automatically:

```txt
database_url -> DATABASE_URL
frontend_url -> FRONTEND_URL
ollama_base_url -> OLLAMA_BASE_URL
```

The backend database code uses:

```python
settings.database_url
```

That value comes from `DATABASE_URL` when the app starts.

## Why The URL Says Pooler

Neon often gives a host that includes:

```txt
pooler
```

That means the connection goes through Neon's connection pooler.

For this project, using the pooled connection string is fine. The backend uses a
small Python connection pool, and Neon also helps manage database connections on
their side.

## Test Neon From Local Production Compose

You can test the AWS-style backend stack locally while using Neon:

```bash
DATABASE_URL='postgresql://USER:PASSWORD@HOST/DB_NAME?sslmode=require&channel_binding=require' \
FRONTEND_URL=http://localhost:3000 \
docker compose -f docker-compose.prod.yml up --build
```

In another terminal:

```bash
curl http://localhost:8000/health
```

Expected response:

```json
{"status":"ok"}
```

Then pull the Ollama models if they are not already installed:

```bash
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```

## Test Neon Directly With psql

If you have `psql` installed, test the connection:

```bash
psql "postgresql://USER:PASSWORD@HOST/DB_NAME?sslmode=require&channel_binding=require"
```

Then run:

```sql
SELECT 1;
```

Expected result:

```txt
1
```

Check pgvector:

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'vector';
```

Exit `psql`:

```sql
\q
```

## Networking

For this MVP, Neon public networking is okay:

```txt
AWS EC2 backend -> public internet with SSL -> Neon Postgres
```

Keep Neon set to allow public internet traffic, and keep SSL enabled in the
connection string.

Private networking or VPC access is a later production hardening step. It is
not required for the first deployment.

## Common Mistakes

Do not put `DATABASE_URL` in Vercel frontend env vars. The browser must never
receive the database password.

Do not remove:

```txt
sslmode=require
```

Do not commit a real `.env` file.

Do not use the Neon SQL query editor password or connection string in screenshots
or public docs.

Make sure `EMBEDDING_DIMENSION` matches the embedding model:

```txt
nomic-embed-text -> 768
```

If you change embedding models later, you may need to recreate old embeddings and
possibly change the vector column dimension.

## Secret Safety

If a real database URL is accidentally pasted into chat, committed to Git, or
shared in a screenshot, rotate the Neon password before production use.

After rotating, update only the private `.env` files or deployment secret
settings. Do not update documentation with the real password.
