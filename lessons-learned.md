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

<!-- Add new entries below as you go. -->
