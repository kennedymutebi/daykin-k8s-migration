# 0003: Build and test outside production

**Status:** Accepted

## Context
The production VPS has 3 vCPUs and about 7.8 GiB RAM, and it hosts other applications. A previous K3s attempt on this host interfered with ports and apps.

## Decision
Build and validate the Kubernetes version in an isolated environment (a local kind/k3d/minikube cluster, or a separate VM) using **copies** of data. Do not point test Pods at the production database.

## Why
- Production keeps serving users while I learn.
- Kubernetes system components use CPU and memory that the small VPS may not spare.
- Mistakes become lessons instead of outages.

## Consequences
- I must restore a data copy into the test cluster, which also proves backups work.
- The final decision on where Kubernetes runs in production (same VPS vs separate host) is made after measuring resource needs.
