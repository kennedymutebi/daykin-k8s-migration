# 0002: Keep Nginx Proxy Manager in front, initially

**Status:** Proposed

## Context
Nginx Proxy Manager already owns host ports 80, 81 and 443 on the VPS, and the VPS hosts other applications. An earlier K3s install conflicted with these ports.

## Decision
Keep Nginx Proxy Manager as the public entry point. Expose the Kubernetes workload on a tested internal upstream (for example a NodePort or a non-conflicting ingress port) and re-point the proxy host to it only at cutover.

## Why
- Avoids taking over ports 80/443 and breaking other hosted apps.
- Makes cutover and rollback simple: change one upstream in the proxy host, and change it back to roll back.
- TLS and DNS keep working unchanged.

## Alternatives considered
- **Replace it with a Kubernetes ingress controller on 80/443:** cleaner long-term, but high risk on a shared VPS.
- **Use a separate host for Kubernetes:** safest isolation; revisit if resources are tight.

## Consequences
- Two layers of routing during the transition (proxy then ingress), which adds a hop but keeps risk low.
- Revisit once all apps are off the shared VPS.
