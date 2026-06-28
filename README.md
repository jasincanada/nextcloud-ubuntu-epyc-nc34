# Nextcloud 34 — Ubuntu Server / AMD EPYC

A lean, purpose-built Nextcloud 34 test environment targeting Ubuntu Server on AMD EPYC hardware. Stripped of bundled observability (Prometheus and Grafana already run natively on the EPYC host) and WSL2/Windows-specific workarounds.

## Goals

- Test Nextcloud 34 features, upgrades, and app compatibility
- Run on bare Ubuntu Server (no Docker Desktop, no WSL2)
- Expose metrics to the existing host Prometheus/Grafana stack
- Keep the stack minimal and easy to tear down and rebuild

## What Is Included

| Service | Image | Purpose |
|---|---|---|
| `nextcloud` | nextcloud:34 | Core app |
| `nextcloud-ai-worker-1..4` | nextcloud:34 | TaskProcessing workers |
| `cron` | nextcloud:34 | Background cron |
| `notify_push` | nextcloud:34 | Client push notifications |
| `db` | postgres:16-alpine | Primary database |
| `pgbouncer` | pgbouncer/pgbouncer | Connection pooler |
| `redis` | redis:7.2-alpine | Cache + session lock |
| `collabora` | collabora/code:latest | Online document editing |
| `elasticsearch` | elasticsearch:8.13.4 | Full-text search |
| `postgres-backup` | prodrigestivill/postgres-backup-local:16 | Daily PG backups |
| `postgres-exporter` | prometheuscommunity/postgres-exporter | Metrics → host Prometheus |
| `redis-exporter` | oliver006/redis_exporter | Metrics → host Prometheus |
| `nextcloud-appapi-dsp` | tecnativa/docker-socket-proxy | AppAPI Docker proxy |

## What Is Intentionally Absent

- **No Prometheus / Grafana / Loki** — the EPYC host already runs these; exporters point to host
- **No Watchtower** — manual image updates for a controlled test environment
- **No Authentik** — single-instance test, Nextcloud built-in auth is sufficient
- **No Talk / HPB / Recording** — not in scope for NC34 testing
- **No Whisper / LLM Gateway** — add back as needed for AI feature testing
- **No Portainer / pgAdmin / MinIO / Mailhog / Duplicati** — keeping it lean

## Quick Start

```bash
git clone https://github.com/jasincanada/nextcloud-ubuntu-epyc-nc34.git
cd nextcloud-ubuntu-epyc-nc34
cp .env.example .env
# Edit .env with your values
docker compose up -d
```

Nextcloud will be available at `http://<server-ip>:8081`.

## Prometheus Scrape Config

Add to your existing Prometheus `prometheus.yml` on the EPYC host:

```yaml
scrape_configs:
  - job_name: 'nextcloud-postgres'
    static_configs:
      - targets: ['localhost:9187']
  - job_name: 'nextcloud-redis'
    static_configs:
      - targets: ['localhost:9121']
```

Then reload Prometheus: `curl -X POST http://localhost:9090/-/reload`

## Upgrading Nextcloud

Since this is a controlled environment, images use pinned patch versions. To test an upgrade:

```bash
# Update the version in docker-compose.yaml, then:
docker compose pull nextcloud
docker compose up -d --force-recreate nextcloud cron notify_push \
  nextcloud-ai-worker-1 nextcloud-ai-worker-2 nextcloud-ai-worker-3 nextcloud-ai-worker-4
```

Always take a PostgreSQL dump before upgrading:

```bash
docker exec nextcloud-ubuntu-epyc-nc34-db-1 pg_dump -U nextcloud nextcloud > backup_pre_upgrade.sql
```

## EPYC-Specific Notes

- PostgreSQL is tuned for high core counts and large RAM (adjust `shared_buffers`, `max_connections` in compose as needed)
- `OMP_NUM_THREADS` and `OPENBLAS_NUM_THREADS` are set on AI workers to prevent thread oversubscription
- Named volumes use the Docker default driver — on NVMe-backed Ubuntu, performance is native
- No Windows bind-mount workarounds needed — all volumes are Docker named volumes

## Documentation

- `docs/SETUP.md` — first-run and post-install checklist
- `docs/TESTING.md` — NC34 feature testing notes
- `docs/TROUBLESHOOTING.md` — common issues on Ubuntu/EPYC
