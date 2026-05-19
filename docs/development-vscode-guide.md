# Development Guide: VS Code + Docker Compose

This guide explains how to run the project locally for development in VS Code.
It uses `docker-compose.dev.yml`, which is only for local testing and does not
change the production setup.

## What You Will Run

For development, use this command from the project root:

```bash
docker compose -f docker-compose.dev.yml up -d --build
```

This starts the local development stack:

- PostgreSQL with pgvector
- Ollama
- FastAPI backend
- Next.js frontend

## Prerequisites

Install these first:

- Docker Desktop
- VS Code
- Git

Make sure Docker Desktop is running before you start the app.

## Step 1: Open The Project In VS Code

Open the project folder:

```bash
cd /path_to_project/company-ai-assistant
code .
```

If the `code` command is not available, open VS Code manually and choose:

```text
File > Open Folder
```

Then select the `company-ai-assistant` folder.

## Step 2: Start The Development Containers

Run this from the project root:

```bash
docker compose -f docker-compose.dev.yml up -d --build
```

### What This Command Means

`docker compose` runs a group of containers defined in a Compose file.

`-f docker-compose.dev.yml` tells Docker to use the development Compose file
instead of the default `docker-compose.yml`.

`up` creates and starts all services listed in the Compose file.

`-d` means detached mode. The containers keep running in the background, so your
terminal is free for other commands.

`--build` rebuilds the backend and frontend images before starting them. This is
useful when dependencies, Dockerfiles, or app setup files have changed.

In plain English, the command means:

```text
Use the development Docker settings, rebuild the app images, start all services,
and keep them running in the background.
```

## Step 3: Pull The Ollama Models

The Ollama container starts with the app, but the model files may not exist yet.
Pull them once after the containers are running:

```bash
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```

### Why This Is Needed

The backend uses Ollama for two different jobs:

- `llama3.2:3b` generates chat answers.
- `nomic-embed-text` creates embeddings for document search.

The Docker volume keeps these models after they are downloaded, so you usually
do not need to pull them again unless you reset the Ollama volume.

## Step 4: Open The App

After the containers are running, open:

```text
http://localhost:3000
```

Useful local URLs:

| Service | URL |
| --- | --- |
| Frontend | `http://localhost:3000` |
| Backend API docs | `http://localhost:8000/docs` |
| Backend health check | `http://localhost:8000/health` |
| Ollama | `http://localhost:11434` |
| PostgreSQL | `localhost:5432` |

## Step 5: Development Workflow In VS Code

Edit frontend files under:

```text
frontend/
```

The frontend runs with Next.js development mode, so most frontend changes reload
automatically in the browser.

Edit backend files under:

```text
backend/app/
```

The development Compose file runs the backend with:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### Why The Backend Uses `--reload`

`--reload` watches the Python source files and restarts the FastAPI server when
you save changes. This is useful for local VS Code development because you do
not need to manually restart the backend after every code edit.

This setting is only in `docker-compose.dev.yml`. Production Compose settings
are separate and are not changed by this development guide.

## Common Commands

View all running project containers:

```bash
docker compose -f docker-compose.dev.yml ps
```

Follow backend logs:

```bash
docker compose -f docker-compose.dev.yml logs -f backend
```

Follow frontend logs:

```bash
docker compose -f docker-compose.dev.yml logs -f frontend
```

Follow Ollama logs:

```bash
docker compose -f docker-compose.dev.yml logs -f ollama
```

Stop the development stack:

```bash
docker compose -f docker-compose.dev.yml down
```

Stop the stack and delete local database and Ollama volumes:

```bash
docker compose -f docker-compose.dev.yml down -v
```

Use `down -v` only when you want a clean reset. It removes uploaded/indexed
document data and downloaded Ollama models.

## When To Rebuild

Use the full command when you want to start cleanly or after dependency changes:

```bash
docker compose -f docker-compose.dev.yml up -d --build
```

For normal daily development, if the containers already exist and dependencies
did not change, this is usually enough:

```bash
docker compose -f docker-compose.dev.yml up -d
```

If frontend dependencies seem stale after changing `package.json`, run:

```bash
docker compose -f docker-compose.dev.yml up -d --build --renew-anon-volumes
```

This recreates the anonymous `/app/node_modules` volume used by the frontend
container.

## Troubleshooting

If the frontend does not open, check:

```bash
docker compose -f docker-compose.dev.yml logs -f frontend
```

If the backend does not respond, check:

```bash
docker compose -f docker-compose.dev.yml logs -f backend
```

If chat or document upload fails because of Ollama models, check:

```bash
docker exec company-ai-ollama ollama list
```

If the models are missing, pull them again:

```bash
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```

If ports are already in use, make sure no other local services are using:

- `3000`
- `8000`
- `5432`
- `11434`

