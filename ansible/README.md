# Ansible deploy

Playbooks for deploying the shared **nginx-proxy** + **acme-companion** stack to the DEV server.

Run all commands from this directory (`ansible/`).

## Layout

| File | Purpose |
|------|---------|
| `deploy.yml` | Sync files + start Compose stack |
| `vars_dev.yml` | DEV host, paths, rsync excludes, compose file |
| `hosts.ini` | Inventory (`dev_hosts` → `155.212.224.19`) |
| `ansible.cfg` | Inventory + log path |
| `requirements.yml` | Galaxy collections (`community.docker`, `ansible.posix`) |
| `logs/` | Ansible run log (gitignored except `.gitkeep`) |

## GitHub Actions

Workflow: [`.github/workflows/ci.yml`](../.github/workflows/ci.yml)

| Job | When |
|-----|------|
| `validate` | Every push, PR, and manual run (compose config check) |
| `deploy` | After green validate on **push to main/master** or **workflow_dispatch** (not on PR) |

Manual run: Actions → **CI / CD** → Run workflow → choose Ansible tags (`dev` / `sync` / `compose` / `bootstrap`).

Required GitHub configuration:

1. Repository secret **`DEV_SSH_PRIVATE_KEY`** — full PEM for `appuser@155.212.224.19` (same key as `~/.ssh/id_beget_ebaluksf` locally).
2. Environment **`dev`** — used by the deploy job (optional protection / reviewers).

`DEFAULT_EMAIL` stays on the server in **`proxy.env`** (not synced by rsync; not written by Actions).

## Prerequisites

- Ansible 2.14+ on your machine
- SSH key: set `ANSIBLE_PRIVATE_KEY_FILE` or use default `~/.ssh/id_beget_ebaluksf`
- Docker + Compose plugin on the remote host
- Host ports **80** and **443** free for the proxy

Install collections once:

```bash
ansible-galaxy collection install -r requirements.yml
```

## First-time server setup

1. Ensure the remote path exists (`deploy_dest` in `vars_dev.yml`, default `/home/appuser/proxy`).
2. Create env on the server (rsync **never** copies `proxy.env`):

   ```bash
   # Option A: copy a filled env from your machine
   scp -i ~/.ssh/id_beget_ebaluksf ../proxy.env \
     appuser@155.212.224.19:/home/appuser/proxy/

   # Option B: bootstrap from example, then edit on server
   ansible-playbook deploy.yml -e "filevar=dev" --tags=bootstrap
   ```

3. Edit `proxy.env` on the server and set a real `DEFAULT_EMAIL` (Let's Encrypt contact).

## Deploy

Full deploy (rsync + compose pull/up):

```bash
ansible-playbook deploy.yml -e "filevar=dev" --tags=dev
```

| Tag | What it does |
|-----|----------------|
| `dev` | Sync + compose up |
| `sync` | Rsync only |
| `compose` | Compose pull/up only |
| `bootstrap` | Copy `proxy.env.example` → `proxy.env` if missing |

## Verify

```bash
ssh -i ~/.ssh/id_beget_ebaluksf appuser@155.212.224.19 \
  'docker ps --filter name=nginx-proxy --format "{{.Names}} {{.Status}}"'
docker network ls | grep nginx-proxy-network
```

App stacks join `nginx-proxy-network` as `external: true` and set `VIRTUAL_HOST` / `LETSENCRYPT_*`.
