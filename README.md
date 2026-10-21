Ebiseera: Docker Compose → Kubernetes migration

A hands-on, documented migration of a real production application (Django API + Celery + MySQL + Redis + React frontend) from Docker Compose on a single VPS to Kubernetes, done with a backup-first, test-in-isolation, planned-cutover approach.

> **Status:** 🚧 In progress. Started 20 October 2026. This README is updated at each milestone.

## Why I'm doing this

I had learned Kubernetes theoretically and wanted to prove it on something real, not a to-do-app tutorial. Daykin or now called ebiseera is a live application with state (MySQL, uploaded media), background jobs (Celery worker and Beat), and a public reverse proxy, which makes it a good test of whether I can migrate carefully without breaking production.

Goals:

- Learn Kubernetes by operating it, not just reading about it.
- Practise safe change management: inventory → backup → isolated test → validation → planned cutover → cleanup only after stability.
- Document the real problems and decisions so others can learn from them.

## Before: Docker Compose on one VPS

```mermaid
flowchart LR
    U[Users] --> NPM["Nginx Proxy Manager<br/>host :80 / :81 / :443"]
    NPM --> FE["Frontend container<br/>host :3006"]
    NPM --> NG["Daykin Nginx<br/>host :8085"]
    NG -->|/api/| BE["Django backend :8000"]
    NG -->|/media/, /static/| VOL[("Docker volumes<br/>media + static")]
    BE --> DB[("MySQL 8.0<br/>Docker volume")]
    BE --> RD[("Redis 7")]
    CW["Celery worker"] --> RD
    CB["Celery Beat"] --> RD
    CW --> DB

```


## After: Kubernetes (target design)

```mermaid
flowchart LR
    U[Users] --> NPM["Nginx Proxy Manager<br/>(kept initially)"]
    NPM --> ING["Kubernetes entry point<br/>(Ingress / NodePort, tested)"]
    subgraph NS["Namespace: daykin"]
        ING --> FES["Service: frontend"]
        ING --> BES["Service: backend :8000"]
        FES --> FED["Deployment: frontend"]
        BES --> BED["Deployment: backend (Django)"]
        CWD["Deployment: celery-worker"]
        CBD["Deployment: celery-beat (1 replica)"]
        BED --> MY[("StatefulSet: MySQL<br/>+ PVC")]
        BED --> RDS[("Redis")]
        BED --> MED[("PVC: media")]
        CWD --> RDS
        CBD --> RDS
        CM["ConfigMap + Secrets"] -.-> BED
        CM -.-> CWD
    end
```

This is the **target design**. It is validated step by step, and the diagram is updated to match reality as the work progresses.

## Migration approach

| Phase | Goal | Status |
|---|---|---|
| 0. Discovery | Inventory the current deployment, record unknowns | ✅ Done ([plan](docs/Daykin_Docker_to_Kubernetes_Migration_Plan.pdf)) |
| 1. Backups and recovery | Verified MySQL and media backups, tested restore | ⬜ |
| 2. Kubernetes foundation (isolated) | Local/test cluster, namespace, storage, config | ⬜ |
| 3. App and data validation | Deploy all components, restore data copy, test | ⬜ |
| 4. Traffic cutover | Route via the proxy to Kubernetes, with rollback plan | ⬜ |
| 5. Cleanup | Remove old resources only after stability | ⬜ |

## Repository layout

```
.
├── README.md                 ← you are here
├── lessons-learned.md        ← real problems hit and how they were fixed
├── .env.example              ← variable NAMES only, never real values
├── k8s/                      ← starter manifests (placeholders, no secrets)
└── docs/
    ├── Daykin_Docker_to_Kubernetes_Migration_Plan.pdf
    ├── runbook.md            ← backup, restore, deploy, rollback steps
    ├── decisions/            ← short decision records (why X, not Y)
    ├── evidence/             ← screenshots / recordings checklist
    └── posts/                ← plan for the milestone write-ups
```

## Key decisions (summary)

- [Why MySQL is a StatefulSet with a PVC](docs/decisions/0001-mysql-statefulset.md)
- [Why Nginx Proxy Manager stays in front at first](docs/decisions/0002-keep-nginx-proxy-manager.md)
- [Why I test outside production](docs/decisions/0003-test-outside-production.md)

## Security

No secrets are committed to this repository. Manifests use placeholders, real values live in Kubernetes Secrets created out-of-band, and `.env` files are git-ignored. See `.env.example` for variable names only.



## Evidence

### Phase 0: Starting point
![Ebiseera running on Docker Compose before the migration](docs/evidence/2026-10-20-phase0-site-before.png)

### Phase 1: Backups
Backup created on the server with no downtime (MySQL dump and media archive):

![Backup files on the server](docs/evidence/2026-10-21-phase1-backup-on-server.png)

Copy kept on a second machine:

![Backup copied to my computer](docs/evidence/2026-10-21-phase1-backup-copied-local.png)
Restore test: the backup was loaded into a throwaway MySQL on my computer and row counts were compared with the live database.

Test copy (restored from backup):

![Row counts in the restored test database](docs/evidence/2026-10-21-phase1-restore-counts-test.png)

Live database:

![Row counts in the live database](docs/evidence/2026-10-21-phase1-restore-counts-live.png)

Result: all 22 tables restored and row counts matched the live database.

## Write-ups

Milestone posts will be linked here as they are published.

## About

Built by Kennedy Mutebi, computer scientist. Questions or feedback are welcome via issues.
