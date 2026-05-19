# Detailed Guide: Run The Backend On EC2 With Docker Compose

This guide explains how to run the production backend stack on an AWS EC2
Ubuntu server using Docker Compose.

It explains each step clearly, including what the commands do and why they are
needed.

This guide does not run the frontend on EC2. The intended production setup is:

```text
User browser
  -> Vercel frontend
  -> HTTPS backend domain
  -> EC2 Nginx
  -> FastAPI backend container
  -> Ollama container
  -> Managed Postgres database
```

## What Runs Where

On EC2:

- FastAPI backend
- Ollama
- Nginx reverse proxy

Outside EC2:

- Frontend on Vercel
- PostgreSQL database from Neon, Supabase, RDS, or another managed Postgres
  provider with pgvector

The EC2 server does not run the Next.js frontend and does not run the database.

## Files Used By This Guide

The main Compose file for EC2 is:

```text
docker-compose.prod.yml
```

The backend environment values come from a real `.env` file in the project root:

```text
company-ai-assistant/
├── .env
├── backend/
└── docker-compose.prod.yml
```

Do not commit the real `.env` file to GitHub because it contains secrets.

## Before You Start

You should already have:

- An AWS account
- An EC2 Ubuntu instance
- SSH access to the instance
- A GitHub repository for this project
- A managed Postgres database with pgvector enabled
- A deployed Vercel frontend, or at least the final Vercel URL you plan to use
- A domain or subdomain for the backend, such as `api.example.com`

Example backend domain used in this guide:

```text
companyaiass.ddns.net
```

Replace it with your real backend domain.

## Step 1: Connect To EC2

You connect to EC2 so you can prepare and run the backend on the actual server that will host it. The SSH connection gives you a secure terminal on the Ubuntu machine, where you can install packages, clone the project, configure environment variables, start Docker Compose, and later inspect logs or restart services when needed.

From your local computer, connect with SSH:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Example:

```bash
ssh -i ~/Downloads/company-ai.pem ubuntu@107.23.86.2
```

What this does:

- `ssh` opens a secure terminal connection to your server.
- `-i /path/to/your-key.pem` tells SSH which private key to use.
- `ubuntu` is the default username for Ubuntu EC2 images.
- `YOUR_EC2_PUBLIC_IP` is the public IPv4 address shown in AWS EC2.

After this step, the commands you type run on the EC2 server, not on your local computer.

## Step 2: Update Ubuntu Packages

Run:

```bash
sudo apt update
sudo apt upgrade -y
```

What this does:

- `sudo` stands for "superuser do" (or sometimes "substitute user do").
- `sudo` runs the command with administrator permissions.
- `apt` is Ubuntu's package manager, the tool used to find, install, update, and remove software packages.
- `apt update` refreshes Ubuntu's package list.
- `apt upgrade -y` installs available updates.
- `-y` automatically answers yes to install prompts.

This is a good first step on a new server because it installs security updates.

## Step 3: Install Git

Run:

```bash
sudo apt install -y git
```

Check it:

```bash
git --version
```

Why this is needed:

- Git is used to clone your project from GitHub onto EC2.
- Later, Git is used to pull new backend updates from GitHub.

## Step 4: Install Docker And Docker Compose

Install packages needed to add Docker's official Ubuntu repository:

```bash
sudo apt install -y ca-certificates curl gnupg
```

Create Docker's keyring folder:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's signing key:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Allow Ubuntu to read the key:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Add Docker's package repository:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Check Docker:

```bash
docker --version
docker compose version
```

Why this is needed:

- Docker runs the backend and Ollama in containers.
- Docker Compose starts both containers together using `docker-compose.prod.yml`.

## Step 5: Allow Your User To Run Docker

Run:

```bash
sudo usermod -aG docker $USER
```

Then disconnect from SSH:

```bash
exit
```

Connect again:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Test Docker without `sudo`:

```bash
docker ps
```

Why this is needed:

- By default, Docker often requires administrator access.
- Adding your user to the `docker` group lets you run `docker` commands without
  typing `sudo` each time.
- Logging out and back in reloads your Linux group permissions.

## Step 6: Clone The Project Onto EC2

Create an apps folder:

```bash
mkdir -p ~/apps
cd ~/apps
```

Clone your repository:

```bash
git clone <YOUR_GITHUB_REPO_URL> company-ai-assistant
```

Example:

```bash
git clone https://github.com/YOUR_USERNAME/company-ai-assistant.git company-ai-assistant
```

Go into the project:

```bash
cd company-ai-assistant
```

Check the files:

```bash
ls
```

You should see:

```text
README.md
backend
frontend
docs
docker-compose.prod.yml
```

Why this is needed:

- EC2 needs a copy of your backend code.
- Docker Compose uses the files in this project folder to build and run the
  backend container.

## Step 7: Understand The Production Compose File

The production Compose file is:

```text
docker-compose.prod.yml
```

It starts two services:

```text
ollama
backend
```

The backend is exposed like this:

```yaml
ports:
  - "127.0.0.1:8000:8000"
```

What this means:

- The backend listens on port `8000` inside the container.
- EC2 exposes that port only on `127.0.0.1`.
- `127.0.0.1` means localhost inside the EC2 server.
- The public internet cannot directly reach this backend port.

This is good for production because users should access the backend through
Nginx and HTTPS, not directly through port `8000`.

## Step 8: Create The Root `.env` File

From the project root, create `.env`:

```bash
nano .env
```

Paste values like this:

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

Replace:

- `USER` with your database username.
- `PASSWORD` with your database password.
- `HOST` with your database host.
- `DB_NAME` with your database name.
- `https://your-vercel-app.vercel.app` with your real Vercel frontend URL.

Save in Nano:

```text
Ctrl + O
Enter
Ctrl + X
```

Check that the file exists:

```bash
ls -la .env
```

Important:

- The `.env` file must be next to `docker-compose.prod.yml`.
- Do not put production backend secrets in `frontend/.env`.
- Do not commit `.env` to GitHub.
- Do not share screenshots showing the full `DATABASE_URL`.

## Step 9: Understand The Important Environment Variables

`DATABASE_URL` tells the backend how to connect to Postgres.

Example shape:

```env
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/DB_NAME
```

`FRONTEND_URL` is your public frontend URL.

Example:

```env
FRONTEND_URL=https://company-ai-assistant.vercel.app
```

`CORS_ORIGINS` controls which browser origins can call the backend.

For one frontend, it can match `FRONTEND_URL`:

```env
CORS_ORIGINS=https://company-ai-assistant.vercel.app
```

`OLLAMA_CHAT_MODEL` is the model used to write answers:

```env
OLLAMA_CHAT_MODEL=llama3.2:3b
```

`OLLAMA_EMBEDDING_MODEL` is the model used for document search embeddings:

```env
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
```

`EMBEDDING_DIMENSION` must match the embedding model:

```env
EMBEDDING_DIMENSION=768
```

## Step 10: Start The Backend Stack

From the project root, run:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

What this command means:

- `docker compose` runs multiple containers together.
- `-f docker-compose.prod.yml` tells Docker to use the production Compose file.
- `up` creates and starts the services.
- `-d` runs them in the background.
- `--build` rebuilds the backend image from the latest code before starting.

In plain English:

```text
Use the production Compose settings, build the backend image, start backend and
Ollama, and keep them running in the background.
```

Check the services:

```bash
docker compose -f docker-compose.prod.yml ps
```

Expected result:

```text
company-ai-backend   Up
company-ai-ollama    Up
```

## Step 11: Read Backend Logs

Run:

```bash
docker compose -f docker-compose.prod.yml logs --tail=100 backend
```

Good signs:

```text
Application startup complete.
Uvicorn running on http://0.0.0.0:8000
```

If you see database connection errors, check:

- `DATABASE_URL` in `.env`
- Database password
- Database host
- Whether your database allows connections from EC2
- Whether pgvector is enabled in the database

## Step 12: Test The Backend From Inside EC2

Run:

```bash
curl http://127.0.0.1:8000/health
```

Expected result:

```json
{"status":"ok"}
```

Why this works:

- The backend is bound to EC2 localhost.
- You are running `curl` from inside the EC2 server.
- So EC2 can reach its own private backend port.

Why your browser cannot use this URL:

- In your browser, `127.0.0.1` means your own laptop, not EC2.
- Production users need a public HTTPS domain.

## Step 13: Pull The Ollama Models

Run:

```bash
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```

Check installed models:

```bash
docker exec company-ai-ollama ollama list
```

Why this is needed:

- The Ollama container is running, but model files may not be downloaded yet.
- The chat model is needed for answers.
- The embedding model is needed for document upload and search.
- The `ollama_data` Docker volume keeps the models after restart.

## Step 14: Point Your Domain To EC2

In your DNS provider, create an `A` record:

```text
companyaiass.ddns.net -> YOUR_EC2_PUBLIC_IP
```

Example:

```text
companyaiass.ddns.net -> 107.23.86.2
```

What this does:

- A domain name is easier to remember than an IP address.
- DNS tells browsers which server IP belongs to the domain.
- Nginx and Certbot need a real domain to set up HTTPS.

Check DNS:

```bash
ping companyaiass.ddns.net
```

Or:

```bash
nslookup companyaiass.ddns.net
```

The returned IP should match your EC2 public IPv4 address.

Important note:

- A normal EC2 public IP can change if the instance is stopped and started.
- For a stable backend, use an AWS Elastic IP and point DNS to the Elastic IP.

## Step 15: Configure The EC2 Security Group

The EC2 Security Group is AWS's firewall for your server.

For production with Nginx and HTTPS, allow:

```text
HTTP    TCP   80    0.0.0.0/0
HTTPS   TCP   443   0.0.0.0/0
SSH     TCP   22    your-ip-address/32
```

Do not publicly expose these:

```text
8000   FastAPI direct backend port
11434  Ollama
5432   Postgres
```

Why:

- Port `80` is used for HTTP and Let's Encrypt certificate setup.
- Port `443` is used for HTTPS.
- Port `22` is only for SSH administration.
- Port `8000` should stay private behind Nginx.
- Ollama should never be public.
- Postgres is managed outside EC2 and should not be opened from this server.

## Step 16: Install Nginx

Install Nginx:

```bash
sudo apt update
sudo apt install -y nginx
```

Start it:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Check status:

```bash
sudo systemctl status nginx
```

Good result:

```text
Active: active (running)
```

What Nginx does:

- Receives public web requests on port `80` and `443`.
- Forwards API requests to FastAPI on `127.0.0.1:8000`.
- Later, handles HTTPS certificates.

## Step 17: Create The Nginx Reverse Proxy Config

Create a new site config. The Nano file name is the full path after `sudo nano`: `/etc/nginx/sites-available/company-ai-backend`. This creates or edits an Nginx site configuration file named `company-ai-backend` inside the `sites-available` directory.

```bash
sudo nano /etc/nginx/sites-available/company-ai-backend
```

Paste this config, replacing the domain:

```nginx
server {
    listen 80;
    server_name companyaiass.ddns.net;

    location / {
        proxy_pass http://127.0.0.1:8000;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Save:

```text
Ctrl + O
Enter
Ctrl + X
```

Enable the site:

Nginx keeps available site configs in `/etc/nginx/sites-available/`, but it only uses the configs that are linked from `/etc/nginx/sites-enabled/`. This command creates a shortcut from `sites-enabled` to the config file you just created, which tells Nginx to start using your backend site configuration.

Here, `ln` means "link". The `-s` flag creates a symbolic link, which is a pointer to the original file rather than a separate copy.

```bash
sudo ln -s /etc/nginx/sites-available/company-ai-backend /etc/nginx/sites-enabled/company-ai-backend
```

Remove the default Nginx site:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Test the config:

The `-t` flag tells Nginx to test the configuration syntax and referenced files without starting or reloading the server.

```bash
sudo nginx -t
```

Expected result:

```text
syntax is ok
test is successful
```

Reload Nginx:

```bash
sudo systemctl reload nginx
```

Test through the domain over HTTP:

```bash
curl http://companyaiass.ddns.net/health
```

Expected result:

```json
{"status":"ok"}
```

What this proves:

- DNS points to EC2.
- AWS Security Group allows port `80`.
- Nginx is running.
- Nginx can reach FastAPI on `127.0.0.1:8000`.
- FastAPI is healthy.

## Step 18: Add HTTPS With Certbot

Install Certbot:

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

Request a certificate:

```bash
sudo certbot --nginx -d companyaiass.ddns.net
```

Certbot may ask for:

```text
Email address
Agree to terms
Share email with EFF
Redirect HTTP to HTTPS
```

Recommended choices:

```text
Agree to terms: yes
Share email: no
Redirect HTTP to HTTPS: yes
```

Test Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Test HTTPS:

```bash
curl https://companyaiass.ddns.net/health
```

Expected result:

```json
{"status":"ok"}
```

Check certificate renewal:

```bash
sudo certbot renew --dry-run
```

Why HTTPS is needed:

- Vercel serves the frontend over HTTPS.
- Browsers can block HTTPS pages from calling HTTP APIs.
- HTTPS protects traffic between the user's browser and EC2.
- Certbot installs a trusted Let's Encrypt certificate for free.

## Step 19: Set The Vercel Frontend API URL

In Vercel, open:

```text
Project -> Settings -> Environment Variables
```

Set:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

Then redeploy the frontend.

Why this is needed:

- The frontend needs to know where the backend API lives.
- `NEXT_PUBLIC_` variables are included in the browser JavaScript bundle.
- Changing this value in Vercel does not affect an already-built deployment.
- You must redeploy for the new value to be included.

Do not use:

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

In a user's browser, `localhost` means the user's own computer, not EC2.

Do not use:

```env
NEXT_PUBLIC_API_BASE_URL=http://companyaiass.ddns.net:8000
```

That uses plain HTTP and the direct backend port. Production should use HTTPS
through Nginx.

## Step 20: Check CORS

Your EC2 `.env` should include the Vercel frontend URL:

```env
FRONTEND_URL=https://your-vercel-app.vercel.app
CORS_ORIGINS=https://your-vercel-app.vercel.app
```

If your Vercel app has a custom domain, use that domain too.

Example:

```env
FRONTEND_URL=https://app.example.com
CORS_ORIGINS=https://app.example.com
```

If you allow multiple frontend origins, separate them with commas:

```env
CORS_ORIGINS=https://app.example.com,https://company-ai-assistant.vercel.app
```

After editing `.env`, restart:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

Check what the backend container sees:

```bash
docker compose -f docker-compose.prod.yml exec backend env | grep -E 'FRONTEND_URL|CORS_ORIGINS'
```

## Step 21: Final Production Checklist

Backend health works inside EC2:

```bash
curl http://127.0.0.1:8000/health
```

Backend health works through HTTPS:

```bash
curl https://companyaiass.ddns.net/health
```

Docker services are up:

```bash
docker compose -f docker-compose.prod.yml ps
```

Ollama models are installed:

```bash
docker exec company-ai-ollama ollama list
```

Vercel has:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

EC2 Security Group allows:

```text
80
443
22 from your IP only
```

EC2 Security Group does not publicly allow:

```text
8000
11434
5432
```

## Step 22: How To Update The Backend Later

On your local computer:

```bash
git status
git add backend docker-compose.prod.yml docs
git commit -m "Update backend"
git push origin main
```

Only commit files you intentionally changed.

Do not commit:

```text
.env
backend/.env
frontend/.env
```

Then SSH into EC2:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Go to the project:

```bash
cd ~/apps/company-ai-assistant
```

Pull the latest code:

```bash
git pull origin main
```

Rebuild and restart:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

Check logs:

```bash
docker compose -f docker-compose.prod.yml logs --tail=100 backend
```

Check health:

```bash
curl http://127.0.0.1:8000/health
curl https://companyaiass.ddns.net/health
```

## Useful Commands

Show running containers:

```bash
docker ps
```

Show production Compose services:

```bash
docker compose -f docker-compose.prod.yml ps
```

View backend logs:

```bash
docker compose -f docker-compose.prod.yml logs --tail=100 backend
```

Follow backend logs live:

```bash
docker compose -f docker-compose.prod.yml logs -f backend
```

View Ollama logs:

```bash
docker compose -f docker-compose.prod.yml logs --tail=100 ollama
```

Restart the production stack:

```bash
docker compose -f docker-compose.prod.yml restart
```

Stop the production stack:

```bash
docker compose -f docker-compose.prod.yml down
```

Start it again:

```bash
docker compose -f docker-compose.prod.yml up -d
```

Rebuild after code changes:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

Check backend container environment:

```bash
docker compose -f docker-compose.prod.yml exec backend env | grep -E 'DATABASE_URL|FRONTEND_URL|CORS_ORIGINS|OLLAMA'
```

Be careful: this can print secrets. Do not share the output publicly.

## Common Problems

### `curl http://127.0.0.1:8000/health` Fails On EC2

Check containers:

```bash
docker compose -f docker-compose.prod.yml ps
```

Check backend logs:

```bash
docker compose -f docker-compose.prod.yml logs --tail=100 backend
```

Common causes:

- Backend container failed to start.
- `.env` is missing.
- `DATABASE_URL` is wrong.
- Database does not allow EC2 to connect.

### `curl http://your-domain/health` Fails

Check:

- DNS points to the EC2 public IP.
- EC2 Security Group allows port `80`.
- Nginx is running.
- Nginx config has the correct `server_name`.

Commands:

```bash
sudo systemctl status nginx
sudo nginx -t
curl http://127.0.0.1:8000/health
```

### HTTPS Fails

Check:

- EC2 Security Group allows port `443`.
- Certbot finished successfully.
- DNS points to the correct EC2 IP.
- Nginx config is valid.

Commands:

```bash
sudo nginx -t
sudo certbot certificates
curl https://companyaiass.ddns.net/health
```

### Vercel Frontend Still Calls `localhost`

Check Vercel environment variable:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

Then redeploy the frontend.

Why:

- Next.js public environment variables are baked into the frontend build.
- Changing the variable requires a new deployment.

### Browser Shows A CORS Error

Check the EC2 `.env` file:

```env
CORS_ORIGINS=https://your-vercel-app.vercel.app
```

If you changed it, restart:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

### Browser Shows A Mixed Content Error

This usually means the frontend is HTTPS but the backend URL is HTTP.

Use:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

Do not use:

```env
NEXT_PUBLIC_API_BASE_URL=http://companyaiass.ddns.net
```

### Ollama Model Missing

Pull the models:

```bash
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```

Check:

```bash
docker exec company-ai-ollama ollama list
```
