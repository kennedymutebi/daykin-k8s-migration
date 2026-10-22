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

<!-- Add new entries below as you go. -->
