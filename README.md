# gcp-vm-skills

AI agent skills for deploying and managing back-end and full-stack applications on Google Cloud Platform Compute Engine VMs.

## Skills

### [`gcp-node-vm-deploy`](./gcp-node-vm-deploy/SKILL.md)

Provisions a GCP Compute Engine VM and wires up a full production stack:

| Component | Choice |
|---|---|
| Machine | `e2-small` (1 vCPU / 2 GB RAM), Ubuntu 24.04 LTS |
| Node.js | 24.x via NodeSource |
| Reverse proxy | Caddy (automatic Let's Encrypt HTTPS) |
| Process manager | PM2 (systemd-managed, survives reboots) |
| Database | PostgreSQL 16 (same VM) |
| CD pipeline | GitHub Actions via `appleboy/ssh-action` on push to `main` |
| Extras | 2 GB swap, UFW firewall, unattended security upgrades, nightly DB backup |

**Covers the full lifecycle:**

- Phase 0 — Gather variables
- Phase 1 — Scaffold deploy files (`setup-vm.sh`, `deploy.sh`, `Caddyfile`, `backup.sh`, `ecosystem.config.cjs`, `deploy.yml`)
- Phase 2 — Push to remote
- Phase 3 — Provision VM + static IP on GCP
- Phase 4 — Bootstrap VM (one-shot, idempotent)
- Phase 5 — Generate GitHub Actions SSH key
- Phase 6 — Clone repo + write `.env`
- Phase 7 — First deploy + nightly backup cron
- Phase 8 — Set GitHub Actions secrets
- Phase 9 — Verify automated deploys
- Phase 10 — Communicate remaining manual steps (DNS, HTTPS, CORS)

Includes a troubleshooting section for `Permission denied (publickey)` SSH errors.

## Installation

### Via skills CLI (recommended)

```shell
npx skills add RenderbitTechnologies/gcp-vm-skills
```

### From a release (manual)

Download the `.skill` file for a specific skill from the [Releases](https://github.com/RenderbitTechnologies/gcp-vm-skills/releases) page, then install it:

```shell
# Example for gcp-node-vm-deploy
unzip gcp-node-vm-deploy.skill -d ~/.agents/skills/
ln -s "../../.agents/skills/gcp-node-vm-deploy" \
      ~/.claude/skills/gcp-node-vm-deploy
```

### Claude Code (manual)

```shell
# 1. Clone this repo
git clone https://github.com/RenderbitTechnologies/gcp-vm-skills.git /tmp/gcp-vm-skills

# 2. Copy the skill(s) you want
mkdir -p ~/.agents/skills/gcp-node-vm-deploy
cp -r /tmp/gcp-vm-skills/gcp-node-vm-deploy/. \
      ~/.agents/skills/gcp-node-vm-deploy/

# 3. Create the symlink Claude Code needs
ln -s "../../.agents/skills/gcp-node-vm-deploy" \
      ~/.claude/skills/gcp-node-vm-deploy
```

Restart Claude Code — the skill will appear automatically.

## Releases

Tagged releases (e.g. `v1.0.0`) automatically package every skill as a `.skill` file and attach it to the [GitHub Release](https://github.com/RenderbitTechnologies/gcp-vm-skills/releases). A `.skill` file is a standard zip archive — you can inspect or extract it with any zip tool.

## Usage

Once installed, trigger the skill by telling Claude Code something like:

- *"Deploy my Node.js backend to GCP"*
- *"Set up a GCP VM with PM2 and Caddy"*
- *"Help me migrate from Vercel to a persistent GCP VM"*
- *"Configure GitHub Actions CD for my GCP instance"*

Claude will walk you through the full provisioning flow, filling in your project's real values for every placeholder.

## Repository layout

```
gcp-vm-skills/
└── gcp-node-vm-deploy/
    ├── SKILL.md              # Skill instructions and deploy script templates
    └── references/
        └── templates.md      # Shell script and config file templates
```

## License

MIT
