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
- **What happened:** The Redis pod stayed `ContainerCreating` for about 90 seconds, and my first `kubectl logs` failed with "waiting to start".
- **Root cause:** The cluster node had to download the image from Docker Hub for the first time, over a slow connection.
- **Fix:** Waited, then read `kubectl describe pod` Events to confirm the pull succeeded.
- **Lesson:** `ContainerCreating` isn't an error. Check Events with `describe` before assuming something is broken.

### 2026-10-20: A database credential was exposed during troubleshooting
- **Phase:** 0: Discovery
- **What happened:** A MySQL credential was pasted while troubleshooting.
- **Root cause:** Copying command output without checking it for secrets.
- **Fix / plan:** Treat the credential as compromised and rotate it. Changing `MYSQL_ROOT_PASSWORD` in Compose alone does **not** change an existing MySQL account; the password must be changed inside MySQL (`ALTER USER ...`), then the application config updated.
- **Prevention:** Redact output before sharing; keep secrets in Kubernetes Secrets or a secret manager; never commit `.env` files.
### 2026-10-23: CreateContainerConfigError on MySQL
- **What happened:** `mysql-0` stuck in `CreateContainerConfigError`.
- **Root cause:** The pod referenced a ConfigMap key (`MYSQL_DATABASE`) that no longer existed after I updated the ConfigMap. The old manifest was still applied.
- **How I found it:** `kubectl describe pod` → Events showed "couldn't find key MYSQL_DATABASE in ConfigMap".
- **Fix:** Replaced the manifest, re-applied, deleted the pod so it restarted.
- **Lesson:** When config objects change, update every manifest that references them, then re-apply.
### 2026-10-23: StatefulSet stuck on a broken pod
- **What happened:** `mysql-0` stayed in `CreateContainerConfigError` even after I fixed the manifest and deleted the pod.
- **Root cause:** My first manifest referenced ConfigMap/Secret keys that didn't exist. A StatefulSet will not roll out a fixed definition while its existing pod is unhealthy, and the pod kept being recreated from the old one.
- **How I found it:** `kubectl describe pod` → Events: "couldn't find key MYSQL_DATABASE in ConfigMap".
- **Fix:** Deleted the StatefulSet (the disk is kept) and applied the corrected manifest fresh.
- **Lesson:** Check that every referenced ConfigMap and Secret key exists before applying. If a StatefulSet is stuck, delete and re-create it.
Problem: the backend crashed in a loop with Unknown server host 'daykin_db'.
Cause: a hostname from Docker Compose was assumed by the app.
How I found it: kubectl logs.

### 2026-10-23: StatefulSet stuck on a broken pod
- **Phase:** 3: Validation
- **What happened:** `mysql-0` stayed in `CreateContainerConfigError` even after I fixed the manifest and deleted the pod.
- **Root cause:** The first manifest referenced ConfigMap/Secret keys that did not exist. A StatefulSet will not roll out a fixed definition while its existing pod is unhealthy.
- **How I found it:** `kubectl describe pod` → Events: "couldn't find key MYSQL_DATABASE in ConfigMap".
- **Fix:** Deleted the StatefulSet (the disk is kept) and applied the corrected manifest.
- **Lesson:** Check every referenced ConfigMap/Secret key exists before applying.

### 2026-10-24: Hard-coded database hostname in the app
- **Phase:** 3: Validation
- **What happened:** The backend crash-looped with `Unknown server host 'daykin_db'`.
- **Root cause:** `settings.py` hard-coded the Compose service name as the DB host when `RUNNING_IN_DOCKER` is true, ignoring `DB_HOST`. Kubernetes also forbids underscores in Service names, so the old name could not be recreated.
- **Fix:** Read the host from the environment, keeping today's value as the default: `config('DB_HOST', default=...)`. Built and tested a local image; nothing was pushed to production.
- **Lesson:** Configuration belongs in the environment, not in code. Name rules differ between Docker Compose and Kubernetes.

### 2026-10-24: Local image build took 91 minutes
- **What happened:** `docker build` took 5,466 s on a slow connection (pip install about 39 min).
- **Lesson:** Layer caching makes later builds faster. For real releases, let CI build the image.

### 2026-10-24: kind was not found in an old PowerShell window
- **Root cause:** The window was opened before kind was installed, so its PATH was stale. `kubectl set image` ran anyway and pointed the Deployment at an image the cluster did not yet have.
- **Fix:** Used a new window, ran `kind load docker-image`, then recreated the pod.
- **Lesson:** Run the commands one at a time and check each result.

### 2026-10-24: Restore garbled emoji and special characters
- **Phase:** 3: App and data validation
- **What happened:** Row counts matched the live database, but story text in the restored copy showed mojibake (for example `Ã°Å¸Ëœâ€š` instead of an emoji, and `â€œ ... â€` instead of curly quotes).
- **Root cause:** I extracted only the `daykin_db` section of an `--all-databases` dump. That removed the header line that sets the client character set, and I restored without specifying one, so UTF-8 text was read with the wrong encoding.
- **How I found it:** By reading real records through the API in the browser, then comparing with the live site, which showed the text correctly.
- **Fix:** Dropped the test database and reloaded the same file with `--default-character-set=utf8mb4`. The live database and the backup file were never affected.
- **Result:** Row counts and text now match the live site.
- **Lesson:** Matching row counts do not prove matching content. Verify actual text, and set the character set explicitly for backups and restores.
- **Evidence:** `docs/evidence/2026-10-24-phase3-api-emoji-fixed.png`

<!-- Add new entries below as you go. -->
