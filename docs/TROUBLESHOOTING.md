# Troubleshooting — Ubuntu / EPYC

## General

```bash
# All container states
docker compose ps

# Follow logs
docker compose logs -f <service-name>

# Resource usage
docker stats --no-stream
```

## Nextcloud

**Maintenance mode stuck**
```bash
docker exec -u www-data nextcloud-nc34 php occ maintenance:mode --off
```

**Trusted domain error**
Add your server IP to `NEXTCLOUD_TRUSTED_DOMAINS` in `.env`, then:
```bash
docker compose up -d --force-recreate nextcloud
```

**OCC: "Nextcloud is not installed"**
Wait for `status.php` to return `{"installed":true}` before running OCC commands. The first startup takes 2-3 minutes.

## PostgreSQL / PGBouncer

**"too many clients"**
Increase `PGBOUNCER_DEFAULT_POOL_SIZE` in `docker-compose.yaml` and recreate pgbouncer.

**PG not ready at startup**
All services use `depends_on: db: condition: service_healthy`. If PG takes longer than expected on first boot (large EPYC system initialising shared memory), increase the healthcheck `start_period`.

## EPYC-Specific

**High context-switch rate on AI workers**
EPYC exposes many logical cores. PHP workers default to using all available threads for some linear algebra libraries. The `OMP_NUM_THREADS=4` and `OPENBLAS_NUM_THREADS=4` env vars in each AI worker container cap this. Tune to your workload.

**Docker network MTU mismatch**
If inter-container traffic is dropping or fragmented, verify the network MTU in `docker-compose.yaml` matches your EPYC host NIC MTU:
```bash
ip link show | grep mtu
```
Update `com.docker.network.driver.mtu` accordingly.

**`/proc/sys/net/ipv4/ip_forward` not enabled**
```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-docker.conf
sudo sysctl -p /etc/sysctl.d/99-docker.conf
```

**Elasticsearch won't start — vm.max_map_count too low**
```bash
sudo sysctl -w vm.max_map_count=262144
# Make permanent:
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-elasticsearch.conf
sudo sysctl -p
```

## Metrics / Prometheus

**Exporters not reachable from host Prometheus**
Confirm the ports are bound to the host:
```bash
ss -tlnp | grep -E '9187|9121'
```
Both should show `0.0.0.0:9187` and `0.0.0.0:9121`. If not, check that the `ports:` entries in `docker-compose.yaml` are not commented out.

**Prometheus reload fails**
Verify the new scrape config has valid YAML before reloading:
```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```
