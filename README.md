# Shared Docker reverse proxy + SSL

Host infrastructure for apps on the DEV VM: **[nginx-proxy](https://github.com/nginx-proxy/nginx-proxy)** routes HTTP/HTTPS by `VIRTUAL_HOST`, and **[acme-companion](https://github.com/nginx-proxy/acme-companion)** issues/renews Let’s Encrypt certificates from `LETSENCRYPT_*`.

App repos (gilmoreplace, fauchon, …) do **not** run this stack — they join the external Docker network and advertise env vars.

```
Internet :80 / :443
        │
        ▼
┌──────────────────────────────────────┐
│ nginx-proxy + acme-companion         │  this repo
│ network: nginx-proxy-network         │
└──────────────────────────────────────┘
        │ VIRTUAL_HOST / LETSENCRYPT_*
        ▼
┌──────────────────────────────────────┐
│ App compose stacks (external network)│
└──────────────────────────────────────┘
```

## Layout

| Path | Purpose |
|------|---------|
| `compose.yaml` | `nginx-proxy` + `acme-companion` |
| `proxy.env.example` | Template for Let’s Encrypt default email |
| `ansible/` | Deploy to `appuser@155.212.224.19` |
| `.github/workflows/ci.yml` | Validate compose + Ansible deploy |

## App contract

Containers on `nginx-proxy-network` should set:

| Variable | Purpose |
|----------|---------|
| `VIRTUAL_HOST` | Public hostname |
| `VIRTUAL_PORT` | Backend port inside the container network |
| `LETSENCRYPT_HOST` | Cert hostname(s), usually same as `VIRTUAL_HOST` |
| `LETSENCRYPT_EMAIL` | Contact email (optional if `DEFAULT_EMAIL` is set on the companion) |

In app compose files:

```yaml
networks:
  nginx-proxy-network:
    external: true
```

DNS for each `VIRTUAL_HOST` must point at the proxy host before the first certificate can be issued.

## First boot (server)

1. Copy env and set a real email:

   ```bash
   cp proxy.env.example proxy.env
   # edit DEFAULT_EMAIL=
   ```

2. Start:

   ```bash
   docker compose --env-file proxy.env up -d
   ```

3. Confirm:

   ```bash
   docker ps --filter name=nginx-proxy
   docker network ls | grep nginx-proxy-network
   ```

## Ansible deploy

See [ansible/README.md](ansible/README.md).

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook deploy.yml -e "filevar=dev" --tags=dev
```

`proxy.env` is never rsynced; bootstrap once with `--tags=bootstrap`, then edit `DEFAULT_EMAIL` on the server.

## GitHub Actions

On push to `main`/`master` (or manual dispatch): validate `compose.yaml`, then deploy via Ansible.

Required: repository secret `DEV_SSH_PRIVATE_KEY`, environment `dev`.
