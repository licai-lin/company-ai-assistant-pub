# Deployment Guide

This guide describes the simple production-style setup for the Company AI Assistant.

For a beginner-friendly command-by-command EC2 walkthrough, see
[EC2 Backend Compose Guide](./ec2-backend-compose-guide.md).

## Target Setup

```txt
Vercel
  Next.js frontend

AWS EC2
  Docker Compose
    FastAPI backend
    Ollama

Managed Postgres
  Supabase, Neon, or AWS RDS with pgvector enabled
```

For this MVP, keep the deployment simple:

- Deploy the frontend to Vercel.
- Run only the backend and Ollama on AWS.
- Use managed Postgres instead of running the database on the EC2 instance.
- Put HTTPS in front of the backend with Nginx or Caddy.

## Production Files

The project includes these production-oriented files:

```txt
docker-compose.prod.yml
backend/.env.production.example
frontend/.env.production.example
```

The production Compose file runs only:

```txt
backend
ollama
```

It does not run:

```txt
frontend
postgres
```

The frontend belongs on Vercel. Postgres should be managed separately.

## Backend Environment

On the AWS server, create a real `.env` file for the backend stack:

```env
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/DB_NAME
OLLAMA_CHAT_MODEL=llama3.2:3b
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
OLLAMA_NUM_PREDICT=96
EMBEDDING_DIMENSION=768
FRONTEND_URL=https://your-vercel-app.vercel.app
CORS_ORIGINS=https://your-vercel-app.vercel.app
MIN_RELEVANCE_SCORE=0.35
```

Notes:

- `DATABASE_URL` should point to your managed Postgres database.
- `FRONTEND_URL` must match the real Vercel frontend URL.
- `CORS_ORIGINS` should include the browser origins allowed to call the backend.
  For one Vercel app, set it to the same value as `FRONTEND_URL`.
- `MIN_RELEVANCE_SCORE` controls when the assistant refuses questions that are
  not covered by the uploaded PDFs.
- `OLLAMA_BASE_URL` is already set in `docker-compose.prod.yml` as `http://ollama:11434`.

## How Backend Env Variables Are Loaded

The backend uses Pydantic `BaseSettings` in `backend/app/core/config.py`.

In Python, the settings fields use lowercase snake case:

```python
frontend_url: str = "http://localhost:3000"
database_url: str = "postgresql://postgres:postgres@localhost:5432/company_ai"
ollama_base_url: str = "http://localhost:11434"
```

In `.env` files and Docker Compose, the same values use uppercase environment
variable names:

```env
FRONTEND_URL=https://your-vercel-app.vercel.app
CORS_ORIGINS=https://your-vercel-app.vercel.app
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/DB_NAME
OLLAMA_BASE_URL=http://ollama:11434
```

Pydantic maps these automatically:

```txt
frontend_url        -> FRONTEND_URL
database_url        -> DATABASE_URL
ollama_base_url     -> OLLAMA_BASE_URL
ollama_chat_model   -> OLLAMA_CHAT_MODEL
embedding_dimension -> EMBEDDING_DIMENSION
```

That is why the Python code can use:

```python
settings.frontend_url
```

while the deployment config uses:

```env
FRONTEND_URL=https://your-vercel-app.vercel.app
```

The value priority is:

```txt
real environment variable
.env file
default value in config.py
```

In production Docker Compose, this line passes the value into the backend
container:

```yaml
FRONTEND_URL: ${FRONTEND_URL}
CORS_ORIGINS: ${CORS_ORIGINS:-${FRONTEND_URL}}
```

Then Pydantic loads it into:

```python
settings.frontend_url
```

The backend uses that value in `backend/app/main.py` for CORS:

```python
allow_origins=[settings.frontend_url]
```

So if the Vercel frontend is `https://your-vercel-app.vercel.app`, the backend
will allow browser requests from that domain.

## Frontend Environment

In Vercel, set:

```env
NEXT_PUBLIC_API_BASE_URL=https://api.yourdomain.com
```

This should point to the public HTTPS URL for the FastAPI backend.

## Local Production Compose Test

Before deploying to AWS, you can test `docker-compose.prod.yml` locally.

### Option 1: Use Local Compose Postgres

Start only the local Postgres service from the development Compose file:

```bash
docker compose up -d postgres
```

Then run the production backend stack while pointing to that local Postgres:

```bash
DATABASE_URL=postgresql://postgres:postgres@host.docker.internal:5432/company_ai \
FRONTEND_URL=http://localhost:3000 \
docker compose -f docker-compose.prod.yml up --build
```

In another terminal, check the backend:

```bash
curl http://localhost:8000/health
```

Expected response:

```json
{"status":"ok"}
```

### Option 2: Use Managed Postgres

If you already have Supabase, Neon, or RDS ready, test closer to production:

```bash
DATABASE_URL='postgresql://USER:PASSWORD@HOST:5432/DB_NAME' \
FRONTEND_URL=http://localhost:3000 \
docker compose -f docker-compose.prod.yml up --build
```

Then check:

```bash
curl http://localhost:8000/health
```

## Pull Ollama Models

After Ollama starts, pull the required models:

```bash
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```

Without these models:

- Upload may fail when creating embeddings.
- Chat may fail when generating an answer.

## AWS Server Notes

Recommended MVP instance:

```txt
EC2 g6.xlarge or g5.xlarge
4 vCPU
16 GB RAM
1 GPU
100 GB disk
Ubuntu
```

Install on the server:

```txt
Docker
Docker Compose
NVIDIA driver/runtime, if using a GPU instance
Nginx or Caddy for HTTPS
```

Security group:

```txt
22   SSH, only from your IP
80   HTTP
443  HTTPS
```

Do not expose these publicly:

```txt
8000   FastAPI direct port
11434  Ollama
5432   Postgres
```

The production Compose file binds FastAPI to:

```txt
127.0.0.1:8000
```

That means the public internet should reach the backend only through the HTTPS proxy.

## Start On AWS

After copying the project to AWS and creating the real `.env` file:

```bash
docker compose -f docker-compose.prod.yml --env-file .env up -d --build
```

Check logs:

```bash
docker compose -f docker-compose.prod.yml logs -f backend
docker compose -f docker-compose.prod.yml logs -f ollama
```

Check health from the server:

```bash
curl http://localhost:8000/health
```

## Update The AWS Backend After Pushing Code

After changing backend or deployment code locally, commit and push the update:

```bash
git status
git add backend docker-compose.prod.yml docs
git commit -m "Describe the backend update"
git push origin main
```

Only add files you intentionally changed. Never commit `.env`.

Then SSH into EC2:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

On EC2, pull the latest code and restart the production backend stack:

```bash
cd ~/apps/company-ai-assistant
git pull origin main
docker compose -f docker-compose.prod.yml up -d --build
```

If Git refuses to pull because `docker-compose.prod.yml` has local server
changes, inspect and stash the local file first:

```bash
git status
git diff -- docker-compose.prod.yml
git stash push -m "ec2 local docker compose change" docker-compose.prod.yml
git pull origin main
docker compose -f docker-compose.prod.yml up -d --build
```

Verify the server:

```bash
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs --tail=100 backend
curl http://127.0.0.1:8000/health
docker compose -f docker-compose.prod.yml exec backend env | grep -E 'FRONTEND_URL|CORS_ORIGINS|MIN_RELEVANCE_SCORE'
```

The final command should show your Vercel frontend URL for `FRONTEND_URL` and
`CORS_ORIGINS`, plus the configured relevance threshold. If not, edit the EC2
`.env` file and rebuild:

```bash
nano .env
docker compose -f docker-compose.prod.yml up -d --build
```

## Database Requirement

Postgres must have pgvector enabled.

The backend tries to run:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

If your managed Postgres user cannot create extensions, enable pgvector manually in the provider dashboard or SQL console.

## Uploaded File Storage

The app currently does not store the original PDF file.

It stores:

```txt
documents table
  filename
  title
  created_at

document_chunks table
  extracted text chunks
  metadata
  embedding vector
```

For a later production improvement, store original PDFs in S3 and keep the extracted chunks plus embeddings in Postgres.
