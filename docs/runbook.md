# Runbook

Operational steps for the Daykin migration. **Draft:** fill in and verify each step as you actually perform it, then tick it off. Never paste secret values into this file.

Rule for every destructive or production-touching step: backup verified first, rollback known, and no other hosted app affected.

## 1. Backups (Phase 1)

### 1.1 MySQL logical backup (current Docker deployment)

```bash
# Run on the VPS. Container name from discovery: daykin_db
# Read the password from the container environment so it is not typed into history.
docker exec daykin_db sh -c 'mysqldump --single-transaction --routines --triggers \
  -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases' > daykin-$(date +%F).sql
gzip daykin-$(date +%F).sql
sha256sum daykin-$(date +%F).sql.gz > daykin-$(date +%F).sql.gz.sha256
```

- [ ] Backup file created
- [ ] Checksum recorded
- [ ] Copied off the VPS to a second location

### 1.2 Media backup

```bash
# Volume name from discovery: backend_daykin_media
docker run --rm -v backend_daykin_media:/data:ro -v "$PWD":/backup alpine \
  tar czf /backup/daykin-media-$(date +%F).tar.gz -C /data .
```

- [ ] Archive created and listed (`tar tzf ...`)
- [ ] Copied off the VPS

### 1.3 Config backup

- [ ] Compose files, `nginx.conf`, Nginx Proxy Manager proxy host settings, image tags/digests

### 1.4 Restore rehearsal (required before moving on)

- [ ] Restore the SQL dump into a **throwaway** MySQL (not production) and run a few `SELECT COUNT(*)` checks
- [ ] Extract the media archive into a temp directory and spot-check files
- [ ] Time how long the restore took: `____ minutes`

## 2. Local test cluster (Phase 2)

```bash
# Example using kind (any local cluster is fine)
kind create cluster --name daykin-test
kubectl create namespace daykin
kubectl config set-context --current --namespace=daykin
```

Create secrets out-of-band (never from a committed file):

```bash
kubectl create secret generic daykin-secrets \
  --from-literal=DJANGO_SECRET_KEY='<value>' \
  --from-literal=MYSQL_PASSWORD='<value>' \
  --from-literal=MYSQL_ROOT_PASSWORD='<value>'
```

Apply manifests in order:

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/10-configmap.yaml
kubectl apply -f k8s/20-mysql.yaml
kubectl apply -f k8s/21-redis.yaml
kubectl rollout status statefulset/mysql
kubectl apply -f k8s/30-backend.yaml -f k8s/31-celery.yaml -f k8s/40-frontend.yaml
kubectl get pods
```

- [ ] All pods Ready
- [ ] Service discovery works (`backend` reaches `mysql` and `redis` by name)

## 3. Data restore into the test cluster (Phase 3)

- [ ] Restore the SQL dump into the Kubernetes MySQL
- [ ] Copy media into the media PVC
- [ ] Run Django migrations check and `collectstatic` if needed
- [ ] Test: login, API calls, uploads, static assets, a scheduled job, pod restart recovery

## 4. Rolling update and rollback (evidence for the portfolio)

```bash
kubectl set image deployment/backend backend=kennedymutebi/daykin-backend:<NEW_TAG>
kubectl rollout status deployment/backend
kubectl rollout history deployment/backend
kubectl rollout undo deployment/backend      # rollback
```

- [ ] Screenshot/recording of a successful rollout
- [ ] Screenshot/recording of a rollback

## 5. Cutover checklist (Phase 4)

- [ ] Fresh verified backups taken immediately before
- [ ] Maintenance window agreed (if needed)
- [ ] Nginx Proxy Manager proxy host re-pointed to the tested Kubernetes entry point
- [ ] HTTPS, DNS, API, uploads, DB writes and logs verified
- [ ] Rollback: re-point the proxy host to the old upstream (old containers still running)
- [ ] Old deployment kept running for an agreed rollback window: `____ days`

## 6. Cleanup (Phase 5)

- [ ] Only after stability and verified backups
- [ ] Never delete Docker volumes until contents are confirmed migrated and recoverable
### Restore rules (learned the hard way)
- Always restore with `--default-character-set=utf8mb4`.
- Verify real text after a restore (open a few records through the API), not only row counts.
- When extracting one database from an `--all-databases` dump, remember that the dump header (which sets the character set) is not included. Set the character set explicitly.
- Better for next time: back up only the app database with
  `mysqldump --single-transaction --default-character-set=utf8mb4 --databases daykin_db`.
