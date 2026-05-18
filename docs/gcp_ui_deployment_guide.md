# Deploying a Static UI to Google Cloud Run
### A Step-by-Step Reference Guide

---

> **Scope:** This guide covers the end-to-end process of deploying a working local UI project (plain HTML/CSS/JS, React, Vue, or any SPA) to Google Cloud Run with a custom domain, free SSL, and automated CI/CD via GitHub Actions.
> 
> **Prerequisite:** You have working code locally, a GitHub repository, a GCP account with billing enabled, and a registered domain (e.g., in GoDaddy).

---

## Table of Contents

1. [One-Time Setup: Install & Authenticate gcloud](#1-one-time-setup-install--authenticate-gcloud)
2. [GCP Project Setup](#2-gcp-project-setup)
3. [Enable Required APIs](#3-enable-required-apis)
4. [Create Artifact Registry Repository](#4-create-artifact-registry-repository)
5. [Create a Service Account for CI/CD](#5-create-a-service-account-for-cicd)
6. [Containerize the App with Docker](#6-containerize-the-app-with-docker)
7. [Set Up GitHub Actions Workflow](#7-set-up-github-actions-workflow)
8. [Add GitHub Secrets](#8-add-github-secrets)
9. [Trigger the First Deployment](#9-trigger-the-first-deployment)
10. [Map a Custom Domain](#10-map-a-custom-domain)
11. [Update GoDaddy DNS Records](#11-update-godaddy-dns-records)
12. [Verify Everything is Live](#12-verify-everything-is-live)
13. [Quick Reference Cheat Sheet](#13-quick-reference-cheat-sheet)

---

## 1. One-Time Setup: Install & Authenticate gcloud

If you haven't installed the Google Cloud CLI yet:

```bash
# macOS (via Homebrew)
brew install google-cloud-sdk

# Or download the installer from:
# https://cloud.google.com/sdk/docs/install
```

Authenticate your local terminal with your Google account:

```bash
gcloud auth login
```

A browser window will open — log in with the same Google account that has access to GCP.

---

## 2. GCP Project Setup

Every deployment lives inside a **GCP Project**. Create a dedicated project per client or application.

```bash
# Create a new project
gcloud projects create PROJECT_ID --name="PROJECT_DISPLAY_NAME"
# Example:
gcloud projects create my-client-app --name="My Client App"

# Set it as your active project
gcloud config set project PROJECT_ID
```

> **Naming rules for PROJECT_ID:** lowercase letters, digits, and hyphens only. Must be globally unique across all of GCP.

**Link your billing account** (required to use Cloud Run):

```bash
# List your billing accounts
gcloud billing accounts list

# Link the billing account to the project
gcloud billing projects link PROJECT_ID \
  --billing-account=BILLING_ACCOUNT_ID
```

---

## 3. Enable Required APIs

Cloud Run, Artifact Registry, and Cloud Build APIs must be enabled:

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  --project=PROJECT_ID
```

> ⏳ This takes about 30–60 seconds. You only need to do this once per project.

---

## 4. Create Artifact Registry Repository

Artifact Registry stores your Docker images. Think of it as a private Docker Hub for your project.

```bash
gcloud artifacts repositories create REPO_NAME \
  --repository-format=docker \
  --location=us-central1 \
  --project=PROJECT_ID
# Example: --name naveenh-repo
```

**Recommended regions:** `us-central1` (cheapest & fastest for most use cases).

---

## 5. Create a Service Account for CI/CD

GitHub Actions needs a dedicated GCP identity to authenticate securely. Never use your personal credentials in CI/CD.

```bash
# 1. Create the service account
gcloud iam service-accounts create github-deployer \
  --display-name="GitHub Deployer" \
  --project=PROJECT_ID

# 2. Grant it the three required roles
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:github-deployer@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:github-deployer@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:github-deployer@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# 3. Create a JSON key and print it (copy the output!)
gcloud iam service-accounts keys create github-sa-key.json \
  --iam-account=github-deployer@PROJECT_ID.iam.gserviceaccount.com

cat github-sa-key.json

# 4. IMPORTANT: Delete the local key file after copying its contents
rm github-sa-key.json
```

> [!CAUTION]
> Never commit the `github-sa-key.json` file to Git. Delete it locally immediately after copying the JSON content. Add `*.json` to your `.gitignore`.

---

## 6. Containerize the App with Docker

Cloud Run runs Docker containers. You need two files: a `Dockerfile` and an `nginx.conf`.

### 6a. Create `nginx.conf`

This configures the Nginx web server running inside the container:

```nginx
server {
    listen 8080;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Compression for performance
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;

    # Cache static assets (JS, CSS, images) for 1 year
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Never cache HTML (so new deployments are seen immediately)
    location ~* \.html$ {
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    # SPA fallback — required for React/Vue Router
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

> **Note:** Cloud Run requires the container to listen on port **8080**. Never use port 80 or 443.

### 6b. Create `Dockerfile`

```dockerfile
FROM nginx:alpine

# Remove default Nginx config
RUN rm /etc/nginx/conf.d/default.conf

# Copy your custom Nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy your built site files into the Nginx web root
COPY . /usr/share/nginx/html

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
```

> **For React/Vue/Next.js apps:** Run `npm run build` first and copy only the `dist/` or `build/` folder:
> ```dockerfile
> COPY dist/ /usr/share/nginx/html
> ```

### 6c. Create `.dockerignore`

Prevents unnecessary files from being included in the Docker image:

```
.git
.github
node_modules
*.md
.DS_Store
.idea
*.json
*.log
```

---

## 7. Set Up GitHub Actions Workflow

Create the directory and workflow file:

```bash
mkdir -p .github/workflows
```

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Cloud Run

on:
  push:
    branches:
      - main

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  REGION: us-central1
  SERVICE_NAME: your-service-name    # ← Change this to your Cloud Run service name

jobs:
  deploy:
    name: Build & Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker for GCP
        run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev --quiet

      - name: Build and push Docker image
        run: |
          docker build -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/REPO_NAME/${{ env.SERVICE_NAME }}:${{ github.sha }} .
          docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/REPO_NAME/${{ env.SERVICE_NAME }}:${{ github.sha }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy ${{ env.SERVICE_NAME }} \
            --image ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/REPO_NAME/${{ env.SERVICE_NAME }}:${{ github.sha }} \
            --region ${{ env.REGION }} \
            --platform managed \
            --allow-unauthenticated \
            --port 8080 \
            --memory 256Mi \
            --cpu 1 \
            --min-instances 0 \
            --max-instances 3

      - name: Print service URL
        run: |
          gcloud run services describe ${{ env.SERVICE_NAME }} \
            --region ${{ env.REGION }} \
            --format="value(status.url)"
```

> Replace `REPO_NAME` with the Artifact Registry repo name you created in Step 4.

---

## 8. Add GitHub Secrets

Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

Add these two secrets:

| Secret Name | Value |
|:---|:---|
| `GCP_PROJECT_ID` | Your GCP Project ID (e.g., `my-client-app`) |
| `GCP_SA_KEY` | The **entire JSON content** of `github-sa-key.json` from Step 5 |

> [!IMPORTANT]
> Paste the **full raw JSON** (including the curly braces `{ }`) as the value of `GCP_SA_KEY`. A common mistake is pasting only part of it.

---

## 9. Trigger the First Deployment

Commit all new files and push to `main`. GitHub Actions will automatically pick it up.

```bash
git add .
git commit -m "Add Dockerfile, nginx config, and GitHub Actions workflow"
git push origin main
```

**Monitor the pipeline:**
- Go to your GitHub repository → **Actions** tab.
- You should see the `Deploy to Cloud Run` workflow running.
- The full pipeline (build + push + deploy) typically takes **2–5 minutes**.

Once completed, you will see the Cloud Run service URL printed in the logs, like:
```
https://your-service-name-1234567890.us-central1.run.app
```

Open that URL in a browser to verify the site is live.

---

## 10. Map a Custom Domain

Before you can map a custom domain, Google must verify you own it.

### 10a. Verify Domain Ownership

```bash
gcloud domains verify YOUR_DOMAIN.com
```

This opens Google Search Console in your browser. Follow the instructions to verify via a **TXT DNS record** added in GoDaddy (or your registrar). Verification is usually instant.

### 10b. Create Domain Mappings

Map the naked domain (`example.com`) and the `www` subdomain separately:

```bash
# Map the root domain
gcloud beta run domain-mappings create \
  --service your-service-name \
  --domain YOUR_DOMAIN.com \
  --region us-central1 \
  --project=PROJECT_ID

# Map the www subdomain
gcloud beta run domain-mappings create \
  --service your-service-name \
  --domain www.YOUR_DOMAIN.com \
  --region us-central1 \
  --project=PROJECT_ID
```

The output will print the DNS records you need to add (example):

```
NAME             RECORD TYPE  CONTENTS
naveenh-website  A            216.239.32.21
naveenh-website  A            216.239.34.21
...
www              CNAME        ghs.googlehosted.com.
```

---

## 11. Update GoDaddy DNS Records

Log in to GoDaddy → **My Products** → Click **DNS** next to your domain.

**Delete any old DNS records** (old Netlify `A` or `CNAME` records) before adding new ones.

### A Records (root domain)

Add these four `A` records pointing `@` (root) to Google's servers:

| Type | Name | Value | TTL |
|:---:|:---:|:---:|:---:|
| A | `@` | `216.239.32.21` | 1 Hour |
| A | `@` | `216.239.34.21` | 1 Hour |
| A | `@` | `216.239.36.21` | 1 Hour |
| A | `@` | `216.239.38.21` | 1 Hour |

### CNAME Record (www subdomain)

| Type | Name | Value | TTL |
|:---:|:---:|:---:|:---:|
| CNAME | `www` | `ghs.googlehosted.com.` | 1 Hour |

> [!NOTE]
> DNS changes can take anywhere from a few minutes to 48 hours to propagate globally, but typically happen within 5–30 minutes for GoDaddy.

**Google will automatically provision a free SSL certificate** for your domain once it detects the DNS records resolve correctly. No manual SSL setup is needed.

---

## 12. Verify Everything is Live

### Check DNS Propagation (Terminal)

```bash
# Should return Google's IP addresses (216.239.x.x)
dig YOUR_DOMAIN.com +short

# Should return ghs.googlehosted.com
dig www.YOUR_DOMAIN.com +short
```

### Check Domain Mapping Status (Terminal)

```bash
# Check if certificate is provisioned
gcloud run domain-mappings list --project=PROJECT_ID
```

When the status shows `CertificateProvisioned: True`, you are fully live with HTTPS!

### Check via Google Cloud Console (Browser)

Go to: **[Cloud Run > Domain Mappings](https://console.cloud.google.com/run/domains)**

A green checkmark ✅ next to your domain means SSL is active and your site is live.

---

## 13. Quick Reference Cheat Sheet

```bash
# ─── CREATE PROJECT ──────────────────────────────────────────────────────────
gcloud projects create PROJECT_ID --name="Display Name"
gcloud config set project PROJECT_ID
gcloud billing projects link PROJECT_ID --billing-account=BILLING_ID

# ─── ENABLE APIS ─────────────────────────────────────────────────────────────
gcloud services enable run.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com

# ─── ARTIFACT REGISTRY ───────────────────────────────────────────────────────
gcloud artifacts repositories create REPO_NAME --repository-format=docker --location=us-central1

# ─── SERVICE ACCOUNT ─────────────────────────────────────────────────────────
gcloud iam service-accounts create github-deployer --display-name="GitHub Deployer"
gcloud projects add-iam-policy-binding PROJECT_ID --member="serviceAccount:github-deployer@PROJECT_ID.iam.gserviceaccount.com" --role="roles/run.admin"
gcloud projects add-iam-policy-binding PROJECT_ID --member="serviceAccount:github-deployer@PROJECT_ID.iam.gserviceaccount.com" --role="roles/artifactregistry.writer"
gcloud projects add-iam-policy-binding PROJECT_ID --member="serviceAccount:github-deployer@PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountUser"
gcloud iam service-accounts keys create github-sa-key.json --iam-account=github-deployer@PROJECT_ID.iam.gserviceaccount.com && cat github-sa-key.json && rm github-sa-key.json

# ─── MANUAL DEPLOY (FIRST TIME / TESTING) ────────────────────────────────────
gcloud builds submit --tag us-central1-docker.pkg.dev/PROJECT_ID/REPO_NAME/SERVICE_NAME:manual
gcloud run deploy SERVICE_NAME --image us-central1-docker.pkg.dev/PROJECT_ID/REPO_NAME/SERVICE_NAME:manual --region us-central1 --platform managed --allow-unauthenticated --port 8080

# ─── DOMAIN MAPPING ──────────────────────────────────────────────────────────
gcloud domains verify YOUR_DOMAIN.com
gcloud beta run domain-mappings create --service SERVICE_NAME --domain YOUR_DOMAIN.com --region us-central1
gcloud beta run domain-mappings create --service SERVICE_NAME --domain www.YOUR_DOMAIN.com --region us-central1

# ─── STATUS CHECK ────────────────────────────────────────────────────────────
gcloud run domain-mappings list --project=PROJECT_ID
dig YOUR_DOMAIN.com +short
```

---

## Architecture Overview

```
Developer pushes to main (GitHub)
         │
         ▼
GitHub Actions Workflow (.github/workflows/deploy.yml)
    1. Authenticate to GCP (using GCP_SA_KEY secret)
    2. docker build → docker push → Artifact Registry
    3. gcloud run deploy → Cloud Run (us-central1)
         │
         ▼
Cloud Run Service (naveenh-website)
  ├── Auto-scales: 0 to 3 instances
  ├── Serves: nginx:alpine container on port 8080
  └── HTTPS endpoint: *.run.app
         │
         ▼ (domain mapping)
Custom Domain (YOUR_DOMAIN.com)
  ├── GoDaddy A records → Google Load Balancer
  ├── Free SSL Certificate (auto-provisioned by Google)
  └── Live at: https://YOUR_DOMAIN.com ✅
```

---

*Last updated: May 2026 | Works with Google Cloud SDK v568+*
