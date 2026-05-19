# Environment Variables Guide

This guide explains where to put `.env` files, which variables this project uses, and what each variable controls.

Do not commit real `.env` files to GitHub. They can contain passwords, private URLs, and deployment secrets.

This repo keeps example files only:

```text
frontend/.env.example
frontend/.env.production.example
backend/.env.example
backend/.env.production.example
```

Use those files as templates, then create your own local `.env` files.

For the EC2 production Docker Compose setup, create a repo-root `.env` file next
to `docker-compose.prod.yml`. That file is also private and should not be
committed.

## Local Frontend Env

For local frontend development, create:

```text
frontend/.env
```

Example:

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

### Frontend Variables

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

This tells the browser where the backend API is running.

Local value:

```text
http://localhost:8000
```

Production value:

```text
https://companyaiass.ddns.net
```

The `NEXT_PUBLIC_` prefix is important. In Next.js, variables with this prefix are exposed to browser code.

Do not put secrets in `NEXT_PUBLIC_` variables.

Good:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

Bad:

```env
NEXT_PUBLIC_DATABASE_PASSWORD=secret-password
```

The frontend code reads this value in:

```text
frontend/src/services/api-client.ts
```

If `NEXT_PUBLIC_API_BASE_URL` is missing, the frontend uses this fallback:

```text
http://localhost:8000
```

That fallback only applies during local development. In production, the frontend
throws an error if `NEXT_PUBLIC_API_BASE_URL` is missing because deployed upload
and chat features need the real backend URL.

## Local Backend Env

For local backend development, create:

```text
backend/.env
```

Example:

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/company_ai
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_CHAT_MODEL=llama3.2:3b
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
OLLAMA_NUM_PREDICT=96
EMBEDDING_DIMENSION=768
FRONTEND_URL=http://localhost:3000
CORS_ORIGINS=http://localhost:3000,http://127.0.0.1:3000
MAX_UPLOAD_MB=25
CHUNK_SIZE=900
CHUNK_OVERLAP=150
SEARCH_RESULT_COUNT=3
MIN_RELEVANCE_SCORE=0.35
```

The backend reads these variables in:

```text
backend/app/core/config.py
```

The backend uses Pydantic settings, so variable names are uppercase in `.env` but lowercase in Python.

Example:

```text
DATABASE_URL -> settings.database_url
FRONTEND_URL -> settings.frontend_url
```

Required backend variables:

```text
DATABASE_URL
```

Recommended backend variables:

```text
OLLAMA_BASE_URL
OLLAMA_CHAT_MODEL
OLLAMA_EMBEDDING_MODEL
OLLAMA_NUM_PREDICT
EMBEDDING_DIMENSION
FRONTEND_URL
CORS_ORIGINS
MIN_RELEVANCE_SCORE
```

Optional tuning variables with code defaults:

```text
MAX_UPLOAD_MB
CHUNK_SIZE
CHUNK_OVERLAP
SEARCH_RESULT_COUNT
```

## Backend Variables

### DATABASE_URL

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/company_ai
```

This tells the backend how to connect to PostgreSQL.

It includes:

- database type
- username
- password
- host
- port
- database name

Local Docker Compose usually uses:

```env
DATABASE_URL=postgresql://postgres:postgres@postgres:5432/company_ai
```

Local backend running directly on your machine usually uses:

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/company_ai
```

Production should use your managed database URL, for example Neon or AWS RDS:

```env
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/DB_NAME
```

Never put the real production database URL in frontend env variables.

### OLLAMA_BASE_URL

```env
OLLAMA_BASE_URL=http://localhost:11434
```

This tells the backend where Ollama is running.

Local value:

```text
http://localhost:11434
```

Docker Compose may use:

```text
http://ollama:11434
```

The backend uses this URL to create embeddings and stream chat responses.

### OLLAMA_CHAT_MODEL

```env
OLLAMA_CHAT_MODEL=llama3.2:3b
```

This is the model used to generate chat answers.

Example values:

```text
llama3.2:3b
llama3.1:8b
```

Use a smaller model if your machine or server has limited memory.

### OLLAMA_EMBEDDING_MODEL

```env
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
```

This is the model used to convert documents and questions into vectors.

Those vectors are used for document search.

If you change this model, make sure `EMBEDDING_DIMENSION` still matches the model output size.

### OLLAMA_NUM_PREDICT

```env
OLLAMA_NUM_PREDICT=96
```

This controls the maximum number of tokens the chat model should generate per answer.

Higher values can produce longer answers but may be slower.

### EMBEDDING_DIMENSION

```env
EMBEDDING_DIMENSION=768
```

This is the vector size stored in PostgreSQL.

For `nomic-embed-text`, the dimension is usually `768`.

If this value does not match the embedding model, document search can fail.

### FRONTEND_URL

```env
FRONTEND_URL=http://localhost:3000
```

This tells the backend which frontend origin is allowed to call the API from a browser.

It is used for CORS.

Local value:

```text
http://localhost:3000
```

Production value:

```text
https://your-vercel-app.vercel.app
```

After deploying frontend to Vercel, set `FRONTEND_URL` in the backend deployment to the Vercel URL.

### CORS_ORIGINS

```env
CORS_ORIGINS=http://localhost:3000,http://127.0.0.1:3000
```

This is the comma-separated list of browser origins allowed to call the backend.

Local value:

```text
http://localhost:3000,http://127.0.0.1:3000
```

Production value:

```text
https://your-vercel-app.vercel.app
```

For a single production frontend, set `CORS_ORIGINS` to the same URL as
`FRONTEND_URL`. If chat works locally but shows a browser network error in
production, confirm this value is present in the backend container.

### MAX_UPLOAD_MB

```env
MAX_UPLOAD_MB=25
```

This controls the maximum PDF upload size in megabytes.

If a user uploads a file larger than this, the backend rejects it.

### CHUNK_SIZE

```env
CHUNK_SIZE=900
```

This controls how much text goes into each document chunk before embedding.

Larger chunks include more context but may make search less precise.

### CHUNK_OVERLAP

```env
CHUNK_OVERLAP=150
```

This controls how much text overlaps between neighboring chunks.

Overlap helps avoid losing context at chunk boundaries.

### SEARCH_RESULT_COUNT

```env
SEARCH_RESULT_COUNT=3
```

This controls how many document chunks are returned from vector search and used as context for the chat answer.

Higher values give the model more context but can make prompts larger.

### MIN_RELEVANCE_SCORE

```env
MIN_RELEVANCE_SCORE=0.35
```

This controls when the backend decides a question is not covered by the
uploaded PDFs.

If the best matching chunk score is below this value, the backend skips the
chat model and returns a normal answer saying it does not know from the uploaded
documents. This avoids hallucinated answers and prevents unrelated questions
from looking like broken network requests.

Raise it if the assistant answers too many unrelated questions. Lower it if the
assistant refuses questions that really are covered by the PDFs.

## Production Frontend Env In Vercel

Do not create a production `.env` file inside Vercel manually in the repo.

Instead, add frontend environment variables in the Vercel dashboard:

```text
Project -> Settings -> Environment Variables
```

Add:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

Then redeploy the frontend.

Next.js includes `NEXT_PUBLIC_` variables in the browser bundle during build, so changing this value usually requires a new Vercel deployment.

## Production Backend Env In AWS

Do not commit production backend secrets to GitHub.

Put backend environment variables in the AWS service that runs the backend.

For example:

- ECS task environment variables
- Elastic Beanstalk environment properties
- EC2 systemd environment file
- AWS Secrets Manager
- AWS Systems Manager Parameter Store

If you are using this repo's EC2 Docker Compose production file:

```text
docker-compose.prod.yml
```

create this file on the EC2 server:

```text
.env
```

That repo-root `.env` is read by Docker Compose. It should sit beside
`docker-compose.prod.yml`, not inside `backend/`.

EC2 Docker Compose example:

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

In `docker-compose.prod.yml`, `OLLAMA_BASE_URL` is already set to:

```text
http://ollama:11434
```

That is the internal Docker network address for the Ollama container, so you do
not need to put `OLLAMA_BASE_URL` in the repo-root production `.env` unless you
change the Compose file.

Production backend example:

```env
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/DB_NAME
OLLAMA_BASE_URL=http://your-ollama-host:11434
OLLAMA_CHAT_MODEL=llama3.2:3b
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
OLLAMA_NUM_PREDICT=96
EMBEDDING_DIMENSION=768
FRONTEND_URL=https://your-vercel-app.vercel.app
CORS_ORIGINS=https://your-vercel-app.vercel.app
MAX_UPLOAD_MB=25
CHUNK_SIZE=900
CHUNK_OVERLAP=150
SEARCH_RESULT_COUNT=3
MIN_RELEVANCE_SCORE=0.35
```

## Local Docker Compose Env

Docker Compose already passes several environment variables in:

```text
docker-compose.yml
docker-compose.dev.yml
```

For the local Compose stack, you usually do not need `frontend/.env` or
`backend/.env` because the Compose files define the values for each container.

For Docker Compose, the backend talks to Postgres using the service name:

```env
DATABASE_URL=postgresql://postgres:postgres@postgres:5432/company_ai
```

That works because `postgres` is the container name on the Docker network.

The browser is outside Docker, so the frontend API URL uses localhost:

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

## What To Commit

Commit example files and documentation:

```text
frontend/.env.example
frontend/.env.production.example
backend/.env.example
backend/.env.production.example
docs/env-guide.md
```

Do not commit real env files:

```text
frontend/.env
frontend/.env.local
frontend/.env.production
backend/.env
backend/.env.local
backend/.env.production
.env
```

Also do not commit private notes that contain real connection strings or passwords.

## Quick Setup Checklist

Local frontend:

```text
frontend/.env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

Local backend:

```text
backend/.env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/company_ai
OLLAMA_BASE_URL=http://localhost:11434
FRONTEND_URL=http://localhost:3000
```

Vercel frontend:

```text
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

AWS backend:

```text
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/DB_NAME
FRONTEND_URL=https://your-vercel-app.vercel.app
CORS_ORIGINS=https://your-vercel-app.vercel.app
MIN_RELEVANCE_SCORE=0.35
```

EC2 Docker Compose backend:

```text
.env
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/DB_NAME
OLLAMA_CHAT_MODEL=llama3.2:3b
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
OLLAMA_NUM_PREDICT=96
EMBEDDING_DIMENSION=768
FRONTEND_URL=https://your-vercel-app.vercel.app
CORS_ORIGINS=https://your-vercel-app.vercel.app
MIN_RELEVANCE_SCORE=0.35
```
