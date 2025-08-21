# Cloud VM + Docker Compose + Caddy + GitHub Actions
### A reusable deployment blueprint for *any* project on Google Cloud

**Scope:** This is a project‑agnostic, production‑ready playbook you can copy into any repo. It covers:
- CI/CD with **GitHub Actions** (auth via a **CI Service Account** JSON key)
- A single **Compute Engine VM** running **Docker + docker compose v2**
- **Artifact Registry (AR)** for container images
- **Secret Manager (SM)** for environment variables
- **Caddy** as the reverse proxy on ports 80/443
- Health and disk‑space runbooks

No hardcoded project IDs, regions, or service names. Swap the placeholders below and ship.

---

## 🔧 Parameters (fill these once)

Replace **ALL** angle‑bracket placeholders with your values:

| Placeholder | Example | Notes |
|---|---|---|
| `<PROJECT_ID>` | `my-gcp-project` | Your GCP project ID |
| `<REGION>` | `europe-west3` | AR region & nearest VM region |
| `<ZONE>` | `europe-west3-b` | VM zone |
| `<VM_NAME>` | `app-runtime` | The VM instance name |
| `<CI_SA_NAME>` | `github-deploy` | CI service account logical name |
| `<VM_SA_EMAIL>` | `vm-runtime@<PROJECT_ID>.iam.gserviceaccount.com` | Email of SA attached to your VM |
| `<AR_REPO>` | `apps` | Artifact Registry repository name |
| `<IMAGE_NAME>` | `my-app` | Image name inside AR repo |
| `<DOMAIN>` | `example.com` | Optional, if you terminate TLS with Caddy |
| `<SECRET_NAME>` | `runtime-env` | A Secret Manager secret that stores .env |
| `<SERVICE_A>` | `service-a` | First app/container |
| `<SERVICE_B>` | `service-b` | Second app/container (optional) |

> Pro tip: keep parameters in a small `.env.tooling` (not committed) and use simple shell substitution in your local scripts.

---

## 🧠 Architecture (mental model)

### Runtime network (on the VM)
```
Internet
  |
  |  TCP 80/443
  v
+--------------------------------------------------+
| GCE VM: <VM_NAME> (zone: <ZONE>)                 |
| Project: <PROJECT_ID>                            |
| Attached Service Account: <VM_SA_EMAIL>          |
|                                                  |
|  [Docker network: web]                           |
|    +--------------+      +--------------------+  |
|    | Caddy 80/443 | ---> | <SERVICE_A>:<PORT> |  |
|    | (proxy)      | ---> | <SERVICE_B>:<PORT> |  |
|    +--------------+      +--------------------+  |
|                                                  |
|  VM pulls images from Artifact Registry,         |
|  fetches .env from Secret Manager at runtime.    |
+------------------^------------------^------------+
                   |                  |
                   | pull images      | access secrets
        +--------------------+   +--------------------+
        | Artifact Registry  |   | Secret Manager     |
        | <REGION>/<AR_REPO> |   | <SECRET_NAME>      |
        +--------------------+   +--------------------+
```

### CI/CD flow (on every push)
```
Dev push → GitHub Actions
     |
     v
   Build image → Push to AR (:SHA, :latest)
     |
     v
   Update VM metadata (startup-script + IMAGE_TAG)
   Restart VM (or do rolling compose update)
     |
     v
   VM boots → startup-script:
      - read .env (Secret Manager)
      - auth Docker to AR
      - docker compose pull && up -d --wait
```

**Analogy:** The VM is a kitchen; Caddy is the waiter on 80/443; containers are dishes; Secret Manager is the spice cabinet; Artifact Registry is the loading dock. The VM’s Service Account is the chef keyring.

---

## 🔐 Identities & IAM

### CI Service Account (used by GitHub Actions)
- Email: `<CI_SA_NAME>@<PROJECT_ID>.iam.gserviceaccount.com`
- **Grant at project level:**
  - `roles/compute.instanceAdmin.v1` (set metadata, stop/start, describe)
  - `roles/artifactregistry.writer` (push images)
  - *(Optional)* `roles/compute.viewer` (read-only describe)

### VM’s attached Service Account (used by code running on the VM)
- Exactly **one** SA is attached to the VM.
- **Grant at project or resource level:**
  - `roles/secretmanager.secretAccessor` (read your `<SECRET_NAME>`)
  - `roles/artifactregistry.reader` (pull images)
- VM access scopes: prefer **cloud-platform**.

**Check who’s attached:**
```bash
gcloud compute instances describe <VM_NAME> --zone=<ZONE>   --format="value(serviceAccounts.email)"
```

---

## 📦 Environment variables (who needs what)

### GitHub Actions → Repository → Settings → *Secrets and variables* → *Actions*
- **Secrets**
  - `GCP_SA_KEY`: the **entire JSON key** for the CI Service Account
- **Variables**
  - `PROJECT_ID=<PROJECT_ID>`
  - `REGION=<REGION>`
  - `ZONE=<ZONE>`
  - `VM_NAME=<VM_NAME>`

### VM startup metadata (set by CI)
- `IMAGE_TAG=${GITHUB_SHA}`
- `PROJECT_ID`, `REGION`

### Runtime `.env` in Secret Manager (`<SECRET_NAME>`)
*(example sketch; tailor to your app)*
```
ENV=prod
LOG_LEVEL=info
TZ=Asia/Tbilisi

# Example: tokens, DB, cache
APP_TOKEN=...
DATABASE_URL=postgresql://user:pass@host:5432/db
REDIS_URL=redis://redis:6379/0
EXTERNAL_API_KEY=...
```

---

## 🗂️ Files your repo must provide

```
.
├─ docker-compose.yml
├─ Caddyfile
├─ startup-script.sh          # VM boot script (CI injects via metadata)
└─ .github/
   └─ workflows/
      └─ deploy.yml
```

### `docker-compose.yml` (template, two services + Caddy)
```yaml
version: "3.9"

services:
  <SERVICE_A>:
    image: ${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/${IMAGE_NAME}:${IMAGE_TAG:-latest}
    container_name: <SERVICE_A>
    restart: unless-stopped
    command: ["sh","-lc","./start-a.sh"]       # change to your entrypoint
    env_file: [.env]
    expose: ["8000"]                           # internal port
    networks: [web]
    mem_limit: 512m
    cpus: "0.30"
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:8000/health || exit 1"]
      interval: 20s
      timeout: 5s
      retries: 5
      start_period: 30s

  <SERVICE_B>:
    image: ${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/${IMAGE_NAME}:${IMAGE_TAG:-latest}
    container_name: <SERVICE_B>
    restart: unless-stopped
    command: ["sh","-lc","./start-b.sh"]
    env_file: [.env]
    expose: ["8001"]
    networks: [web]
    mem_limit: 512m
    cpus: "0.30"
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:8001/health || exit 1"]
      interval: 20s
      timeout: 5s
      retries: 5
      start_period: 30s

  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    ports: ["80:80","443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
      - caddy_logs:/var/log/caddy
    networks: [web]
    mem_limit: 128m
    cpus: "0.25"
    healthcheck:
      test: ["CMD","caddy","version"]
      interval: 30s
    depends_on:
      <SERVICE_A>:
        condition: service_healthy
      <SERVICE_B>:
        condition: service_healthy

volumes:
  caddy_data:
    external: true     # create once on the VM: docker volume create caddy_data
  caddy_config:
  caddy_logs:

networks:
  web:
    external: true     # create once on the VM: docker network create web
```

### `Caddyfile` (HTTP only; add TLS block below if you own a domain)
```caddyfile
:80 {
  log { output stdout format console }

  @health { path /health }
  respond @health "OK" 200

  handle /a/* {
    uri strip_prefix /a
    reverse_proxy <SERVICE_A>:8000
  }
  handle /b/* {
    uri strip_prefix /b
    reverse_proxy <SERVICE_B>:8001
  }

  handle /* {
    respond "Service Edge: Running" 200
  }

  encode gzip zstd
  header {
    -Server
    X-Content-Type-Options nosniff
    X-Frame-Options DENY
    Referrer-Policy strict-origin-when-cross-origin
  }
}
```

**TLS variant (recommended when you have `<DOMAIN>`):**
```caddyfile
<DOMAIN>, www.<DOMAIN> {
  encode gzip zstd
  handle /health { respond "OK" 200 }

  handle /a/* {
    uri strip_prefix /a
    reverse_proxy <SERVICE_A>:8000
  }
  handle /b/* {
    uri strip_prefix /b
    reverse_proxy <SERVICE_B>:8001
  }

  header { -Server }
}
```

### `startup-script.sh` (VM boot script)
```bash
#!/usr/bin/env bash
set -euo pipefail

APP_DIR="/opt/apps/generic-app"
mkdir -p "$APP_DIR"
cd "$APP_DIR"

# Fetch runtime env (writes .env)
gcloud secrets versions access latest   --secret="<SECRET_NAME>"   --project="$(curl -s -H 'Metadata-Flavor: Google'      http://metadata.google.internal/computeMetadata/v1/project/project-id)" > .env

# Docker auth for Artifact Registry (uses VM SA)
REGION="${REGION:-<REGION>}"
gcloud auth configure-docker "${REGION}-docker.pkg.dev" --quiet

# Deploy
docker compose pull || true
docker compose down --remove-orphans || true
docker compose up -d --wait
```

### `.github/workflows/deploy.yml` (build → push → reboot or roll)
```yaml
name: "🚀 Deploy (Generic VM + Compose + Caddy)"

on:
  push:
    branches: [ master ]
  workflow_dispatch: {}

env:
  PROJECT_ID: <PROJECT_ID>
  REGION: <REGION>
  ZONE: <ZONE>
  VM_NAME: <VM_NAME>
  AR_REPO: <AR_REPO>
  IMAGE_NAME: <IMAGE_NAME>

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 🔐 Auth to GCP with JSON key
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: ☁️ Setup gcloud
        uses: google-github-actions/setup-gcloud@v2

      - name: 🧭 Set project
        run: gcloud config set project "${PROJECT_ID}"

      - name: 🐳 Configure Docker for Artifact Registry
        run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev --quiet

      - name: 🏗️ Build & Push image (:SHA + :latest)
        env:
          IMAGE: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.AR_REPO }}/${{ env.IMAGE_NAME }}
        run: |
          docker build -t "${IMAGE}:${GITHUB_SHA}" -t "${IMAGE}:latest" .
          docker push "${IMAGE}:${GITHUB_SHA}"
          docker push "${IMAGE}:latest"

      - name: 📤 Create startup script (inline)
        run: |
          cat > startup-script.sh <<'EOF'
          #!/usr/bin/env bash
          set -euo pipefail
          APP_DIR="/opt/apps/generic-app"
          mkdir -p "$APP_DIR"
          cd "$APP_DIR"
          gcloud secrets versions access latest --secret="<SECRET_NAME>" --project="${PROJECT_ID}" > .env
          gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet
          docker compose pull || true
          docker compose down --remove-orphans || true
          docker compose up -d --wait
          EOF
          chmod +x startup-script.sh

      - name: 🚀 Apply metadata & reboot (pass IMAGE_TAG=SHA)
        run: |
          gcloud compute instances add-metadata "${VM_NAME}"             --zone="${ZONE}"             --metadata-from-file startup-script=startup-script.sh             --metadata IMAGE_TAG="${GITHUB_SHA}",PROJECT_ID="${PROJECT_ID}",REGION="${REGION}"

          gcloud compute instances stop "${VM_NAME}" --zone="${ZONE}" --quiet
          gcloud compute instances start "${VM_NAME}" --zone="${ZONE}" --quiet

          sleep 120

      - name: 📜 Serial console (startup logs)
        run: gcloud compute instances get-serial-port-output "${VM_NAME}" --zone="${ZONE}" --port=1 | tail -n 400 || true

      - name: 🔍 Health checks
        run: |
          VM_IP=$(gcloud compute instances describe "${VM_NAME}" --zone="${ZONE}"             --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
          echo "VM IP: $VM_IP"
          for i in {1..10}; do
            if curl -fsS --connect-timeout 10 "http://$VM_IP/health" >/dev/null; then
              echo "✅ Health OK (attempt $i)"; break
            fi
            echo "⏳ not ready ($i)"; sleep 10
            [ $i -eq 10 ] && { echo "❌ failed"; exit 1; }
          done
```

> **Zero-downtime variant:** Instead of rebooting the VM, replace the metadata/stop/start steps with an SSH or serial‑in‑order command that runs:
> ```bash
> IMAGE_TAG=$GITHUB_SHA docker compose pull && docker compose up -d --wait
> ```
> You can enable blue/green by duplicating services with different labels and swapping routes in Caddy after health passes.

---

## 🧾 One‑time provisioning (CLI)

### Enable APIs
```bash
gcloud services enable compute.googleapis.com artifactregistry.googleapis.com secretmanager.googleapis.com
```

### Create AR repo
```bash
gcloud artifacts repositories create <AR_REPO>   --repository-format=docker   --location=<REGION>   --description="Docker images for <IMAGE_NAME>"
```

### CI Service Account (+ key)
```bash
gcloud iam service-accounts create <CI_SA_NAME>   --description="CI for GitHub Actions" --display-name="GitHub Deploy CI"

gcloud projects add-iam-policy-binding <PROJECT_ID>   --member="serviceAccount:<CI_SA_NAME>@<PROJECT_ID>.iam.gserviceaccount.com"   --role="roles/compute.instanceAdmin.v1"

gcloud projects add-iam-policy-binding <PROJECT_ID>   --member="serviceAccount:<CI_SA_NAME>@<PROJECT_ID>.iam.gserviceaccount.com"   --role="roles/artifactregistry.writer"

gcloud iam service-accounts keys create ./ci-key.json   --iam-account=<CI_SA_NAME>@<PROJECT_ID>.iam.gserviceaccount.com
# Upload ci-key.json to GitHub as repository secret: GCP_SA_KEY
```

### VM Service Account permissions
```bash
gcloud projects add-iam-policy-binding <PROJECT_ID>   --member="serviceAccount:<VM_SA_EMAIL>"   --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding <PROJECT_ID>   --member="serviceAccount:<VM_SA_EMAIL>"   --role="roles/artifactregistry.reader"
```

### VM OS prep (run once on the VM)
```bash
# Install Docker + compose v2 + gcloud via official docs

sudo mkdir -p /opt/apps/generic-app && sudo chown $USER:$USER /opt/apps/generic-app
cd /opt/apps/generic-app

docker network create web || true
docker volume create caddy_data || true

# Ensure firewall allows TCP 80,443 to the VM (via tag or subnet rule)
```

---

## 🧪 Deployment flow (every push)

1. Commit to your default branch (e.g., `master`).
2. GitHub Actions builds and pushes images: `:SHA` and `:latest`.
3. CI attaches/updates startup‑script metadata and either **reboots** the VM or performs a **rolling compose** update.
4. VM boot/update:
   - fetch `.env` from Secret Manager
   - auth Docker to AR
   - `docker compose pull && up -d --wait`
5. CI prints serial console logs and probes `/health` over HTTP(S).

---

## 🩺 Runbook — Health & Space

**App health (on VM):**
```bash
curl -fsS http://localhost/health && echo OK
docker compose ps
docker compose logs --tail=200
docker stats --no-stream
top -b -n1 | head -n 20
```

**Disk space:**
```bash
df -h
docker system df
sudo du -sh /var/lib/docker/* | sort -h | tail
sudo du -sh /var/log/* | sort -h | tail
```

**Cleanup (be cautious in prod):**
```bash
docker system prune -f
docker image prune -a -f
sudo journalctl --vacuum-time=7d
```
> Keep **10–20% free space** to avoid failed image pulls.

---

## 🧯 Troubleshooting (quick picks)

- **Startup script didn’t run**
  ```bash
  gcloud compute instances get-serial-port-output <VM_NAME> --zone=<ZONE> --port=1 | tail -n 400
  gcloud compute instances describe <VM_NAME> --zone=<ZONE> --format="flattened(metadata)"
  ```

- **Artifact Registry pull fails on VM**
  - Ensure `<VM_SA_EMAIL>` has `artifactregistry.reader`.
  - Reconfigure:
    ```bash
    gcloud auth configure-docker <REGION>-docker.pkg.dev --quiet
    ```

- **Secret fetch fails**
  - Ensure `<VM_SA_EMAIL>` has `secretmanager.secretAccessor` and the secret exists:
    ```bash
    gcloud secrets versions access latest --secret="<SECRET_NAME>" --project="<PROJECT_ID>"
    ```

- **Compose errors about missing network/volume**
  ```bash
  docker network create web
  docker volume create caddy_data
  ```

- **Reverse proxy gotchas (aka “weird URL / TLS / CORS issues”)**
  - Upstream protocol must match (HTTP vs HTTPS to the container).
  - Preserve host if backend expects it: `header_up Host {host}` in Caddy.
  - When behind a CDN/proxy, set Full (Strict) TLS end‑to‑end and verify `X-Forwarded-*` handling.
  - Expose exact webhook paths/methods; verify body size/timeouts.

---

## 🛡️ Security & hygiene

- Keep the **CI SA key** only in GitHub secrets and **rotate** periodically.
- Principle of least privilege: secret accessor only on the needed secret(s).
- Prefer immutable `:SHA` tags; treat `:latest` as convenience only.
- Centralize sensitive env in Secret Manager; avoid committing `.env` to Git.
- Configure log retention (`journalctl --vacuum-time`) or ship to Cloud Logging.
- Snapshot the VM disk weekly and store infrastructure files in Git.

---

## ✅ Final checklist

- [ ] CI SA has `compute.instanceAdmin.v1` + `artifactregistry.writer`
- [ ] VM SA has `secretmanager.secretAccessor` + `artifactregistry.reader`
- [ ] VM has Docker, compose v2, gcloud; firewall 80/443 open
- [ ] `docker network create web` and `docker volume create caddy_data` created
- [ ] Secret `<SECRET_NAME>` contains runtime `.env`
- [ ] AR repo `<AR_REPO>` exists in `<REGION>`
- [ ] Compose + Caddyfile + startup script + GitHub workflow present
- [ ] Health endpoints wired (`/health`, and service routes `/a/*`, `/b/*`)

---

### Appendix — Minimal service swap in Caddy (blue/green idea)
Switch traffic by toggling the upstream in one stanza after the new service is healthy:
```caddyfile
handle /a/* {
  uri strip_prefix /a
  # toggle between v1 and v2
  reverse_proxy <SERVICE_A>-v2:8000
  # reverse_proxy <SERVICE_A>-v1:8000
}
```
Then `docker compose up -d --wait` the new stack, flip the route, and remove the old one.

---

**Use this file as a template** for any GCP project. Drop it in your repos’ `/docs/deploy-blueprint.md` and parameterize with your values. Happy shipping!
