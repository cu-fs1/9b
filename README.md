# Next.js Static Export — Docker, GHCR & GitHub Actions CI/CD

A step-by-step guide for students to build a Next.js app, containerize it with Docker, and ship it automatically using GitHub Actions.

By the end you will have:
- A **Next.js** static site served by **Nginx** inside Docker
- A **CI/CD pipeline** that tests every PR and publishes a Docker image to **GHCR** on every push to `main`
- **Slack notifications** on each deployment

---

## Table of Contents

1. [What You Will Build](#what-you-will-build)
2. [Tech Stack](#tech-stack)
3. [Prerequisites](#prerequisites)
4. [Step 1 — Create a Next.js App](#step-1--create-a-nextjs-app)
5. [Step 2 — Configure Static Export](#step-2--configure-static-export)
6. [Step 3 — Dockerize the App](#step-3--dockerise-the-app)
7. [Step 4 — Run Locally with Docker](#step-4--run-locally-with-docker)
8. [Step 5 — Set Up GitHub Actions CI/CD](#step-5--set-up-github-actions-cicd)
9. [Step 6 — Set Up Slack Notifications](#step-6--set-up-slack-notifications)
10. [Project Structure](#project-structure)

---

## What You Will Build

A single-page Next.js app that is exported as pure static HTML/CSS/JS and served by a rootless Nginx container. No Node.js runtime is needed after the build. GitHub Actions handles testing, building, and pushing the Docker image to GitHub Container Registry automatically.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16 (App Router, static export) |
| Styling | Tailwind CSS v4 |
| Language | TypeScript |
| Package manager | pnpm |
| Container runtime | Docker (multi-stage build) |
| Web server | Nginx (nginx-unprivileged, rootless) |
| Registry | GitHub Container Registry (GHCR) |
| CI/CD | GitHub Actions |
| Notifications | Slack Incoming Webhooks |

---

## Prerequisites

Install these before you begin:

- [Node.js 24+](https://nodejs.org/)
- [pnpm](https://pnpm.io/installation) — `npm install -g pnpm`
- [Docker](https://docs.docker.com/get-docker/)
- A [GitHub](https://github.com) account with a new **empty** repository created

---

## Step 1 — Create a Next.js App

Scaffold a new Next.js project with pnpm:

```bash
pnpm create next-app@latest my-app --typescript --tailwind --eslint --app --src-dir no --import-alias "@/*"
cd my-app
```

Start the dev server to verify everything works:

```bash
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000).

### Available scripts

| Command | What it does |
|---|---|
| `pnpm dev` | Dev server with hot reload |
| `pnpm build` | Generates static export to `/out` |
| `pnpm lint` | Runs ESLint |
| `pnpm test:ci` | Lint + build (used by CI) |

Add the CI script to your `package.json`:

```json
"scripts": {
  "test:ci": "pnpm lint && pnpm build"
}
```

---

## Step 2 — Configure Static Export

Open `next.config.ts` and set:

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "export",
  images: {
    unoptimized: true,
  },
};

export default nextConfig;
```

`output: "export"` tells Next.js to write plain HTML/CSS/JS to `/out` instead of starting a Node server.

---

## Step 3 — Dockerise the App

### 3a — Create the Dockerfile

Create a `Dockerfile` in the project root with three stages:

```dockerfile
ARG NODE_VERSION=24.13.0-slim
ARG NGINXINC_IMAGE_TAG=alpine3.22

# Stage 1: install dependencies
FROM node:${NODE_VERSION} AS dependencies
WORKDIR /app
COPY package.json pnpm-lock.yaml* ./
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    corepack enable pnpm && pnpm install --frozen-lockfile

# Stage 2: build
FROM node:${NODE_VERSION} AS builder
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
ENV NODE_ENV=production
RUN --mount=type=cache,target=/app/.next/cache \
    corepack enable pnpm && pnpm build

# Stage 3: serve with Nginx
FROM nginxinc/nginx-unprivileged:${NGINXINC_IMAGE_TAG} AS runner
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=builder /app/out /usr/share/nginx/html
USER nginx
EXPOSE 8080
ENTRYPOINT ["nginx", "-c", "/etc/nginx/nginx.conf"]
CMD ["-g", "daemon off;"]
```

**Why three stages?** Each stage only carries what it needs into the next. The final image contains only Nginx and your static files — no Node.js, no source code.

### 3b — Create nginx.conf

```nginx
worker_processes 1;
pid /tmp/nginx.pid;

events { worker_connections 1024; }

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile on;

    server {
        listen 8080;
        root /usr/share/nginx/html;
        index index.html;

        location / {
            try_files $uri $uri.html $uri/ =404;
        }

        location ~ ^/_next/ {
            try_files $uri =404;
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        error_page 404 /404.html;
    }
}
```

### 3c — Create compose.yml

```yaml
services:
  nextjs-static-export:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    restart: unless-stopped
```

---

## Step 4 — Run Locally with Docker

```bash
docker compose up -d --build
```

Open [http://localhost:8080](http://localhost:8080). You should see your app served by Nginx.

```bash
# Stop the container
docker compose down
```

---

## Step 5 — Set Up GitHub Actions CI/CD

### 5a — Push your code to GitHub

```bash
git init
git add .
git commit -m "initial commit"
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

### 5b — Create the workflow file

Create `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD

on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [main]

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 10
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm test:ci

  docker:
    name: Docker Publish
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - id: image
        run: echo "image=ghcr.io/${GITHUB_REPOSITORY,,}" >> "$GITHUB_OUTPUT"
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.image.outputs.image }}
          tags: |
            type=raw,value=latest
            type=sha,format=long
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
```

### 5c — What the pipeline does

```
On every PR:
  test job → pnpm lint + pnpm build

On push to main:
  test job → docker job
                ├── Login to GHCR
                ├── Build Docker image
                └── Push with two tags:
                      ghcr.io/<owner>/<repo>:latest
                      ghcr.io/<owner>/<repo>:sha-<commit>
```

No extra secrets are needed — the workflow uses the built-in `GITHUB_TOKEN` to push to GHCR.

---

## Step 6 — Set Up Slack Notifications

The pipeline sends a [Block Kit](https://docs.slack.dev/block-kit) message to Slack after every deployment. Follow the [official Slack docs](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/) for full details.

### 6a — Create a Slack app

1. Go to [https://api.slack.com/apps](https://api.slack.com/apps) → **Create New App → From scratch**
2. Name it (e.g. `my-app-ci`) and select your workspace → **Create App**

### 6b — Enable Incoming Webhooks

1. In your app settings, select **Incoming Webhooks** from the left sidebar
2. Toggle **Activate Incoming Webhooks** to **On**
3. Click **Add New Webhook to Workspace**, pick a channel, then click **Allow**
4. Copy the URL — it looks like:
   ```
   https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
   ```
   > **Keep it secret.** Never commit this URL to version control. Slack actively revokes leaked secrets.

### 6c — Add the secret to GitHub

1. Repo → **Settings → Secrets and variables → Actions → New repository secret**
2. Name: `SLACK_WEBHOOK_URL`
3. Value: paste the webhook URL

### 6d — Add the notification step to your workflow

Append this step inside the `docker` job, after the build-push step:

```yaml
      - name: Slack notification
        if: always()
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          STATUS: ${{ job.status }}
          IMAGE: ${{ steps.image.outputs.image }}
          SHA: ${{ github.sha }}
          RUN_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        run: |
          if [ -z "$SLACK_WEBHOOK_URL" ]; then
            echo "SLACK_WEBHOOK_URL secret not set, skipping."
            exit 0
          fi
          payload=$(cat <<EOF
          {
            "text": ":rocket: CI/CD ${STATUS} — ${{ github.repository }}",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": ":rocket: *CI/CD ${STATUS}*\n*Repo:* ${{ github.repository }}\n*Image:* ${IMAGE}\n*Commit:* ${SHA}\n*Run:* <${RUN_URL}|View run>"
                }
              }
            ]
          }
          EOF
          )
          curl -sS -X POST -H "Content-Type: application/json" \
            --data "$payload" \
            "$SLACK_WEBHOOK_URL"
```

When it works, you will see a message like this in your channel:

![Slack notification showing CI/CD success with image and commit details](docs/slack-notification.png)

---

## Project Structure

```
my-app/
├── .github/
│   └── workflows/
│       └── ci-cd.yml        # GitHub Actions pipeline
├── app/
│   ├── globals.css          # Global Tailwind styles
│   ├── layout.tsx           # Root layout
│   └── page.tsx             # Landing page
├── public/                  # Static assets
├── Dockerfile               # Multi-stage Docker build
├── compose.yml              # Docker Compose for local testing
├── nginx.conf               # Nginx config (rootless, port 8080)
├── next.config.ts           # Static export config
├── package.json             # Scripts and dependencies
└── pnpm-lock.yaml           # Locked dependency tree
```
