# NC34 Feature Testing Notes

Use this file to track what has been tested, what works, and what needs follow-up.

## NC34 Focus Areas

- [ ] Core upgrade path from NC32 (tested via backup/restore into this testbed)
- [ ] TaskProcessing API changes (AI workers)
- [ ] AppAPI ExApp compatibility with NC34
- [ ] Full-text search (Elasticsearch 8.x connector)
- [ ] Collabora CODE integration
- [ ] notify_push client push
- [ ] PHP 8.3 compatibility (check `php occ check`)

## Test Procedure

### Baseline health check
```bash
docker exec -u www-data nextcloud-nc34 php occ check
docker exec -u www-data nextcloud-nc34 php occ status
docker exec -u www-data nextcloud-nc34 php occ app:list
```

### Simulate upgrade from NC32 data
1. Take a PG dump from the NC32 testbed:
   ```bash
   docker exec nextcloud-nc32-db-1 pg_dump -U nextcloud nextcloud > nc32_export.sql
   ```
2. Import into this testbed's DB:
   ```bash
   docker exec -i nextcloud-epyc-testbed-db-1 psql -U nextcloud nextcloud < nc32_export.sql
   ```
3. Copy userdata volume contents
4. Run upgrade:
   ```bash
   docker exec -u www-data nextcloud-nc34 php occ upgrade
   ```

### Performance baseline (EPYC)
Capture before/after metrics in Grafana:
- DB query latency (postgres-exporter)
- Redis hit rate (redis-exporter)
- PHP-FPM worker utilisation
- AI task processing throughput (number of jobs/minute via admin panel)

## Known Issues / Observations

_Fill in as testing progresses._
