# EC2 Backend Compose Guide

This guide starts after you have already connected to the EC2 instance with SSH.
It assumes the EC2 instance is running Ubuntu.

The goal is to run only the backend services on EC2:

```txt
EC2
  Docker Compose
    FastAPI backend
    Ollama

Vercel
  Next.js frontend

Managed Postgres
  Neon, Supabase, RDS, or another Postgres database with pgvector
```

The frontend is not started on EC2. The frontend should be deployed separately
to Vercel.

## 1. Update The Server

Run:

```bash
sudo apt update
sudo apt upgrade -y
```

This updates the package list and installs available security updates.

## 2. Install Git

Run:

```bash
sudo apt install -y git
```

Check that Git is installed:

```bash
git --version
```

## 3. Install Docker

Run:

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Add the Docker package source:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker and Docker Compose:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Check that Docker works:

```bash
docker --version
docker compose version
```

## 4. Allow Your User To Run Docker

Run:

```bash
sudo usermod -aG docker $USER
```

Then log out of SSH and connect again.

After reconnecting, test Docker without `sudo`:

```bash
docker ps
```

If it works, continue. If it says permission denied, reconnect to SSH again.

## 5. Clone The Repository

Choose a folder for the project:

```bash
mkdir -p ~/apps
cd ~/apps
```

Clone the GitHub repository:

```bash
git clone <YOUR_GITHUB_REPO_URL> company-ai-assistant
cd company-ai-assistant
```

Example:

```bash
git clone https://github.com/YOUR_USERNAME/company-ai-assistant.git company-ai-assistant
cd company-ai-assistant
```

Check the files:

```bash
ls
```

You should see files like:

```txt
README.md
docker-compose.prod.yml
backend
frontend
docs
```

## 6. Create The Backend Environment File

Create the real `.env` file in the project root:

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

Important:

- The real `.env` belongs in the project root, next to `docker-compose.prod.yml`.
- Do not put the real production backend env in `frontend/.env`.
- Do not commit `.env` to GitHub.
- `FRONTEND_URL` must be your real Vercel frontend URL.
- `CORS_ORIGINS` should include the frontend origins allowed to call the
  backend from a browser. For one Vercel app, set it to the same value as
  `FRONTEND_URL`.
- `MIN_RELEVANCE_SCORE` controls when the assistant refuses questions that are
  not covered by the uploaded PDFs.
- `DATABASE_URL` must point to your managed Postgres database.

Save and exit Nano:

```txt
Ctrl + O
Enter
Ctrl + X
```

Check that the file exists:

```bash
ls -la .env
```

Do not print the full `.env` in screenshots or chat messages because it contains
database secrets.

## 7. Start The Backend Stack

From the project root, run:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

This starts:

```txt
company-ai-backend
company-ai-ollama
```

Check the containers:

```bash
docker compose -f docker-compose.prod.yml ps
```

Expected result:

```txt
company-ai-backend   Up
company-ai-ollama    Up
```

## 8. Check Backend Logs

Run:

```bash
docker logs company-ai-backend --tail=100
```

Good logs look like this:

```txt
Application startup complete.
Uvicorn running on http://0.0.0.0:8000
```

If the backend exits or shows database errors, check the `DATABASE_URL` value in
the root `.env` file.

## 9. Test The Backend From EC2

Run:

```bash
curl http://127.0.0.1:8000/health
```

Expected result:

```json
{"status":"ok"}
```

You can also test the API docs HTML:

```bash
curl http://127.0.0.1:8000/docs
```

## 10. Optional Temporary Public IP Test

For production, the backend should stay behind HTTPS with Caddy or Nginx.
However, during early testing you may want to check the backend from your own
browser using the EC2 public IP.

This only works if both are true:

- The EC2 Security Group allows inbound TCP port `8000`.
- `docker-compose.prod.yml` exposes the backend publicly with `8000:8000`.

Temporary Compose port mapping:

```yaml
ports:
  - "8000:8000"
```

Then restart:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

Open these URLs in your browser, replacing the IP with your EC2 public IPv4:

```txt
http://44.201.44.167:8000/health
http://44.201.44.167:8000/docs

Example domain test:

http://107.23.86.2:8000/health
http://companyaiass.ddns.net:8000/health
```

Expected health response:

```json
{"status":"ok"}
```

Important:

- Do not use this as the final production setup.
- Do not expose port `8000` publicly long term.
- The Vercel frontend uses HTTPS, so browsers may block calls to plain HTTP
  backend URLs as mixed content.
- For the final setup, use `https://companyaiass.ddns.net` through Nginx.

## 11. Point A No-IP Domain To EC2

If you use a free No-IP hostname, create an `A` record that points to the EC2
public IPv4 address.

Example:

```txt
companyaiass.ddns.net -> 107.23.86.2
```

In No-IP:

```txt
My No-IP -> Dynamic DNS -> Records -> Add Record
```

Use:

```txt
Type: A
Host: companyaiass
Domain: ddns.net
IPv4: your EC2 public IPv4 address
TTL: 60 seconds
```

Why this is needed:

- A domain name is only a human-friendly name.
- The browser still needs to know which server IP address to connect to.
- The No-IP `A` record connects the name to the EC2 public IP.

Check your EC2 public IP in AWS:

```txt
EC2 -> Instances -> select instance -> Public IPv4 address
```

The No-IP IPv4 value must match the EC2 public IPv4 value.

Test DNS from your EC2 terminal or local terminal:

```bash
ping companyaiass.ddns.net
```

Or test the temporary direct backend port if port `8000` is still open:

```bash
curl http://companyaiass.ddns.net:8000/health
```

Expected result:

```json
{"status":"ok"}
```

Important:

- A normal EC2 public IP can change after stopping and starting the instance.
- For a stable production setup, use an AWS Elastic IP and point No-IP to that
  Elastic IP.
- Free No-IP accounts may allow only one hostname. Delete old hostnames you no
  longer use so the active one can stay enabled.

## 12. Configure The EC2 Security Group

The EC2 Security Group is the firewall in front of your instance. If a port is
not allowed here, outside users cannot reach that port even if the app is
running correctly on the server.

During early testing, you may temporarily allow FastAPI directly:

```txt
Custom TCP   8000   0.0.0.0/0
```

This lets you test:

```txt
http://companyaiass.ddns.net:8000/health
```

For the final Nginx setup, allow:

```txt
HTTP    TCP   80    0.0.0.0/0
HTTPS   TCP   443   0.0.0.0/0
```

If you use SSH, keep SSH restricted to your own IP:

```txt
SSH     TCP   22    your-ip-address/32
```

If you use AWS Session Manager, you may not need public SSH at all.

After Nginx and HTTPS work, remove public access to:

```txt
Custom TCP   8000   0.0.0.0/0
```

Why remove port `8000`:

- Users should reach the backend through Nginx on ports `80` and `443`.
- Nginx can handle HTTPS certificates.
- Keeping FastAPI private reduces the public attack surface.

Do not publicly expose these ports:

```txt
8000   FastAPI direct backend port, except for temporary testing
11434  Ollama
5432   Postgres
```

## 13. Install Nginx On Ubuntu

Nginx is the public web server. It accepts browser traffic on port `80` and
later port `443`, then forwards requests to FastAPI on port `8000`.

Install Nginx:

```bash
sudo apt update
sudo apt install nginx -y
```

Start Nginx and enable it after reboot:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Check status:

```bash
sudo systemctl status nginx
```

Good result:

```txt
Active: active (running)
```

Important beginner note:

- `http://companyaiass.ddns.net` is a browser URL, not a Linux command.
- If you type it directly in the terminal, the shell will say `not found`.
- To test a URL from the terminal, use `curl`.

Test Nginx from the terminal:

```bash
curl -i http://companyaiass.ddns.net
```

If Nginx is running and port `80` is open in the security group, you should see
an HTTP response. Before the proxy config is added, this may be the default
Nginx welcome page.

## 14. Create The Nginx Reverse Proxy Config

Create a new Nginx site config:

```bash
sudo nano /etc/nginx/sites-available/companyaiass
```

Paste this config:

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

Save and exit Nano:

```txt
Ctrl + O
Enter
Ctrl + X
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/companyaiass /etc/nginx/sites-enabled/companyaiass
```

Remove the default Nginx site:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Test the Nginx config:

```bash
sudo nginx -t
```

Expected result:

```txt
syntax is ok
test is successful
```

Reload Nginx:

```bash
sudo systemctl reload nginx
```

Test without port `8000`:

```bash
curl http://companyaiass.ddns.net/health
```

Expected result:

```json
{"status":"ok"}
```

What this proves:

- DNS points the domain to EC2.
- AWS allows port `80`.
- Nginx receives the request.
- Nginx forwards the request to FastAPI on `127.0.0.1:8000`.
- FastAPI returns the health response.

## 15. Add HTTPS With Certbot

At this point, HTTP works:

```txt
http://companyaiass.ddns.net/health
```

HTTPS will not work until you install a certificate:

```txt
https://companyaiass.ddns.net/health
```

If `curl https://companyaiass.ddns.net/health` fails with port `443`, that is
expected before this step.

First make sure the EC2 Security Group allows:

```txt
HTTPS   TCP   443   0.0.0.0/0
HTTP    TCP   80    0.0.0.0/0
```

Install Certbot:

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

Request a certificate and let Certbot update Nginx:

```bash
sudo certbot --nginx -d companyaiass.ddns.net
```

Certbot may ask:

```txt
Enter email address
Agree to terms
Share email with EFF
Redirect HTTP to HTTPS
```

Recommended answers:

```txt
Agree to terms: yes
Share email: no
Redirect HTTP to HTTPS: yes
```

Test Nginx again:

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

Check certificate auto-renewal:

```bash
sudo certbot renew --dry-run
```

Why this is needed:

- Vercel frontend pages use HTTPS.
- Browsers often block HTTPS frontend pages from calling HTTP backend APIs.
- Certbot gives Nginx a trusted Let's Encrypt certificate.
- Nginx can then serve your backend at a secure HTTPS URL.

## 16. Update Vercel Frontend Environment

After HTTPS works, update the Vercel frontend API base URL.

In Vercel:

```txt
Project -> Settings -> Environment Variables
```

Set:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

Do not use:

```env
NEXT_PUBLIC_API_BASE_URL=http://companyaiass.ddns.net:8000
```

Why:

- The frontend should call the public HTTPS URL.
- Users should not call the direct FastAPI port.
- The browser bundle reads `NEXT_PUBLIC_API_BASE_URL`.
- In Next.js, `NEXT_PUBLIC_` variables are baked into the frontend at build
  time.

After changing the Vercel variable, redeploy the frontend:

```txt
Vercel -> Deployments -> latest deployment -> Redeploy
```

If the frontend still calls `localhost` or the old URL, check:

- The variable name is exactly `NEXT_PUBLIC_API_BASE_URL`.
- The value is set for the correct Vercel environment, usually `Production`.
- The app was redeployed after changing the variable.

## 17. Final Production Access Pattern

Final request flow:

```txt
Browser
  -> https://companyaiass.ddns.net
  -> EC2 security group port 443
  -> Nginx
  -> http://127.0.0.1:8000
  -> FastAPI backend container
```

Final public URLs:

```txt
https://companyaiass.ddns.net/health
https://companyaiass.ddns.net/docs
```

Final EC2 Security Group:

```txt
HTTP    80    0.0.0.0/0
HTTPS   443   0.0.0.0/0
SSH     22    your IP only, if using SSH
```

Remove public port `8000` after verifying HTTPS works.

## 18. Pull The Ollama Models

Run:

```bash
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```

Use the same model names that you put in `.env`:

```env
OLLAMA_CHAT_MODEL=llama3.2:3b
OLLAMA_EMBEDDING_MODEL=nomic-embed-text
```

Check installed models:

```bash
docker exec company-ai-ollama ollama list
```

## 19. Understand The Public URL

The production Compose file maps the backend to EC2 localhost:

```txt
127.0.0.1:8000
```

That means this works only inside the EC2 instance:

```bash
curl http://127.0.0.1:8000/health
```

The Vercel frontend cannot call `127.0.0.1` on EC2. The browser needs a public
HTTPS backend URL.

For production, use a reverse proxy such as Caddy or Nginx:

```txt
https://companyaiass.ddns.net -> http://127.0.0.1:8000
```

Then set this in Vercel:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

Do not use `http://localhost:8000` in Vercel. In a user's browser, `localhost`
means the user's own computer, not the EC2 server.

## 20. Security Group Checklist

In the EC2 Security Group, allow:

```txt
22   SSH    from your IP only
80   HTTP   from anywhere, if using a web proxy
443  HTTPS  from anywhere, if using a web proxy
```

Do not publicly expose these for production:

```txt
8000   FastAPI direct backend port
11434  Ollama port
5432   Postgres port
```

## 21. Update The Backend Later

When you change backend or deployment code locally, first push the update to
GitHub:

```bash
git status
git add backend docker-compose.prod.yml docs
git commit -m "Describe the backend update"
git push origin main
```

Only commit the files you intentionally changed. Do not commit `.env` files.

Then SSH into EC2:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Go to the project directory. Use the path where you cloned the repo:

```bash
cd ~/apps/company-ai-assistant
```

Pull the newest code from GitHub:

```bash
git pull origin main
```

If Git says local changes would be overwritten, inspect them first:

```bash
git status
git diff -- docker-compose.prod.yml
```

If the local change is only an old server-only edit that should not override
GitHub, stash it:

```bash
git stash push -m "ec2 local docker compose change" docker-compose.prod.yml
git pull origin main
```

After the code is updated, rebuild and restart the backend stack:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

Check that the containers are running:

```bash
docker compose -f docker-compose.prod.yml ps
```

Check the backend logs:

```bash
docker compose -f docker-compose.prod.yml logs --tail=100 backend
```

Check the backend health from inside EC2:

```bash
curl http://127.0.0.1:8000/health
```

Confirm the backend container has the production frontend and CORS settings:

```bash
docker compose -f docker-compose.prod.yml exec backend env | grep -E 'FRONTEND_URL|CORS_ORIGINS|MIN_RELEVANCE_SCORE'
```

Expected shape:

```txt
FRONTEND_URL=https://your-vercel-app.vercel.app
CORS_ORIGINS=https://your-vercel-app.vercel.app
MIN_RELEVANCE_SCORE=0.35
```

If either value is wrong, edit the EC2 `.env` file, then rebuild again:

```bash
nano .env
docker compose -f docker-compose.prod.yml up -d --build
```

If the model names changed, pull the new models:

```bash
docker exec company-ai-ollama ollama pull <MODEL_NAME>
```

If you stashed an old EC2-only edit and the server works after the update, you
can leave the stash alone or remove it:

```bash
git stash list
git stash drop stash@{0}
```

## 22. Useful Commands

Show running containers:

```bash
docker ps
```

Show Compose services:

```bash
docker compose -f docker-compose.prod.yml ps
```

View backend logs:

```bash
docker logs company-ai-backend --tail=100
```

Follow backend logs live:

```bash
docker logs company-ai-backend -f
```

Restart the backend stack:

```bash
docker compose -f docker-compose.prod.yml restart
```

Stop the backend stack:

```bash
docker compose -f docker-compose.prod.yml down
```

Start it again:

```bash
docker compose -f docker-compose.prod.yml up -d
```

## 23. Common Problems

### Frontend Still Calls localhost

If browser DevTools shows:

```txt
http://localhost:8000
```

then the Vercel frontend does not have the correct production environment
variable.

In Vercel, set:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

Make sure it is enabled for the correct environment:

```txt
Production
```

Then redeploy the frontend.

### Mixed Content Error

If browser DevTools shows:

```txt
Mixed Content
```

then the Vercel frontend is using HTTPS but the backend URL is HTTP.

Use HTTPS for the backend:

```env
NEXT_PUBLIC_API_BASE_URL=https://companyaiass.ddns.net
```

### Backend Cannot Connect To Database

Check:

```bash
docker logs company-ai-backend --tail=100
```

Common causes:

- `DATABASE_URL` is wrong.
- The database provider blocks the EC2 IP.
- The database does not have `pgvector` enabled.
- The database user does not have permission to create extensions.

### Ollama Model Missing

If upload or chat fails because a model is missing, run:

```bash
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```
