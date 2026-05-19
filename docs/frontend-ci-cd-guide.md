# Frontend CI/CD Guide

This guide explains the frontend CI/CD setup for this project step by step.

## What CI/CD Means

CI means Continuous Integration.

In this project, CI means GitHub automatically checks the frontend whenever you push code or open a pull request.

It checks:

- Can dependencies install successfully?
- Does TypeScript pass?
- Does ESLint pass?
- Can Next.js build the frontend?

CD means Continuous Deployment.

In this project, CD is handled by Vercel. After you connect the GitHub repository to Vercel, Vercel can automatically deploy the frontend whenever you push to GitHub.

## Files Used For Frontend CI/CD

The main CI file is:

```text
.github/workflows/frontend-ci.yml
```

GitHub Actions looks inside the `.github/workflows` folder automatically.

Any `.yml` file inside that folder can define an automation workflow.

This project also uses:

```text
frontend/package.json
frontend/eslint.config.mjs
```

`package.json` defines commands like `npm run build`.

`eslint.config.mjs` tells ESLint how to check the Next.js frontend code.

## How To Create The Workflow Manually

From the project root, create these folders:

```bash
mkdir -p .github/workflows
```

Then create this file:

```text
.github/workflows/frontend-ci.yml
```

Add this content:

```yaml
name: Frontend CI

on:
  pull_request:
    paths:
      - "frontend/**"
      - ".github/workflows/frontend-ci.yml"
  push:
    branches:
      - main
    paths:
      - "frontend/**"
      - ".github/workflows/frontend-ci.yml"

concurrency:
  group: frontend-ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    name: Build frontend
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: frontend

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Typecheck
        run: npm run typecheck

      - name: Lint
        run: npm run lint

      - name: Build
        run: npm run build
```

After that, commit and push the file to GitHub.

GitHub will detect the workflow automatically.

## Explanation Of The Workflow

```yaml
name: Frontend CI
```

This is the name shown in the GitHub Actions tab.

```yaml
on:
```

This section tells GitHub when to run the workflow.

```yaml
pull_request:
```

Run the workflow when someone opens or updates a pull request.

```yaml
push:
  branches:
    - main
```

Run the workflow when code is pushed to the `main` branch.

```yaml
paths:
  - "frontend/**"
  - ".github/workflows/frontend-ci.yml"
```

Only run this workflow when frontend files or the workflow file change.

This avoids running frontend checks when you only change backend files or docs.

```yaml
concurrency:
  group: frontend-ci-${{ github.ref }}
  cancel-in-progress: true
```

If you push several times quickly, GitHub cancels the older run and keeps only the newest one.

This saves time.

```yaml
jobs:
```

A workflow contains one or more jobs.

This workflow has one job called `build`.

```yaml
runs-on: ubuntu-latest
```

The job runs on a fresh Linux machine provided by GitHub.

```yaml
defaults:
  run:
    working-directory: frontend
```

All `run` commands start inside the `frontend` folder.

That matters because this repo has both frontend and backend folders.

```yaml
- name: Checkout
  uses: actions/checkout@v4
```

Downloads your repository code into the GitHub Actions machine.

Without this step, the workflow has no project files to build.

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
```

Installs Node.js on the GitHub Actions machine.

```yaml
node-version: 22
```

Use Node.js version 22.

```yaml
cache: npm
cache-dependency-path: frontend/package-lock.json
```

Cache npm packages based on the frontend lock file.

This can make future CI runs faster.

```yaml
- name: Install dependencies
  run: npm ci
```

Installs dependencies exactly from `package-lock.json`.

For CI, `npm ci` is preferred over `npm install` because it is stricter and more repeatable.

```yaml
- name: Typecheck
  run: npm run typecheck
```

Runs TypeScript checks.

This catches type errors before deployment.

```yaml
- name: Lint
  run: npm run lint
```

Runs ESLint.

This catches common code quality and React/Next.js issues.

```yaml
- name: Build
  run: npm run build
```

Runs the production Next.js build.

If this passes, the frontend is much more likely to deploy successfully on Vercel.

## Frontend Package Scripts

Inside `frontend/package.json`, these scripts are used by CI:

```json
{
  "scripts": {
    "build": "next build",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit"
  }
}
```

`next build` creates the production frontend build.

`eslint .` checks the code style and common mistakes.

`tsc --noEmit` checks TypeScript without creating output files.

## How Vercel CD Fits In

GitHub Actions CI checks whether the frontend is healthy.

Vercel CD deploys the frontend.

Recommended Vercel settings:

```text
Root Directory: frontend
Framework Preset: Next.js
Install Command: npm install
Build Command: npm run build
Output Directory: default
```

When the AWS backend is ready, add this environment variable in Vercel:

```text
NEXT_PUBLIC_API_BASE_URL=https://your-aws-backend-url.com
```

Before the backend is ready, the frontend can still deploy, but upload/chat API features will not work in production.

## Typical Workflow

A normal workflow looks like this:

1. Change frontend code locally.
2. Run checks locally:

```bash
cd frontend
npm run typecheck
npm run lint
npm run build
```

3. Commit the code:

```bash
git add .
git commit -m "Update frontend"
```

4. Push to GitHub:

```bash
git push
```

5. Open GitHub and check the Actions tab.
6. If CI passes, Vercel can deploy the frontend.

## Important Notes

CI does not replace testing locally.

CI is a safety net that runs again on GitHub.

If GitHub Actions fails, click the failed workflow, open the failed step, read the error message, fix the code locally, then push again.

Vercel deployment and GitHub Actions CI are separate systems:

- GitHub Actions checks the code.
- Vercel deploys the frontend.

For this project, that is enough for a simple and clean frontend CI/CD setup.
