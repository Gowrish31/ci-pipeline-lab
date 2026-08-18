# Cloud Run Pipeline Mapping

This document details the 1-to-1 mapping between the local GitHub Actions pipeline (using Docker Hub and Docker Compose) and a production Google Cloud Run CI/CD pipeline using Google Cloud Artifact Registry and `gcloud run deploy`.

---

## 1. Overview & Conceptual Architecture

The structure and sequencing of the CI/CD pipeline remain identical regardless of the target runtime:

```text
[Commit / Git Push]
        │
        ▼
   [ Checkout ]
        │
        ▼
  [ Authenticate ]  ──▶ GitHub Secrets / GCP Auth
        │
        ▼
  [ Build & Tag ]   ──▶ Immutable Commit SHA Tag
        │
        ▼
     [ Push ]       ──▶ Container Registry
        │
        ▼
    [ Deploy ]      ──▶ Target Runtime & Health Check
```

---

## 2. Stage-by-Stage Mapping Table

| Pipeline Stage | Local Target (This Lab) | Cloud Run Target (Production Mapping) |
| :--- | :--- | :--- |
| **Registry** | Docker Hub (`<username>/app:<sha>`) | GCP Artifact Registry (`<region>-docker.pkg.dev/<project-id>/<repository>/app:<sha>`) |
| **Authentication** | `docker/login-action` using secrets `DOCKERHUB_USER` & `DOCKERHUB_TOKEN` | `google-github-actions/auth` using GCP Workload Identity or Service Account Key (`GCP_SA_KEY`) |
| **Build & Tag** | `docker/build-push-action` tagged with `${{ github.sha }}` | `docker/build-push-action` tagged with `${{ github.sha }}` targeting Artifact Registry URL |
| **Push** | `docker push <username>/app:<sha>` | `docker push <region>-docker.pkg.dev/<project-id>/<repository>/app:<sha>` |
| **Deploy** | Local `docker compose up -d` | `gcloud run deploy service-name --image ... --region ... --platform managed` (or `google-github-actions/deploy-cloudrun`) |
| **Proof / Health Check** | `curl -f http://localhost:8080/health` (HTTP 200) | `curl -f https://<cloud-run-service-url>/health` (HTTP 200) |

---

## 3. GitHub Actions Workflow Mapping (Cloud Run Variant)

Below is the equivalent Cloud Run production workflow YAML:

```yaml
name: Build and Deploy to Cloud Run

on:
  push:
    branches:
      - main

env:
  GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  GCP_REGION: us-central1
  ARTIFACT_REGISTRY_REPO: my-repo
  SERVICE_NAME: my-app-service

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker for Artifact Registry
        run: |
          gcloud auth configure-docker ${{ env.GCP_REGION }}-docker.pkg.dev

      - name: Build and Push Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.GCP_REGION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ env.ARTIFACT_REGISTRY_REPO }}/app:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Deploy to Cloud Run
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: ${{ env.SERVICE_NAME }}
          region: ${{ env.GCP_REGION }}
          image: ${{ env.GCP_REGION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ env.ARTIFACT_REGISTRY_REPO }}/app:${{ github.sha }}

      - name: Verify Service Health
        run: |
          SERVICE_URL=$(gcloud run services describe ${{ env.SERVICE_NAME }} --region ${{ env.GCP_REGION }} --format 'value(status.url)')
          curl -f "${SERVICE_URL}/health"
```

---

## 4. Key Takeaways

1. **Decoupled Runtimes**: Automating image build, immutable SHA tagging, authentication, and deployment produces a universal pipeline pattern.
2. **Zero Code Changes**: The container application code (`app/server.js`) and `Dockerfile` remain completely identical between local Compose and Google Cloud Run.
3. **Deterministic Deployments**: By adding explicit job dependency (`needs: build-and-push`), deployment is guaranteed never to attempt pulling an unpushed image manifest.
