# Lessons learned

A running log of real problems, what caused them, and how I fixed them. Add an entry as soon as something breaks, while the details are fresh. Honest failures are the most valuable part of this repo.

## How to write an entry

Copy the template below. Keep it short and specific.

```
### YYYY-MM-DD: <short title>
- **Phase:** (e.g. Phase 2: foundation)
- **What I expected:**
- **What actually happened:** (error message, symptom)
- **Root cause:**
- **Fix:**
- **What I'd do differently / how to prevent it:**
- **Evidence:** (link to screenshot/log in docs/evidence/)
```

## Entries

### 2026-10-20: Port conflicts from an earlier K3s attempt
- **Phase:** 0: Discovery
- **What I expected:** K3s would install alongside the existing apps without trouble.
- **What actually happened:** An earlier K3s installation on the production VPS interfered with ports and other hosted apps, and had to be removed.
- **Root cause:** The VPS already runs Nginx Proxy Manager on host ports 80/81/443, and K3s ships components that also want to bind host ports.
- **Fix:** Removed K3s. Decided to build and test in an isolated environment and to keep Nginx Proxy Manager as the public entry point at first.
- **What I'd do differently:** Never experiment on the production host. Write a port plan before installing anything.

### 2026-10-20: A database credential was exposed during troubleshooting
- **Phase:** 0: Discovery
- **What happened:** A MySQL credential was pasted while troubleshooting.
- **Root cause:** Copying command output without checking it for secrets.
- **Fix / plan:** Treat the credential as compromised and rotate it. Changing `MYSQL_ROOT_PASSWORD` in Compose alone does **not** change an existing MySQL account; the password must be changed inside MySQL (`ALTER USER ...`), then the application config updated.
- **Prevention:** Redact output before sharing; keep secrets in Kubernetes Secrets or a secret manager; never commit `.env` files.

### 2026-10-23: Redis stayed in ContainerCreating
- **Phase:** 2: Kubernetes foundation
- **What happened:** The Redis pod stayed `ContainerCreating` for about 90 seconds, and my first `kubectl logs` failed with "waiting to start".
- **Root cause:** The cluster node had to download the image from Docker Hub for the first time, over a slow connection.
- **Fix:** Waited, then read the `kubectl describe pod` Events to confirm the pull succeeded.
- **Lesson:** `ContainerCreating` is not an error. Check Events with `describe` before assuming something is broken.
- **Evidence:** `docs/evidence/2026-10-23-phase2-redis-running.png`

### 2026-10-23: MySQL stuck in CreateContainerConfigError, and the StatefulSet would not recover
- **Phase:** 3: App and data validation
- **What happened:** `mysql-0` stayed in `CreateContainerConfigError`, even after I fixed the manifest and deleted the pod.
- **Root cause:** My first manifest referenced a ConfigMap/Secret key (`MYSQL_DATABASE`) that did not exist. A StatefulSet will not roll out a fixed definition while its existing pod is unhealthy, so the pod kept being recreated from the old definition.
- **How I found it:** `kubectl describe pod` → Events: "couldn't find key MYSQL_DATABASE in ConfigMap".
- **Fix:** Deleted the StatefulSet (the disk is kept) and applied the corrected manifest fresh.
- **Lesson:** Check that every referenced ConfigMap and Secret key exists before applying. When config objects change, update every manifest that references them. If a StatefulSet is stuck, delete and re-create it (the PVC survives).

### 2026-10-24: Hard-coded database hostname in the app
- **Phase:** 3: App and data validation
- **What happened:** The backend crash-looped with `Unknown server host 'daykin_db'`.
- **Root cause:** `settings.py` hard-coded the Compose service name as the DB host when `RUNNING_IN_DOCKER` is true, ignoring `DB_HOST`. Kubernetes also forbids underscores in Service names, so the old name could not be recreated.
- **How I found it:** `kubectl logs` on the crashing pod.
- **Fix:** Read the host from the environment, keeping today's value as the default: `config('DB_HOST', default=...)`. Built and tested a local image; nothing was pushed to production (the CI/CD pipeline deploys on every push to `master`).
- **Lesson:** Configuration belongs in the environment, not in code. Name rules differ between Docker Compose and Kubernetes.
- **Evidence:** `docs/evidence/2026-10-24-phase3-backend-logs.png`

### 2026-10-24: Local image build took 91 minutes
- **Phase:** 3: App and data validation
- **What happened:** `docker build` took 5,466 s on a slow connection (pip install alone about 39 minutes).
- **Lesson:** Layer caching makes later builds faster. For real releases, let CI build the image.

### 2026-10-24: `kind` was not found in an old terminal
- **Phase:** 3: App and data validation
- **What happened:** `kind` was not recognised in a terminal I had open inside my editor, although it was installed.
- **Root cause:** That terminal had inherited an old PATH from before the install. `kubectl set image` ran anyway and pointed the Deployment at an image the cluster did not yet have.
- **Fix:** Used a fresh PowerShell window, ran `kind load docker-image`, then recreated the pod.
- **Lesson:** Run commands one at a time and check each result. kind nodes have their own image store, so a local image must be loaded into the cluster.

### 2026-10-24: Restore garbled emoji and special characters
- **Phase:** 3: App and data validation
- **What happened:** Row counts matched the live database, but story text in the restored copy showed mojibake (for example `Ã°Å¸Ëœâ€š` instead of an emoji, and `â€œ ... â€` instead of curly quotes).
- **Root cause:** I extracted only the `daykin_db` section of an `--all-databases` dump. That removed the header line that sets the client character set, and I restored without specifying one, so UTF-8 text was read with the wrong encoding.
- **How I found it:** By reading real records through the API in the browser and comparing with the live site, which showed the text correctly.
- **Fix:** Dropped the test database and reloaded the same file with `--default-character-set=utf8mb4`. The live database and the backup file were never affected.
- **Result:** Row counts and text now match the live site.
- **Lesson:** Matching row counts do not prove matching content. Verify actual text, and set the character set explicitly for backups and restores.
- **Evidence:** `docs/evidence/2026-10-24-phase3-api-emoji-fixed.png`

### 2026-10-24: kubectl stalled with "TLS handshake timeout"
- **Phase:** 3: App and data validation
- **What happened:** `kubectl` calls timed out and an `exec` session dropped in the middle of a restore.
- **Root cause:** The local cluster's API server was briefly not answering (a heavily loaded Docker Desktop).
- **Fix:** Waited, checked `kubectl get nodes` and the pod state, then repeated only what had not finished.
- **Lesson:** After a dropped session, check the state first. Do not repeat a change that may already have happened.

### 2026-10-27: Static files exist only inside the backend pod
- **Phase:** 3: App and data validation
- **What happened:** `/static/` returned 404 through the cluster, because nothing served it.
- **Root cause:** The backend runs `collectstatic` at every start, so the files exist only in that pod. Compose solved this with a shared volume; a separate Nginx pod in Kubernetes cannot see another pod's filesystem.
- **Fix:** Ran Nginx as a sidecar in the backend pod, sharing an `emptyDir` for `/app/staticfiles` and the media volume ([decision 0004](docs/decisions/0004-nginx-sidecar.md)).
- **Trade-off:** Django and Nginx now scale together. Fine at one replica.
- **Evidence:** `docs/evidence/2026-10-27-phase3-nginx-static-browser.png`

### 2026-10-27: `kubectl apply` overwrote an image I had set by hand
- **Phase:** 3: App and data validation
- **What happened:** Applying the updated backend manifest briefly put the production image back, replacing my locally built test image.
- **Root cause:** I had changed the image imperatively (`kubectl set image`), but the manifest on disk still had the production tag.
- **Fix:** Ran `kubectl set image` again right after the apply.
- **Lesson:** Imperative changes are lost on the next `apply`. The manifest should carry the image I want, or I must re-set it every time.

### 2026-10-27: kubectl cp and Windows paths
- **Phase:** 3: App and data validation
- **What happened:** The media archive copy left no file in the pod, and I only noticed when the extracted file count was 1 instead of 143.
- **Root cause:** `kubectl cp` reads `C:` in a path as a pod name, so I moved into the backups folder and used a relative file name. My first attempt had also not completed.
- **Fix:** Re-copied, then checked the size (165468186 bytes) and the file count after extraction (143).
- **Lesson:** Verify size after a copy and count after an extract before trusting a restore.
- **Evidence:** `docs/evidence/2026-10-27-phase3-media-restored.png`

### 2026-10-27: Beat catches up after the laptop sleeps
- **Phase:** 3: App and data validation
- **What happened:** After my laptop slept, Beat sent the overdue hourly task as soon as the cluster woke (11:31 and 13:09 instead of on the hour).
- **Lesson:** The schedule survives a pause, so scheduled tasks must be safe to run slightly late. Beat must stay at one replica so a task is never sent twice.
- **Evidence:** `docs/evidence/2026-10-27-phase3-celery-task-ran.png`

### 2026-10-27: A production frontend build points at production
- **Phase:** 3: App and data validation
- **What happened:** The frontend I deployed in the cluster called `https://ebiseera.com/api`, and the browser blocked the calls with a CORS error from `localhost:8081`.
- **Root cause:** The production build has the live API URL baked in.
- **Result:** CORS acted as a safety net, so the test could not touch live data. But it is not a plan.
- **Lesson:** A test build must point at a test backend (see the next entry).
- **Evidence:** `docs/evidence/2026-10-27-phase3-frontend-cors-blocked.png`

### 2026-10-30: Testing the frontend with a build-time variable, not a code edit
- **Phase:** 3: App and data validation
- **What happened:** I needed the frontend to use the cluster backend, but the repo's CI/CD deploys to production on every push to `master`, so editing the API URL in source was risky.
- **Fix:** The code already reads `VITE_API_BASE_URL` at build time. I set it for one PowerShell session (`$env:VITE_API_BASE_URL = "http://localhost:8080/api"`) and ran `npm run dev`. Nothing changed in the source, so there was nothing to revert or accidentally push. The browser's Network tab showed every request going to `localhost`.
- **Lesson:** Prefer a build-time setting over editing code when the pipeline deploys on push. Prove where data comes from (Domain column, backend logs, cutting the tunnel), do not just trust "it works".

### 2026-10-30: Docker Desktop restart made kubectl look broken
- **Phase:** 3: App and data validation
- **What happened:** After a Docker Desktop restart, `kubectl` showed "TLS handshake timeout" and then "the server doesn't have a resource type". `kubectl logs -f` stayed silent and port-forward tunnels were gone.
- **Root cause:** The cluster node was still starting. Tunnels and `logs -f` streams die when the cluster restarts.
- **Fix:** Waited, checked `docker ps` and `kubectl get nodes`, confirmed all pods came back by themselves, then started the tunnels again.
- **Lesson:** Do not delete the cluster in this state: the database and media volumes live inside it. Restart tunnels and log streams after a restart.

<!-- Add new entries below as you go. -->