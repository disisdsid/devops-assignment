# FIXES.md

Document every issue you find and fix in this file.

---

## Fix 1: Service B uses localhost instead of Compose service hostname

**What was wrong:**
- In `docker-compose.yml`, `service-b` had `SERVICE_A_URL=http://localhost:5000`.

**Why it is a problem:**
- In Docker Compose, containers should communicate via service names on the shared network, not `localhost`.
- `localhost` inside `service-b` points to `service-b` itself, so it could not reach `service-a`.

**How I fixed it:**
- Updated `docker-compose.yml` to use `SERVICE_A_URL=http://service-a:5000` for `service-b`.
- Updated `service-b/worker.js` default to `http://service-a:5000` instead of `localhost`.
- Added `restart: unless-stopped` to both services as a basic resiliency improvement.

**What could go wrong if left unfixed:**
- `service-b` would repeatedly fail to poll `service-a`, causing the system to appear broken even though both containers are running.
- The overall Docker Compose stack would not satisfy the assignment requirement that `service-b` successfully polls `service-a`.

---

## Fix 2: Misconfigured env_file and environment mapping in Docker Compose

**What was wrong:**
- `docker-compose.yml` used `env_file: .env.example` but also listed variables under `environment:` without assignment.

**Why it is a problem:**
- In Compose, environment list entries without values are populated from the shell environment, not from `env_file`.
- That meant `SECRET_KEY` and `SERVICE_A_URL` could be empty even though the `.env.example` file contained them.

**How I fixed it:**
- Removed the `environment:` variable lists from both services.
- Kept `env_file: .env.example` so Compose loads values directly from that file.

**What could go wrong if left unfixed:**
- `service-a` would fail at runtime because `SECRET_KEY` would be missing.
- `service-b` could not reach `service-a`, causing the whole stack to fail.

---

## Fix 3: Secrets baked into the Service A image

**What was wrong:**
- `service-a/Dockerfile` set `ENV SECRET_KEY` and `ENV DB_PASSWORD` at build time.

**Why it is a problem:**
- Secrets baked into a container image can be exposed if the image is shared or inspected.
- Runtime configuration should be provided at deployment time, not baked into the build.

**How I fixed it:**
- Removed `ENV SECRET_KEY` and `ENV DB_PASSWORD` from `service-a/Dockerfile`.
- The runtime values are now provided through Compose environment variables.

**What could go wrong if left unfixed:**
- Anyone with image access could extract credentials.
- Secret rotation would require rebuilding images instead of updating deployment configuration.

---

## Fix 4: CI workflow contains plaintext Docker credentials

**What was wrong:**
- `.github/workflows/deploy.yml` stored Docker login credentials directly in source control.

**Why it is a problem:**
- Plaintext credentials in repo files can be leaked and abused.
- This violates secure CICD best practices.

**How I fixed it:**
- Updated the workflow to use GitHub Actions secrets: `DOCKER_USERNAME` and `DOCKER_PASSWORD`.

**What could go wrong if left unfixed:**
- Unauthorized users could gain access to the container registry.
- A leaked secret would allow malicious image pushes or pulls.

---

## Fix 4: Terraform contains hard-coded AWS credentials and overly broad ingress rules

**What was wrong:**
- `terraform/main.tf` included hard-coded AWS access keys.
- The security group allowed all TCP ports from any IP.

**Why it is a problem:**
- Hard-coded credentials in source control are a critical security risk.
- An open security group increases attack surface.

**How I fixed it:**
- Removed explicit AWS credentials from `terraform/main.tf` so the AWS provider uses the standard credential chain.
- Restricted ingress to only SSH (`22`) and application port (`5000`).

**What could go wrong if left unfixed:**
- Credentials could be reused by attackers to compromise AWS resources.
- A wide-open security group could allow unauthorized access to the instance.

---

## Self-initiated Improvements

### Improvement 1: Switched Service A to gunicorn and optimized Docker build
- Updated `service-a/Dockerfile` to run the Flask app with `gunicorn` instead of the Flask dev server.
- Reordered the Dockerfile to copy `requirements.txt` first and install dependencies before the application code.
- This improves container stability and speeds rebuilds when application code changes.

### Improvement 2: Added container build hygiene and repo ignores
- Added `service-a/.dockerignore` and `service-b/.dockerignore` to reduce Docker build context size and avoid leaking local files.
- Added root `.gitignore` to keep `.env`, temporary files, and generated artifacts out of source control.
- Enforced `SECRET_KEY` as a required runtime variable so the app does not silently use a hard-coded secret.
- Updated `docker-compose.yml` to explicitly use `.env` via `env_file`, making the local secret file explicit and avoiding runtime fallback values in source control.
- These changes make the repo safer to work with and reduce accidental commits of local secrets.

### Improvement 3: Added Kubernetes probes and imagePullPolicy
- Updated `k8s/deployment.yaml` to include `livenessProbe`, `readinessProbe`, `imagePullPolicy: IfNotPresent`, and resource limits.
- These changes make the Kubernetes manifest more robust and ready for real deployments.

