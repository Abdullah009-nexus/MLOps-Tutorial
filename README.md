# MLOps Continuous Delivery Demo

This project follows the MLOps Continuous Delivery tutorial and implements a small ML inference API that is packaged in Docker and delivered through a GitHub Actions pipeline.

## Project structure

- `app.py` — Flask service with `/`, `/health`, and `/predict` endpoints
- `requirements.txt` — Python dependencies
- `Dockerfile` — builds the app image
- `compose.yaml` — local Docker Compose setup
- `VERSION` — semantic version of the application
- `tests/test_app.py` — automated endpoint tests
- `.github/workflows/cd.yml` — CI/CD workflow for tag-triggered delivery

---

## Step 3 — Demo ML application

The app is a simple Flask API that exposes:

- `/` — service metadata
- `/health` — health check returning the model version
- `/predict` — dummy ML inference using a simple value * 2 calculation

Why it matters:

- This is the smallest practical ML service for teaching delivery workflows.
- It lets us test health and inference behavior before packaging and deploying it.

---

## Step 4 — Tests

The tests validate the API behavior:

- `/health` should return status `healthy`
- `/predict` with JSON `{"value": 5}` should return `10`

This is the quality gate before the artifact is released.

---

## Step 6 — Containerize the app

The Dockerfile installs dependencies and runs the Flask app on port 5000:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

How it works:

- Python environment is prepared in a clean container.
- App code and dependencies are copied inside the image.
- The container exposes port 5000 so the API can be reached from outside.
- `CMD ["python", "app.py"]` starts the service when the container runs.

Local test commands:

```bash
docker build -t mlops-cd-demo:local .
docker run --rm -p 5000:5000 mlops-cd-demo:local
curl http://localhost:5000/health
```

---

## Step 8 — Version the application

The `VERSION` file contains the semantic version:

```text
1.0.0
```

This is tied to the container artifact so that every release is traceable and reproducible.

Example version mapping:

- `v1.0.0` -> `ghcr.io/<user>/mlops-cd-demo:1.0.0`
- `v1.1.0` -> `ghcr.io/<user>/mlops-cd-demo:1.1.0`

Why this matters:

- Production deployments should use immutable image versions.
- You can rebuild and redeploy the same release without ambiguity.

---

## Step 9 — GitHub Container Registry (GHCR)

The workflow uses GHCR as the release artifact store.

- Build the Docker image
- Tag it with the semantic version
- Push it to GHCR
- Use that exact image for staging and production deployment

This follows the build-once, deploy-many approach.

---

## Current tutorial stage: Step 15–21 — GitHub Environments and deployment gates

This is the next stage in the tutorial after the base app, Docker image, and versioned artifact flow.

### Step 15: GitHub Environments

Create two GitHub environments in the repository settings:

- `staging`
- `production`

Why this matters:

- Staging and production can each have different access rules.
- Production can require a human approval before deployment.
- Environment-specific secrets keep credentials isolated per target.

### Step 16–17: Prepare the deployment server

A Linux server with Docker is needed. The tutorial uses a VM or Ubuntu server and SSH keys for remote deployment.

Typical steps:

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
ssh-keygen -t ed25519 -f deploy_key
ssh-copy-id -i deploy_key.pub user@SERVER_IP
```

This gives GitHub Actions a secure way to reach the deployment host and start the app containers.

### Step 18: Add deployment secrets

The workflow expects secrets such as:

- `STAGING_HOST`
- `STAGING_USER`
- `STAGING_SSH_KEY`
- `PRODUCTION_HOST`
- `PRODUCTION_USER`
- `PRODUCTION_SSH_KEY`

These must be stored in GitHub repository secrets rather than committed in YAML files.

### Step 19–20: Deploy to staging and smoke test

The workflow pulls the exact tagged image for the current version and runs it remotely:

```bash
docker pull ghcr.io/<repo>:<version>
docker stop mlops-api || true
docker rm mlops-api || true
docker run -d --name mlops-api --restart unless-stopped -p 5000:5000 ghcr.io/<repo>:<version>
```

Then a smoke test checks the health endpoint:

```bash
curl --fail http://<host>:5000/health
```

If the health check fails, the deployment is considered broken and the pipeline stops before production.

### Step 21: Production approval gate

The production deployment is linked to a GitHub environment with required reviewers configured in the repository settings.

This is the key difference between continuous delivery and continuous deployment:

- staging deployment is automatic
- production deployment waits for review
- once approved, the same version is promoted to production

This guarantees that only a tested artifact reaches production.

---

## Step 10–14 — Continuous Delivery workflow

The workflow file at `.github/workflows/cd.yml` triggers on semantic version tags such as:

```bash
git tag v1.0.0
git push origin v1.0.0
```

The pipeline runs in this order:

1. `test` job
   - checks out the repository
   - installs dependencies
   - runs `pytest`
2. `build` job
   - extracts the version without the `v` prefix
   - logs into GHCR
   - builds the image
   - pushes both versioned and `latest` tags
3. `deploy-staging`
   - pulls the image to the staging server
   - runs it as a container
   - performs a health smoke test
4. `deploy-production`
   - waits for approval
   - deploys the same image to production after review

This is the core CD pattern: build once, validate, then release the same tested artifact.

---

## Step 15 — GitHub Environments

In GitHub repository settings, create:

- `staging`
- `production`

The workflow uses environment-specific secrets such as:

- `STAGING_HOST`
- `STAGING_USER`
- `STAGING_SSH_KEY`
- `PRODUCTION_HOST`
- `PRODUCTION_USER`
- `PRODUCTION_SSH_KEY`

Why this matters:

- Secrets stay outside the codebase.
- Staging and production can have different deployment rules.
- Production deployment requires manual review.

---

## Step 19–21 — Staging and production deployment

The deployment jobs use SSH to remotely run Docker commands on a server.

Example pattern:

```bash
docker pull ghcr.io/<repo>:<version>
docker stop mlops-api || true
docker rm mlops-api || true
docker run -d --name mlops-api -p 5000:5000 ghcr.io/<repo>:<version>
```

Then the smoke test verifies the deployment health endpoint:

```bash
curl --fail http://<host>:5000/health
```

If that health endpoint succeeds, the pipeline is allowed to move forward. Production is blocked until a reviewer approves the release.

---

## Step 23–25 — Release and rollback

Release flow:

```bash
git checkout main
git pull
git tag v1.0.0
git push origin v1.0.0
```

This triggers the delivery workflow and pauses at the production approval gate.

If a deployment is bad, rollback is straightforward because images are versioned and immutable:

```bash
docker stop mlops-api
docker rm mlops-api
docker run -d --name mlops-api -p 5000:5000 ghcr.io/USERNAME/mlops-cd-demo:1.0.0
```

The key idea is that you restore a known-good image instead of relying on a floating `latest` tag.

---

## Next actions for this repo

To complete the full tutorial on GitHub:

1. Push this repo to GitHub.
2. Create the `staging` and `production` environments.
3. Add required GitHub secrets for SSH deployment.
4. Create a VM/server for staging and production.
5. Install Docker on the server.
6. Tag a release like `v1.0.0` and push it.
7. Watch the workflow move through testing, build, staging, and approval.

---

## Local validation status

The local app and Docker build were validated successfully:

- `python -m pytest -q` -> `2 passed`
- `docker build -t mlops-cd-demo:local .` -> successful

This means the base implementation of the tutorial is working correctly and ready for GitHub-based delivery setup.
