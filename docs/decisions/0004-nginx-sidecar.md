# 0004 - Nginx as a sidecar in the backend pod

## Context
Compose used a separate Nginx container that shared volumes with Django. The backend
entrypoint runs `collectstatic` at every start, so static files exist only inside the
running backend pod, and a separate Nginx pod could not see them.

## Decision
Run Nginx as a second container in the backend pod. The two containers share an
`emptyDir` for `/app/staticfiles` and the `daykin-media` volume for `/app/media`.
Django is reached at 127.0.0.1:8000. A Service named `web` exposes Nginx on port 80.

## Consequences
- Same behaviour as production Nginx, with the config kept in a ConfigMap.
- Nginx and Django scale together (fine at one replica).
- Static files are rebuilt on each pod start (as in Compose).
- If the backend ever needs several replicas, move static files to the image build or
  object storage and run Nginx as its own Deployment.