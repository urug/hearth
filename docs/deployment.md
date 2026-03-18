# Deployment

Hearth is configured for Kamal deployment out of the box.

---

## Stack

| Component | Technology |
|---|---|
| Deployment tool | Kamal 2 (already configured) |
| Web server | Puma (via Thruster for asset caching + compression) |
| Database | PostgreSQL |
| Background jobs | Solid Queue (Postgres-backed, no Redis) |
| Action Cable | Solid Cable (Postgres-backed, no Redis) |
| Cache | Solid Cache (Postgres-backed, no Redis) |
| File storage | Active Storage → local disk (dev) / S3 (production) |
| Container | Docker (Dockerfile already present) |

Everything except file storage runs on Postgres. **No Redis required.** This simplifies the
production stack significantly.

- **Solid Cache** — `Rails.cache` (application-level caching, fragment caches)
- **Solid Queue** — background job queue and scheduling
- **Solid Cable** — Action Cable WebSocket state and pub/sub

---

## Kamal Overview

Kamal deploys Docker containers to any server via SSH. It handles:
- Building the Docker image
- Pushing to a registry (Docker Hub, GHCR, etc.)
- Rolling deploys with zero downtime
- Secrets management
- Reverse proxy via Kamal Proxy (replaces nginx)

```bash
kamal setup      # first deploy, provisions server
kamal deploy     # subsequent deploys
kamal rollback   # revert to previous container
kamal console    # Rails console on production
kamal logs       # tail production logs
```

---

## Configuration Files

### `.kamal/secrets`
```bash
KAMAL_REGISTRY_PASSWORD=...
RAILS_MASTER_KEY=...
HEARTH_DATABASE_PASSWORD=...
```

### `config/deploy.yml`
```yaml
service: hearth
image: your-org/hearth

servers:
  web:
    - 192.168.1.1
  job:
    hosts:
      - 192.168.1.1
    cmd: bin/jobs

registry:
  username: your-dockerhub-username
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  secret:
    - RAILS_MASTER_KEY
    - HEARTH_DATABASE_PASSWORD
```

---

## Server Requirements

Minimum for a small group (URUG scale, ~100 users):

| Resource | Minimum | Comfortable |
|---|---|---|
| CPU | 1 vCPU | 2 vCPU |
| RAM | 1 GB | 2 GB |
| Disk | 20 GB | 40 GB |
| OS | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |

Providers: Hetzner (cheapest), DigitalOcean, Linode/Akamai, Fly.io (different deploy model).

**Hetzner CX22** (~$4/month, 2 vCPU, 4GB RAM) is a popular choice for small Rails apps with Kamal.

---

## Database

Postgres runs on the same server for v1 (simplest). For more durability, use a managed Postgres
service (DigitalOcean Managed Postgres, Supabase, Render Postgres).

The multi-database config in `database.yml` separates cache, queue, and cable into their own
databases — this is already set up. Run all migrations on deploy:

```bash
kamal app exec --reuse 'bin/rails db:migrate'
```

Or configure in `deploy.yml`:
```yaml
boot:
  limit: 10
  wait: 2

hooks:
  pre-deploy: bin/rails db:migrate
```

---

## File Storage in Production

Active Storage with S3 (or any S3-compatible service):

```yaml
# config/storage.yml
amazon:
  service: S3
  access_key_id: <%= ENV["AWS_ACCESS_KEY_ID"] %>
  secret_access_key: <%= ENV["AWS_SECRET_ACCESS_KEY"] %>
  region: us-east-1
  bucket: hearth-production
```

S3-compatible alternatives: Cloudflare R2 (no egress fees), Backblaze B2, MinIO (self-hosted).

---

## CI/CD

GitHub Actions is the natural choice (free for public repos, cheap for private):

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
      - run: gem install kamal
      - run: kamal deploy
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.KAMAL_REGISTRY_PASSWORD }}
          RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
```

---

## References

- [Kamal docs](https://kamal-deploy.org)
- [Kamal 2 announcement](https://dev.37signals.com/kamal-2/)
- [Hetzner Cloud](https://www.hetzner.com/cloud)
- [Cloudflare R2](https://www.cloudflare.com/developer-platform/r2/)
- [Active Storage S3 guide](https://guides.rubyonrails.org/active_storage_overview.html#amazon-s3-service)
