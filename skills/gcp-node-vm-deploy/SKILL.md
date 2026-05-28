---
name: gcp-node-vm-deploy
description: >
  Deploy a Node.js backend to a GCP Compute Engine VM and set up automated GitHub Actions deploys.
  Use this skill whenever the user wants to: provision a GCP VM for a Node.js/Express app, migrate
  from Vercel/serverless to a persistent VM, set up PM2 + Caddy + PostgreSQL on a GCP instance,
  configure SSH-based GitHub Actions CD, or perform any "deploy my backend to GCP" request.
  Also use when the user mentions deploy scripts (setup-vm.sh, deploy.sh, Caddyfile, ecosystem.config.cjs,
  backup.sh) in a GCP context. Even if the user only says "help me set up auto-deploy to GCP", use this skill.
---

# GCP Node.js VM Deploy

This skill walks through provisioning a GCP Compute Engine VM, bootstrapping it with Node.js +
PostgreSQL + Caddy + PM2, deploying a Node.js app, and wiring up GitHub Actions for automated
deploys on every push to `main`.

**Stack (opinionated defaults, all overridable):**
- Machine: `e2-small` (1 vCPU / 2 GB RAM), Ubuntu 24.04 LTS
- Node.js 24.x (via NodeSource), Reverse proxy: Caddy (auto Let's Encrypt HTTPS)
- Process manager: PM2 (systemd-managed, survives reboots)
- Database: PostgreSQL 16 on the same VM (skip if the app uses an external DB)
- Auto-deploy: `appleboy/ssh-action` via GitHub Actions on push to `main`
- 2 GB swap (kernel-tuned), UFW firewall, unattended security upgrades, nightly DB backup

---

## Phase 0 — Gather information

Before touching any infrastructure, collect these values. Ask the user for anything not obvious
from context:

| Variable | Example | Notes |
|---|---|---|
| `GCP_PROJECT` | `hosting-vms` | GCP project ID |
| `GCP_ZONE` | `us-central1-a` | Pick the zone where most other VMs live |
| `VM_NAME` | `myapp-backend-vm` | Kebab-case, descriptive |
| `STATIC_IP_NAME` | `myapp-backend` | Used as the address name in gcloud |
| `APP_NAME` | `myapp-api` | PM2 process name, log prefix |
| `APP_DIR` | `/var/www/myapp` | Where the app lives on the VM |
| `PORT` | `3000` | The port Node.js listens on |
| `MAIN_SCRIPT` | `src/server.js` | Entry point passed to PM2 |
| `DOMAIN` | `api.example.com` | The public hostname (for Caddy TLS) |
| `REPO_URL` | `https://github.com/org/repo.git` | HTTPS clone URL |
| `IS_PRIVATE_REPO` | `false` | If true, need a GitHub PAT for cloning |

Also ask: does the project already have `deploy/setup-vm.sh`, `deploy/deploy.sh`,
`deploy/Caddyfile`, `deploy/backup.sh`, `.github/workflows/deploy.yml`, and `ecosystem.config.cjs`?

---

## Phase 1 — Scaffold deploy files (if missing)

If any of the deploy files are absent, create them from the templates below.
Each template uses `{{PLACEHOLDER}}` syntax — do a find-and-replace with real values before writing.

**ecosystem.config.cjs:**
```js
// PM2 process manager config.
module.exports = {
  apps: [
    {
      name: '{{APP_NAME}}',
      script: '{{MAIN_SCRIPT}}',
      instances: 1,
      exec_mode: 'fork',
      env: { NODE_ENV: 'production' },
      error_file: '{{LOG_DIR}}/error.log',
      out_file:   '{{LOG_DIR}}/out.log',
      merge_logs: true,
      time: true,
      max_memory_restart: '500M',
    },
  ],
};
```

**deploy/setup-vm.sh:**
```bash
#!/usr/bin/env bash
# One-shot VM bootstrap. Idempotent: safe to re-run. Requires Ubuntu 24.04 LTS.
# Usage: sudo bash deploy/setup-vm.sh
set -euo pipefail

DEPLOY_USER="${DEPLOY_USER:-deploy}"
APP_DIR="${APP_DIR:-{{APP_DIR}}}"
LOG_DIR="${LOG_DIR:-{{LOG_DIR}}}"
DB_NAME="${DB_NAME:-{{DB_NAME}}}"
DB_USER="${DB_USER:-{{DB_USER}}}"
NODE_MAJOR="${NODE_MAJOR:-24}"

[[ $EUID -ne 0 ]] && { echo "Run as root (sudo)." >&2; exit 1; }
apt-get update -y
apt-get install -y curl ca-certificates gnupg lsb-release ufw git

# --- Node.js (NodeSource) ---
if ! command -v node >/dev/null || [[ "$(node -v | sed 's/v//' | cut -d. -f1)" != "${NODE_MAJOR}" ]]; then
  install -d -m 0755 /etc/apt/keyrings
  curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
    | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
  echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_${NODE_MAJOR}.x nodistro main" \
    > /etc/apt/sources.list.d/nodesource.list
  apt-get update -y && apt-get install -y nodejs
fi

# --- PostgreSQL 16 ---
if ! command -v psql >/dev/null; then
  install -d /usr/share/postgresql-common/pgdg
  curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc \
    -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc
  echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
    > /etc/apt/sources.list.d/pgdg.list
  apt-get update -y && apt-get install -y postgresql-16
fi
systemctl enable --now postgresql

# --- Caddy ---
if ! command -v caddy >/dev/null; then
  apt-get install -y debian-keyring debian-archive-keyring apt-transport-https
  curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
    | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
  curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
    | tee /etc/apt/sources.list.d/caddy-stable.list
  apt-get update -y && apt-get install -y caddy
fi
npm install -g pm2

# --- deploy user + directories ---
id "${DEPLOY_USER}" >/dev/null 2>&1 || useradd --create-home --shell /bin/bash "${DEPLOY_USER}"
install -d -m 0700 -o "${DEPLOY_USER}" -g "${DEPLOY_USER}" "/home/${DEPLOY_USER}/.ssh"
touch "/home/${DEPLOY_USER}/.ssh/authorized_keys"
chown "${DEPLOY_USER}:${DEPLOY_USER}" "/home/${DEPLOY_USER}/.ssh/authorized_keys"
chmod 0600 "/home/${DEPLOY_USER}/.ssh/authorized_keys"
install -d -o "${DEPLOY_USER}" -g "${DEPLOY_USER}" "${APP_DIR}" "${APP_DIR}/uploads" "${LOG_DIR}"

# --- PostgreSQL role + database ---
sudo -u postgres psql -tc "SELECT 1 FROM pg_roles WHERE rolname = '${DB_USER}'" | grep -q 1 || {
  DB_PASS="$(openssl rand -hex 24)"
  sudo -u postgres psql -c "CREATE ROLE ${DB_USER} LOGIN PASSWORD '${DB_PASS}';"
  echo "    -> Generated DB password for ${DB_USER}: ${DB_PASS}"
}
sudo -u postgres psql -tc "SELECT 1 FROM pg_database WHERE datname = '${DB_NAME}'" | grep -q 1 || \
  sudo -u postgres createdb -O "${DB_USER}" "${DB_NAME}"

# --- 2 GB swap + kernel tuning (vm.swappiness=10, vm.vfs_cache_pressure=50) ---
if [ ! -f /swapfile ]; then
  fallocate -l 2G /swapfile && chmod 600 /swapfile
  mkswap /swapfile && swapon /swapfile
  echo '/swapfile none swap sw 0 0' >> /etc/fstab
fi
for kv in "vm.swappiness=10" "vm.vfs_cache_pressure=50"; do
  key="${kv%%=*}"
  grep -q "${key}" /etc/sysctl.conf \
    && sed -i "s/^${key}=.*/${kv}/" /etc/sysctl.conf \
    || echo "${kv}" >> /etc/sysctl.conf
  sysctl -w "${kv}" >/dev/null
done

# --- Unattended security upgrades ---
apt-get install -y unattended-upgrades apt-listchanges
printf 'APT::Periodic::Update-Package-Lists "1";\nAPT::Periodic::Unattended-Upgrade "1";\n' \
  > /etc/apt/apt.conf.d/20auto-upgrades
systemctl enable --now apt-daily.timer apt-daily-upgrade.timer

# --- UFW firewall (22/80/443) ---
ufw allow OpenSSH && ufw allow 80/tcp && ufw allow 443/tcp && ufw --force enable

# --- Caddy config + PM2 boot ---
install -m 0644 "$(dirname "$0")/Caddyfile" /etc/caddy/Caddyfile
systemctl reload caddy || systemctl restart caddy
env PATH="$PATH:/usr/bin" pm2 startup systemd -u "${DEPLOY_USER}" --hp "/home/${DEPLOY_USER}"

echo "✅  Bootstrap complete. Next: add GitHub Actions public key, clone repo, write .env, run deploy.sh"
```

**deploy/deploy.sh:**
```bash
#!/usr/bin/env bash
# Per-deploy script. Runs on the VM as the deploy user.
# Triggered by .github/workflows/deploy.yml or invoked manually.

set -euo pipefail

APP_DIR="${APP_DIR:-{{APP_DIR}}}"
BRANCH="${BRANCH:-main}"

cd "${APP_DIR}"

echo "==> Fetching latest code"
git fetch --prune origin
git reset --hard "origin/${BRANCH}"

npm ci --omit=dev

# Prisma — remove these three lines if not using Prisma
npx prisma generate
[[ -d prisma/migrations ]] && npx prisma migrate deploy \
  || npx prisma db push --skip-generate --accept-data-loss=false

echo "==> Reloading PM2"
if pm2 describe {{APP_NAME}} >/dev/null 2>&1; then
  pm2 reload ecosystem.config.cjs --update-env
else
  pm2 start ecosystem.config.cjs
  pm2 save
fi

echo "==> Deploy complete"
pm2 status {{APP_NAME}}
```

**deploy/Caddyfile:**
```
{{DOMAIN}} {
    reverse_proxy localhost:{{PORT}}
    encode gzip
    request_body { max_size 20MB }   # remove if no file uploads
    log {
        output file /var/log/caddy/{{APP_NAME}}.log {
            roll_size 50MB
            roll_keep 7
        }
        format json
    }
}
```

**deploy/backup.sh:**
```bash
#!/usr/bin/env bash
# Nightly Postgres backup. Add to root crontab:
#   0 2 * * *  {{APP_DIR}}/deploy/backup.sh >> {{LOG_DIR}}/backup.log 2>&1
# Retention is dual-bounded: age limit (14d) AND count limit (10) — whichever is stricter.
set -euo pipefail

DB_NAME="${DB_NAME:-{{DB_NAME}}}"
BACKUP_DIR="${BACKUP_DIR:-/var/backups/{{APP_NAME}}}"
RETENTION_DAYS="${RETENTION_DAYS:-14}"
RETENTION_COUNT="${RETENTION_COUNT:-10}"
# GCS_BUCKET="${GCS_BUCKET:-gs://{{APP_NAME}}-backups}"

install -d -m 0750 "${BACKUP_DIR}"
OUTFILE="${BACKUP_DIR}/${DB_NAME}-$(date +%Y%m%d-%H%M%S).sql.gz"

sudo -u postgres pg_dump --no-owner --clean --if-exists "${DB_NAME}" | gzip -9 > "${OUTFILE}"
chmod 0640 "${OUTFILE}"
# gsutil cp "${OUTFILE}" "${GCS_BUCKET}/"  # uncomment once GCS bucket is provisioned

find "${BACKUP_DIR}" -name "${DB_NAME}-*.sql.gz" -mtime "+${RETENTION_DAYS}" -delete
find "${BACKUP_DIR}" -name "${DB_NAME}-*.sql.gz" -printf '%T@ %p\0' \
  | sort -z -rn \
  | awk -v keep="${RETENTION_COUNT}" 'BEGIN{RS=ORS="\0"} NR>keep{print $2}' \
  | xargs -0 -r rm -v

echo "[$(date -Iseconds)] Backup complete: ${OUTFILE}"
```

**.github/workflows/deploy.yml:**
```yaml
name: Deploy to GCP VM

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: deploy-production
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: SSH and run deploy script
        uses: appleboy/ssh-action@v1.2.0
        with:
          host:     ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USER }}
          key:      ${{ secrets.VM_SSH_KEY }}
          port:     ${{ secrets.VM_SSH_PORT || 22 }}
          script_stop: true
          script: |
            bash {{APP_DIR}}/deploy/deploy.sh
```

After writing, commit the new files:
```bash
git add deploy/ .github/ ecosystem.config.cjs
git commit -m "chore: add gcp vm deploy infrastructure"
```

---

## Phase 2 — Push all commits to remote

```bash
git push origin main
```

---

## Phase 3 — Provision the VM

### 3a. Reserve a static external IP

```bash
gcloud compute addresses create {{STATIC_IP_NAME}} \
  --project={{GCP_PROJECT}} \
  --region={{GCP_REGION}}

gcloud compute addresses describe {{STATIC_IP_NAME}} \
  --project={{GCP_PROJECT}} \
  --region={{GCP_REGION}} \
  --format="value(address)"
```

### 3b. Create the VM

```bash
gcloud compute instances create {{VM_NAME}} \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}} \
  --machine-type=e2-small \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-standard \
  --tags=http-server,https-server \
  --address={{STATIC_IP_NAME}}
```

Add GCP-level firewall rules if they don't exist (setup-vm.sh also enables UFW inside the VM — both layers are needed):
```bash
gcloud compute firewall-rules create default-allow-http \
  --project={{GCP_PROJECT}} --direction=INGRESS --action=ALLOW --rules=tcp:80 --target-tags=http-server

gcloud compute firewall-rules create default-allow-https \
  --project={{GCP_PROJECT}} --direction=INGRESS --action=ALLOW --rules=tcp:443 --target-tags=https-server
```

---

## Phase 4 — Bootstrap the VM

### 4a. Copy deploy scripts to the VM

```bash
gcloud compute scp --recurse deploy/ {{VM_NAME}}:/tmp/deploy-scripts \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}}
```

### 4b. Run the bootstrap script

```bash
gcloud compute ssh {{VM_NAME}} \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}} \
  --command="sudo bash /tmp/deploy-scripts/setup-vm.sh 2>&1"
```

**Capture the DB password from the output.** It looks like:
```
-> Generated DB password for {{DB_USER}}: <hex-string>
```

The script also creates a 2 GB swapfile, enables UFW (22/80/443), configures unattended security
upgrades, and registers PM2 for systemd autostart.

---

## Phase 5 — Generate GitHub Actions SSH key

```bash
ssh-keygen -t ed25519 -C "github-actions@{{VM_NAME}}" \
  -f /tmp/deploy-key -N ""
```

Add the public key to the VM's deploy user:
```bash
PUB_KEY=$(cat /tmp/deploy-key.pub)
gcloud compute ssh {{VM_NAME}} \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}} \
  --command="echo '${PUB_KEY}' | sudo tee -a /home/deploy/.ssh/authorized_keys \
             && sudo chmod 600 /home/deploy/.ssh/authorized_keys"
```

---

## Phase 6 — Clone the repo and write the .env

```bash
gcloud compute ssh {{VM_NAME}} \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}} \
  --command="sudo -u deploy git clone {{REPO_URL}} {{APP_DIR}}"
```

Write the production .env (chmod 600):
```bash
gcloud compute ssh {{VM_NAME}} \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}} \
  --command="sudo tee {{APP_DIR}}/.env > /dev/null << 'ENVEOF'
PORT={{PORT}}
NODE_ENV=production
DATABASE_URL=\"postgresql://{{DB_USER}}:{{DB_PASS}}@localhost:5432/{{DB_NAME}}\"
JWT_SECRET={{JWT_SECRET}}
CORS_ORIGINS={{CORS_ORIGINS}}
UPLOAD_DIR={{APP_DIR}}/uploads
ENVEOF
sudo chown deploy:deploy {{APP_DIR}}/.env
sudo chmod 600 {{APP_DIR}}/.env"
```

---

## Phase 7 — First deploy

```bash
gcloud compute ssh {{VM_NAME}} \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}} \
  --command="sudo -u deploy bash {{APP_DIR}}/deploy/deploy.sh 2>&1"
```

Smoke test:
```bash
gcloud compute ssh {{VM_NAME}} \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}} \
  --command="curl -s http://localhost:{{PORT}}/healthz"
```

### 7b — Set up nightly backup cron

```bash
gcloud compute ssh {{VM_NAME}} \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}} \
  --command="(crontab -l 2>/dev/null; echo '0 2 * * * {{APP_DIR}}/deploy/backup.sh >> {{LOG_DIR}}/backup.log 2>&1') | crontab -"
```

---

## Phase 8 — Wire up GitHub Actions

```bash
PRIVATE_KEY=$(cat /tmp/deploy-key)

gh secret set VM_HOST    --repo {{GITHUB_REPO}} --body "{{STATIC_IP}}"
gh secret set VM_USER    --repo {{GITHUB_REPO}} --body "deploy"
gh secret set VM_SSH_KEY --repo {{GITHUB_REPO}} --body "${PRIVATE_KEY}"
gh secret set VM_SSH_PORT --repo {{GITHUB_REPO}} --body "22"
```

Clean up the temp key files:
```bash
rm -f /tmp/deploy-key /tmp/deploy-key.pub
```

---

## Phase 9 — Verify automated deployments

```bash
gh workflow run "Deploy to GCP VM" --repo {{GITHUB_REPO}} --ref main
gh run watch $(gh run list --repo {{GITHUB_REPO}} --limit 1 --json databaseId -q '.[0].databaseId') \
  --repo {{GITHUB_REPO}}
```

---

## Phase 10 — Communicate remaining manual steps

Always tell the user:

1. **DNS** — add an `A` record: `{{DOMAIN}}` → `{{STATIC_IP}}`. Caddy cannot issue a TLS cert until this resolves.
2. **HTTPS check** — after DNS propagates: `curl https://{{DOMAIN}}/healthz`
3. **CORS** — update `CORS_ORIGINS` in the `.env`, then `pm2 reload ecosystem.config.cjs --update-env`
4. **Other secrets** — any blank env vars (email, Prisma DIRECT_URL, etc.) must be filled in manually
5. **GCS backups** — when ready to graduate from local-only backups, uncomment the `gsutil` block in `backup.sh` and provision a service account + bucket

---

## Troubleshooting: SSH Permission denied (publickey)

When GitHub Actions fails with `Permission denied (publickey)`:

1. Verify public key is in `/home/deploy/.ssh/authorized_keys` on the VM
2. Check permissions: `.ssh/` must be `700`, `authorized_keys` must be `600`
3. Confirm `VM_SSH_KEY` secret contains the **private** key (not the `.pub` file)
4. Test manually: `ssh -i /tmp/deploy-key deploy@<VM_IP> "echo OK"`
5. Key must be ed25519 or RSA — not DSA or an unrecognized format
