# 0001: MySQL as a StatefulSet with a PVC

**Status:** Proposed (validate during Phase 2/3)

## Context
MySQL holds the application's critical data. In Docker Compose it uses a named volume (`backend_daykin_db_data`). Kubernetes Pods are ephemeral, and Pod IPs change.

## Decision
Run MySQL as a **StatefulSet** with a **PersistentVolumeClaim**, exposed through a stable Service name (`mysql`). If a managed database is available and affordable, prefer that instead.

## Why
- A StatefulSet gives a stable network identity and stable storage per Pod, which suits a database.
- A PVC keeps data when the Pod restarts or is rescheduled.
- Docker named volumes do **not** convert automatically into PVCs, so data is migrated by dump and restore, not by re-mounting the old volume.

## Alternatives considered
- **Deployment + PVC:** works for a single replica but lacks the stable identity guarantees of a StatefulSet.
- **Managed database:** removes operational burden and is often the better production choice; depends on cost and hosting.
- **Keep MySQL in Docker, move only the app:** lowest risk, but it does not let me practise stateful workloads.

## Consequences
- Backups and a tested restore are mandatory before any production data moves.
- A single-replica MySQL is still a single point of failure; replication is out of scope for now.
