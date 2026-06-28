# First-Run Setup

## Prerequisites

- Ubuntu Server 22.04 or 24.04
- Docker Engine (not Docker Desktop): https://docs.docker.com/engine/install/ubuntu/
- `docker compose` plugin (V2)
- User added to `docker` group: `sudo usermod -aG docker $USER`

## Install Docker on Ubuntu

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

## Start the Stack

```bash
git clone https://github.com/jasincanada/nextcloud-ubuntu-epyc-nc34.git
cd nextcloud-ubuntu-epyc-nc34
cp .env.example .env
nano .env  # fill in your values
docker compose up -d
```

Watch startup: `docker compose logs -f nextcloud`

Nextcloud is ready when `http://<server-ip>:8081/status.php` returns `{"installed":true}`.

## Post-Install OCC Commands

Run these after first install to configure integrations:

```bash
NC=nextcloud-nc34

# Set background job mode to cron
docker exec -u www-data $NC php occ background:cron

# Enable full-text search
docker exec -u www-data $NC php occ app:enable fulltextsearch
docker exec -u www-data $NC php occ fulltextsearch:configure '{"elastic_host":"http://elasticsearch:9200"}'
docker exec -u www-data $NC php occ fulltextsearch:index

# Set Redis for file locking
docker exec -u www-data $NC php occ config:system:set redis host --value redis
docker exec -u www-data $NC php occ config:system:set filelocking.enabled --value true --type boolean

# Configure notify_push
docker exec -u www-data $NC php occ notify_push:setup http://notify_push:7867

# Configure AppAPI DSP
docker exec -u www-data $NC php occ app_api:daemon:register \
  --net nextcloud-network \
  --haproxy_password "${APPAPI_DSP_PASSWORD}" \
  local_docker "Local Docker" docker-install http nextcloud-appapi-dsp 2375
```

## Connecting to Existing Prometheus

After the stack is up, add to your host Prometheus config and reload:

```yaml
scrape_configs:
  - job_name: 'nc34-postgres'
    static_configs:
      - targets: ['localhost:9187']
  - job_name: 'nc34-redis'
    static_configs:
      - targets: ['localhost:9121']
```

```bash
curl -X POST http://localhost:9090/-/reload
```

## Verifying the Stack

```bash
# All containers running
docker compose ps

# Nextcloud status
curl http://localhost:8081/status.php

# PostgreSQL healthy
docker exec nextcloud-ubuntu-epyc-nc34-db-1 pg_isready -U nextcloud

# Redis responding
docker exec $(docker ps -qf name=redis) redis-cli ping

# Exporter metrics reachable
curl -s http://localhost:9187/metrics | head -5
curl -s http://localhost:9121/metrics | head -5
```
