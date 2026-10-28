# Immich on Kubernetes

### Cloud-Native Architecture, Persistent Storage & Operational Troubleshooting

A hands-on infrastructure project exploring Kubernetes operations, persistent storage, application deployment, troubleshooting, and observability through a real multi-component application.

The lab runs on a single-node K3s cluster hosted on an Ubuntu virtual machine on Proxmox VE. It uses local SSD-backed persistent volumes for Immich application data and PostgreSQL, and accesses an existing photo library through a read-only NFS mount.

> **Project scope:** This is a learning and portfolio environment, not a production-ready high-availability platform.

## Project Goals

- Deploy and operate a multi-component application on Kubernetes.
- Work with Deployments, StatefulSets, Services, PersistentVolumes (PVs), and PersistentVolumeClaims (PVCs).
- Separate original media, application-managed data, and database files into distinct logical storage boundaries.
- Reduce the risk of accidental changes to original media through a read-only mount.
- Diagnose workload, storage, and node resource-pressure issues using Kubernetes state, events, and logs.
- Document troubleshooting and recovery decisions.
- Practice declarative manifest validation using local YAML checks and Kubernetes server-side dry-run, documenting environment-specific validation failures.
- Demonstrate practical skills relevant to cloud architecture, solutions architecture, and technical presales / sales engineering.

## Architecture Overview

```text
                       Proxmox VE
                           |
                           v
                    Ubuntu VM / K3s
                     (single node)
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
 Immich Server       PostgreSQL              Redis
  (Deployment)       (StatefulSet)        (Deployment)

 Additional workloads:
 - Immich Machine Learning (Deployment)
 - Redis Exporter (Deployment; Prometheus scrape integration
   is pending validation)

 Storage:
 - Immich application data PVC -> local SSD
 - PostgreSQL data PVC -> same local SSD, separate PV
 - Original media -> NFS, mounted read-only in Immich Server

 Web access: NodePort 30082 -> application port 3001
```

The diagram is a high-level view of the architecture, not a complete representation of every application protocol or dependency. Check the manifests and live configuration before relying on it for operational changes.

### Runtime Components

| Component | Kubernetes workload | Declared image | Role |
|---|---|---|---|
| Immich Server | Deployment | `ghcr.io/immich-app/immich-server:v3.2.4` | Web API and asset management |
| Immich Machine Learning | Deployment | `ghcr.io/immich-app/immich-machine-learning:v3.2.4` | Machine-learning tasks such as face detection and search embeddings |
| PostgreSQL | StatefulSet | `ghcr.io/immich-app/postgres:18-vectorchord0.5.3-pgvector0.8.1` | Application metadata and vector-search data |
| Redis | Deployment | `redis:7` | Job queue and caching |
| Redis Exporter | Deployment | `ghcr.io/oliver006/redis_exporter:v1.93.0` | Exposes Redis metrics for monitoring |

The current Kubernetes resource names use a `-v2` suffix to distinguish them from an earlier resource set. This is a naming convention, not a semantic application version. Image tags describe the declared configuration; the running image digests and versions have not been re-verified as part of this README update.

## Storage Architecture

### Logical Separation

The design separates three data categories:

1. **Original media:** An existing photo library accessed over NFS and mounted read-only in the Immich Server container.
2. **Immich application data:** Application-managed files, such as thumbnails and other generated data, stored on a local SSD-backed PVC.
3. **PostgreSQL data:** Database files stored on a separate local SSD-backed PVC.

### What the Design Provides

- **Read-only container mount for originals:** The Immich Server container receives the original library through a read-only mount, reducing the risk of accidental modification through that workload. This does not make the NAS globally immutable or replace independent backups.
- **Separate persistent volumes:** Application data and database files have separate Kubernetes storage resources, making their configuration and capacity easier to manage independently.
- **Node-dependent local storage:** The local persistent volumes are tied to the intended node. This fits the single-node lab but makes recovery dependent on that node and its storage.

### Physical Backing and Limitations

Both the Immich data PV and PostgreSQL PV are backed by the same local SSD. This is **logical separation, not physical storage or I/O isolation**.

The documented design has no storage replication or automatic failover. A node or SSD failure would require recovery using whatever valid backups or other copies are available; a tested backup-and-restore procedure is not claimed here.

### Storage Layout

The following are generalized examples of the lab's storage layout. Adapt paths to the actual environment; private hostnames, personal directory names, and infrastructure addresses are intentionally omitted from this public-facing overview.

| Purpose | Illustrative host path | Container mount | Configuration |
|---|---|---|---|
| SSD backing | `/mnt/immich-ssd` | — | Local SSD partition |
| Immich application data | `/mnt/immich-ssd/immich-data-v2` | `/usr/src/app/upload` in Immich Server | 90 GiB PVC allocation |
| PostgreSQL data | `/mnt/immich-ssd/postgres-v2` | `/var/lib/postgresql` in PostgreSQL | 8 GiB PVC allocation |
| External NAS mount | `/mnt/nfs-library` | — | Illustrative NFS mount; original media is read-only in the application |
| Original photo library | `<NAS-photo-library-path>` | `/mnt/immich-originals` in Immich Server | Original archive |
| Optional pilot library | `/mnt/immich-ssd/pilot-library` | `/mnt/immich-pilot` in Immich Server, if configured | Small test dataset documented as 111 files |

The 90 GiB and 8 GiB figures are configured PVC allocations, not measurements of current consumption. Both volumes share the same underlying SSD; the allocations do not imply separate physical devices. Verify container mount paths against the current manifests before making changes.

## Project Evolution

The lab previously used Immich v1.106.1 with PostgreSQL 14 and pgvecto-rs 0.2.1. The current resource set was created as a clean rebuild.

Project notes describe the rebuild as a way to simplify the transition to a different vector-extension stack and distinguish current Kubernetes resources from the retired set. This README does not claim that a production database migration or a fully tested rollback procedure was performed.

### Declared Configuration

- **Immich images:** `v3.2.4`
- **PostgreSQL image tag:** `18-vectorchord0.5.3-pgvector0.8.1`
- **Redis image tag:** `7`
- **Kubernetes manifests:** Seven manifest files using the `-v2` naming convention

These values describe the configuration recorded in the project materials. Confirm live image digests, running versions, and database extensions against the cluster before treating them as a current runtime inventory.

## Deployment and Configuration

Kubernetes manifests are maintained under `k8s/`:

- `immich-data-v2-storage.yaml` — storage resources for Immich application data.
- `immich-server-v2.yaml` — Immich Server Deployment, NodePort Service, and volume mounts.
- `ml-v2.yaml` — Machine Learning Deployment and Service.
- `postgres-storage-v2.yaml` — PostgreSQL storage resources.
- `postgres-v2.yaml` — PostgreSQL StatefulSet and Service.
- `redis-v2.yaml` — Redis Deployment and Service.
- `redis-exporter-v2.yaml` — Redis Exporter Deployment.

### Secrets Management

The workload configuration is intended to reference database credentials through Kubernetes Secret keys rather than embed secret values in application manifests. Provision secret values separately and never commit credentials to Git. Check the actual Secret name and key references in the manifests before deploying.

Avoid passing real passwords as command-line arguments or placing them in shell history. Use a suitable secret-provisioning method for the environment.

### Manifest Validation

Server-side dry-run can validate resource schemas and admission processing without applying the changes:

```bash
kubectl apply --dry-run=server -f k8s/immich-data-v2-storage.yaml
kubectl apply --dry-run=server -f k8s/immich-server-v2.yaml
kubectl apply --dry-run=server -f k8s/ml-v2.yaml
kubectl apply --dry-run=server -f k8s/postgres-storage-v2.yaml
kubectl apply --dry-run=server -f k8s/postgres-v2.yaml
kubectl apply --dry-run=server -f k8s/redis-v2.yaml
kubectl apply --dry-run=server -f k8s/redis-exporter-v2.yaml
```

These commands illustrate the validation workflow. They are not proof that all seven manifests currently pass; record the date and outcome when running them against the intended cluster.

## Access and Operational Checks

A NodePort service does not by itself establish Internet exposure; actual reachability depends on node networking, routing, and firewall rules. Restrict access to the intended network and users.

The Immich web service is documented as a NodePort:

- **Service name:** `immich-web-service-v2` (confirm against the current manifest)
- **Application port:** `3001`
- **NodePort:** `30082`
- **URL pattern:** `http://<node-ip>:30082`

Replace `<node-ip>` with the address or DNS name appropriate to your own lab. Do not publish private infrastructure addresses unnecessarily.

Useful checks include:

```bash
# Workload status
kubectl get deployments,statefulsets -n immich

# Persistent volume claims
kubectl get pvc -n immich

# Pods and node placement
kubectl get pods -n immich -o wide

# Recent events
kubectl get events -n immich --sort-by=.lastTimestamp

# Endpoint reachability from a machine that can reach the NodePort
curl -I http://<node-ip>:30082
```

A successful HTTP response demonstrates endpoint reachability only. It does not, on its own, establish that every application workflow, database operation, or machine-learning task is healthy.

## Security Assessment with Trivy

Project records indicate that a Trivy scan of `ghcr.io/immich-app/immich-server:v3.2.4` was performed on **2026-10-14**. The recorded findings were:

| Severity | Findings |
|---|---:|
| CRITICAL | 7 |
| HIGH | 144 |
| MEDIUM | 198 |
| LOW | 175 |
| UNKNOWN | 10 |
| **Total** | **534** |

Examples listed among the CRITICAL findings include `handlebars` 4.7.9, `libopenexr` 3.1.13, `libxml2` 2.12.7, `proxy-addr` 2.0.7, and `tar` 7.5.16. Package-level findings and their applicability should be interpreted from the original scan report, including the assessed image digest and vulnerability database metadata where available.

These figures document the scan results recorded for that image at that time; they are not a current scan. **Remediation is outside the scope of this portfolio exercise**, and no claim is made that the findings have been fixed or that the image is free of vulnerabilities.

For a public repository, prefer a concise, reviewed summary of the scan. Keep full JSON and text reports out of the public repository by default unless they have been reviewed for unnecessary detail and you explicitly decide to publish them.

## Troubleshooting and Engineering Learnings

### Node Ephemeral-Storage Pressure

Project notes record an incident in which workloads were evicted while the node was under `ephemeral-storage` pressure.

**Recorded remediation:**

1. Expanded the VM root disk from 25 GiB to 40 GiB through the hypervisor.
2. Grew the partition and filesystem using `growpart` and `resize2fs`.
3. Subsequently observed the node's `DiskPressure` condition clear.

The exact incident date is not established in the material used for this README.

**Engineering lesson:** Node ephemeral storage and persistent-volume capacity are different resource constraints. A PVC can have available space while the node's root filesystem is under pressure. Both need to be considered when diagnosing evictions.

### Storage Lifecycle and Data Validation

Project notes from the earlier deployment describe cases where a database entry did not guarantee that the corresponding file was present on disk, including during interrupted processing or storage pressure.

The resulting design and troubleshooting lessons are:

- Validate application metadata and actual file presence separately when investigating data inconsistencies.
- Treat generated application data as potentially regenerable only where application behavior and source data support that assumption.
- Keep the original library mounted read-only in the Immich container to reduce the risk of accidental changes through that workload.
- Do not treat separate PVCs as a substitute for backups or tested recovery procedures.

This section summarizes documented learning; it does not claim a comprehensive data-integrity test or disaster-recovery exercise has been completed.

### Full-Library Processing Context

Project notes describe an earlier scan involving more than 39,000 assets that coincided with sustained high CPU and memory use on the hypervisor. The available notes do not establish a precise incident date or a fully measured capacity benchmark.

A separate pilot dataset of 111 files is documented for controlled feature testing. The approximately 1.6 TiB figure refers to the NAS's total capacity, not the size of a pending photo migration.

Full-library processing is **not a requirement for the current portfolio objective**. The project is intended to demonstrate Kubernetes deployment, storage boundaries, operational troubleshooting, and security-scanning practice—not to process the entire NAS or claim a comprehensive capacity benchmark.

## Observability

### Redis Exporter

The declared Kubernetes configuration includes a Redis Exporter workload using:

`ghcr.io/oliver006/redis_exporter:v1.93.0`

The exporter is intended to expose Redis metrics through its `/metrics` endpoint, typically on port `9121` in this configuration. Verify the live container arguments, Service, endpoint, and response before describing the exporter as operational.

### Prometheus and Grafana

Prometheus and Grafana exist in the wider lab infrastructure and are managed separately from this repository's application deployment.

**Redis Exporter scrape integration with Prometheus remains pending validation in the available project notes.** This README therefore does not claim that Redis Exporter metrics are currently being scraped or displayed in Grafana dashboards.

## Current Scope and Limitations

The following summarizes the documented design and known validation gaps; it is not a live status report. Runtime readiness was not re-checked as part of this README consolidation.

- **Single-node K3s:** No multi-node scheduling or automatic node failover.
- **Local persistent storage:** Both application and database PVs depend on the same SSD and node.
- **No demonstrated replication:** The documented configuration does not establish replicated storage.
- **Backup and restore:** A tested backup-and-restore procedure is not claimed.
- **Resource requests and limits:** Confirm their presence or absence in the current manifests before describing the live configuration.
- **Monitoring integration:** Redis Exporter integration with Prometheus requires validation.
- **Security scanning:** Historical Trivy findings are documented; remediation is not part of this exercise.
- **Manifest validation:** Run and record fresh server-side dry-runs before claiming current validation.
- **Workload readiness:** Check the live cluster before reporting Deployments or StatefulSets as Ready.

These are transparent boundaries of the lab, not a list of prerequisites to complete the portfolio project.

## Repository Structure

```text
immich-kubernetes-portfolio/
├── k8s/
│   ├── immich-data-v2-storage.yaml
│   ├── immich-server-v2.yaml
│   ├── ml-v2.yaml
│   ├── postgres-storage-v2.yaml
│   ├── postgres-v2.yaml
│   ├── redis-v2.yaml
│   └── redis-exporter-v2.yaml
├── .gitignore
└── README.md
```

The repository contains the README, seven Kubernetes manifest files, and a `.gitignore`. The manifests describe the lab configuration documented in this project.

## Reproducibility and Safe Adaptation

The node hostname and local storage paths in the manifests are environment-specific values that must be reviewed and adapted before deployment. All seven YAML files passed local syntax and basic document-structure checks. However, server-side dry-run validation of the two PersistentVolume manifests against the existing lab cluster was rejected because `nodeAffinity` is immutable on the already-created volumes, whose node affinity differs from the hostname declared in the public manifests. This does not establish that the manifests have passed server-side validation in a clean target cluster.

These manifests document a single-node K3s lab and are provided as an engineering reference, not as a turnkey deployment.

Before applying them to another environment:

- Verify the Kubernetes node name and all host paths.
- Confirm that the required directories and storage mounts exist on the intended node.
- Review PersistentVolume and PersistentVolumeClaim bindings, reclaim policies, and storage capacity.
- Provision the required Kubernetes Secrets separately; never commit secret values.
- Review network exposure, particularly the Immich NodePort service, against the target environment's access requirements.
- Validate the manifests against the target cluster and confirm workload readiness after deployment.

The configuration depends on local storage and a single node. It does not provide high availability, replicated storage, or a demonstrated disaster-recovery procedure. Validate backups and restoration separately before using a similar design for important data.

## About the Author

Telecommunications Engineer with experience in telecommunications, cloud infrastructure, enterprise systems, and technical presales. This project represents continued hands-on work in Kubernetes operations, cloud-native architecture, operational troubleshooting, and solutions engineering.

The focus is on practical engineering: understanding system behavior, forming hypotheses, using evidence to diagnose issues, validating changes, and communicating architectural trade-offs clearly.