# Evidence checklist

Screenshots and short recordings that show the work is real. Add files here and link them from the README and lessons-learned. **Before adding any image, check it for secrets, tokens, IPs you don't want public, and passwords.**

## Capture list

- [ ] `kubectl get nodes` and `kubectl get pods -A` (cluster running)
- [ ] `kubectl get pods,svc,pvc -n daykin` (all components Ready, storage bound)
- [ ] Application loading in the browser from the test cluster
- [ ] Backend logs showing a successful database connection
- [ ] A rolling update: `kubectl rollout status` (before and after)
- [ ] A rollback: `kubectl rollout undo`
- [ ] A pod killed and automatically recreated
- [ X] Backup file created and checksum
- [ X] Restore test: row counts matching source versus restored
- [ ] Celery worker processing a task; Beat scheduling one
- [ ] (Later) Grafana dashboard and the CI/CD pipeline run

## Naming convention

`YYYY-MM-DD-phaseN-short-description.png`, for example `2026-10-22-phase2-pods-ready.png`.

## Tip

A 30–60 second screen recording (rollout then rollback) is more convincing than ten screenshots. Convert to a GIF or upload it and link it from the README.
