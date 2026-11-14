# Proposed solution: make the survey service safer to run

## At a glance

This proposal addresses availability, recovery, agent disk pressure, safer releases and Git controls for the GCP survey service. It keeps the existing GCP region, Azure DevOps Server, VPN and HTTPS entry point.

**This is a proposal, not a deployment report.** No production resources were changed, and no live deployment or recovery test is claimed. The examples need to be checked against the real cluster, namespace, ports, health endpoints, repository names and Azure DevOps service connections before use.

The target architecture is shown in the editable [draw.io diagram](diagrams/architecture.drawio). It uses `me-west1` (Tel Aviv) and its three zones: `me-west1-a`, `me-west1-b` and `me-west1-c`.

## 1. Keep the service available during releases and failures

### Kubernetes changes

The API is the immediate weakness: it has one replica, no health probes or resource sizing, and the two GKE nodes share one zone. The example [API Deployment](kubernetes/api-deployment.yaml) runs three replicas and adds rollout, health, resource and scheduling controls.

| Change | Why it helps | What to check first |
| --- | --- | --- |
| `replicas: 3`, `maxUnavailable: 0`, `maxSurge: 1` | Keeps existing Pods available until replacements are ready. | Leave capacity for one extra Pod during a rollout. |
| Startup, readiness and liveness probes | Gives slow starts time; routes traffic only to ready Pods; restarts stuck processes. | Implement the example health paths. Liveness should test the process, not PostgreSQL availability, to avoid restart storms during a database outage. |
| CPU and memory requests/limits | Helps the scheduler place Pods and caps resource use. | Treat the sample values as starting points. Check them with load tests and telemetry; CPU limits can throttle, and memory limits can cause OOM kills. |
| Zone and node topology spread | Spreads replicas across zones and prefers different nodes. | `minDomains: 2` needs a compatible Kubernetes version. If unsupported, use best-effort `ScheduleAnyway` or upgrade first. |
| 45-second termination grace period | Gives the process time to handle `SIGTERM`, drain requests and close database connections. | The application itself must implement graceful shutdown. |
| PodDisruptionBudget with `minAvailable: 2` | Keeps two API Pods available during voluntary disruptions such as node drains. | It does not prevent involuntary zone or machine loss, and does not replace the rollout strategy. |

Also run multiple Web replicas if Web is on the request path. Replicate the Worker only after confirming that jobs are safe to run concurrently (for example, queue-based and idempotent, or protected by leases/leader election). Otherwise, jobs may run twice. Keep the existing HTTPS load balancer, and check that its health check agrees with Pod readiness.

### Spread infrastructure across zones

Move from two nodes in one zone to a **Regional GKE Standard cluster** with node pools in all three `me-west1` zones. Ensure the remaining two zones have enough capacity to run the critical replicas after one zone is lost. A regional control plane protects the Kubernetes API; node placement, Pod spread and spare capacity protect the application. A regional cluster alone does not guarantee application availability.

First check whether the current control plane is zonal or regional:

- If it is regional and only the node pool is limited to one zone, expand or recreate the pool for all three zones. Confirm node labels and actual Pod placement.
- If it is zonal, create a new Regional cluster and migrate. GKE does not convert an existing zonal cluster in place. Provision the new cluster, recreate networking, identity, secrets and ingress, deploy and validate, shift traffic, then retire the old cluster.

Change Cloud SQL from single-zone to **regional High Availability**. Its standby can take over in another zone while preserving the instance endpoint; clients still need reconnect and retry/backoff logic. This is zonal protection, not regional disaster recovery. Use a regional Cloud Storage bucket for zonal durability; use a dual-region bucket only if the business requirement includes a regional outage.

**Verify before claiming availability:** send continuous synthetic requests through an API rollout; drain a node and observe the PDB; confirm replicas occupy separate zones; simulate node loss in a non-production environment; and test Cloud SQL failover and client reconnection. Run an approved zone-failure game day or controlled capacity test. Record errors, latency, recovery time and dropped or duplicated jobs.

## 2. Recover PostgreSQL and uploaded files together

### Prepare before an incident

- Keep Cloud SQL automated backups for at least seven days. Enable PostgreSQL Point-in-Time Recovery (PITR), with transaction-log retention covering the recovery window (at least seven days if that is the operational requirement). Confirm logs are retained in Cloud Storage. Enabling PITR on an existing instance causes a restart, so schedule and validate it. Keep deletion protection on and limit who can change backup settings.
- Enable Cloud Storage **Soft Delete** for seven days. Enable **Object Versioning** if the application can overwrite object names, and use lifecycle rules that keep noncurrent versions through the recovery and compliance windows. Versioning increases storage cost; it does not replace Soft Delete or protect against bucket deletion.
- Separate application write access from recovery/admin access with least-privilege IAM. Use retention policies or object holds only when required; they can block normal operations.
- Store each uploaded file's bucket, object key, generation, size and checksum in its database row. Prefer unique, immutable object keys. Upload the object before committing its database reference; if that commit fails, clean up the orphan only after a safety delay. For deletes, record a database tombstone and delete the object asynchronously.

The database and bucket do not share a transaction. These records link a restored database row to the exact file generation and help avoid mismatched recovery.

### Runbook: deletion at 10:20, discovered at 10:35

The two-hour recovery clock starts at **10:35**, when the incident is declared, so the target is service restored by **12:35**. Find the first bad delete from audit and application logs; the database recovery point should be immediately before it (for example, 10:19:59), not the discovery time. Measure the 15-minute RPO against the latest committed, recoverable state before the incident.

1. **Contain the problem.** Stop or restrict destructive writes and pause jobs that change survey/file records. Preserve audit logs and identify affected objects. Do not restore over production.
2. **Choose a recovery time.** Correlate database, application and Cloud Storage access logs to locate the first bad delete or overwrite. Record legitimate writes that may be lost after the chosen point.
3. **Restore PostgreSQL to a new instance.** Use Cloud SQL PITR at the selected time. PITR creates a separate instance; it does not rewind production in place. Keep the original isolated for investigation and for any unaffected later records.
4. **Restore exact file versions.** Use the restored rows' object keys, generations and checksums. Recover deleted files from Soft Delete or copy the needed noncurrent generation. Use a recovery bucket or controlled prefix first if a restore might overwrite valid objects. Preserve metadata and verify checksums.
5. **Reconcile the two stores.** Find referenced files that are missing, unreferenced files and checksum mismatches. Restore missing generations; quarantine extras instead of deleting them immediately. Check row counts, referential integrity, file existence, checksums and survey/report behavior.
6. **Cut over carefully.** With approval, point the application to the recovered Cloud SQL instance through its connection configuration or Secret Manager (prefer a Cloud SQL connector/connection name to a hard-coded IP). Restart or deploy safely, run smoke tests, then gradually resume workers and writes. Keep the old database and recovery artifacts until the incident is closed and retention permits cleanup.
7. **Measure and close.** Record detection, restore start, database ready, file recovery complete, validation complete and service restored times. Confirm data loss is no more than 15 minutes and service recovery no more than two hours. Review the incident and update the runbook.

Configuration alone does not guarantee the RTO. Rehearse PITR and file restore with representative data volumes, and measure the full duration. If it takes too long, reduce the recovery dataset/time or consider a warmer recovery design, such as a continuously maintained database replica plus a separately tested object recovery process. A Cloud SQL HA standby is for zonal failure, not logical deletion: it will also receive the bad delete.

## 3. Recover space safely on the Linux build agent

The diagnostics show 76 GB in `/var/lib/docker`, 12 GB in `/opt/ado-agent/_work` and 4 GB in other `/var/lib` content. That explains most, but not necessarily all, of the full 100 GB root disk. Before cleanup, stop new jobs, let the current job finish and confirm the agent is idle.

**Investigate first:**

```bash
df -hT /
df -i /
findmnt /
sudo du -xhd1 /var/lib /opt/ado-agent/_work | sort -h
sudo lsof +L1
docker system df -v
docker ps -a
docker image ls
docker volume ls
docker builder du
journalctl --disk-usage
```

These checks distinguish bytes from inode exhaustion, find large directories without crossing filesystems, reveal deleted-but-open files, and show Docker images, containers, volumes and build cache. Inspect Docker/container logs too. Review `_work` by workspace (`s`, `b`, `a`, `TestResults`) and age; confirm no active build or needed artifact/diagnostic data is there.

**Safe immediate cleanup:** remove the smallest confirmed disposable data first. Delete stale, completed Azure DevOps workspaces only when no job is using them. Keep agent configuration/credentials, `_tasks`, `_tool`, `_diag` and active workspaces. Remove exited containers and clearly unused images or build cache only after checking dependencies. Prefer `docker builder prune` and `docker image prune` to manual deletion. Do not prune volumes or delete `/var/lib/docker`; volumes may contain state and the agent may need its cache. Recheck `df -h` after each batch and preserve diagnostic logs. If a deleted file is still open, release its space by restarting the owning service during an approved window.

**Prevent a repeat:** configure appropriate workspace cleanup, an age/size-based cache policy, Docker log rotation (`max-size`/`max-file`), disk and inode alerts, and a disk-growth or separate-volume plan for Docker and agent workspaces. Publish artifacts to the artifact store instead of leaving them on the VM. Verify cleanup cannot race active jobs and that clean builds still pass.

## 4. Make releases traceable and rollbacks safe

The `latest` tag can move, and the previous deployed digest was not recorded. Give each image a unique tag containing the source commit and build ID, resolve its SHA-256 digest, and deploy using `image@sha256:...`. Enable immutable tags on the Artifact Registry Docker repository:

```bash
gcloud artifacts repositories update REPOSITORY \
  --location=me-west1 --immutable-tags
```

Keep release metadata with the commit SHA, build ID, image digest and migration version. The [Azure DevOps Server pipeline example](pipelines/azure-pipelines.yml) shows build/test/push, metadata publication and production deployment. It uses `PublishBuildArtifacts@1` and `DownloadBuildArtifacts@0`: Azure DevOps Server does not support Pipeline Artifact tasks.

Configure the `production` Environment and its approval/check in Azure DevOps Server before running the example. Do not rely only on a YAML condition: a pipeline author can change YAML. Restrict who can manage approvals or use the GCP deployment identity. Prefer short-lived agent credentials where supported; otherwise use a dedicated, narrowly scoped identity and protect/rotate its credential. Build needs Artifact Registry Writer; deploy needs only required cluster access and namespace-scoped Kubernetes permissions.

If a migration was additive and backward-compatible, stop promotion after failed health checks and redeploy the previously recorded image digest. Leave the expanded schema in place; do not automatically run a destructive down-migration. Use **expand and contract**:

1. Add compatible schema elements while old code still works.
2. Deploy code that can handle old and new forms; dual-write if needed.
3. Backfill and verify data.
4. Switch reads to the new form.
5. Remove old schema only in a later release, after no running version depends on it.

For an incompatible migration, stop and prefer a forward fix or a planned data migration. Restoring the whole database can discard valid transactions written since the restore point; it is a last-resort data recovery, not a normal application rollback. Record metadata for the previous deployment as well as each new release.

**Verify in staging:** deploy a known-bad version and confirm health checks stop promotion; redeploy the previous digest and confirm running Pods use exactly that digest. Rehearse recovery after an additive migration and a simulated incompatible schema change.

## 5. Protect Git and production releases

- Use short-lived feature branches and Pull Requests into protected `main`; ordinary contributors should not push directly.
- Require at least one independent reviewer (two for high-risk changes if the team permits), successful build/test validation and resolved review comments. Require the PR to be current with its target branch or use the configured merge strategy.
- Give production deployment permission to the release pipeline, not broad developer credentials. Protect the Production Environment and service connection separately; branch policy alone does not authorize deployment.
- Use Azure Repos Branch Policies for minimum reviewers, build validation, comment resolution and restrictions on who can complete PRs.
- Start hotfixes from the production commit. Expedite review, but keep independent review, required validation and the same immutable-image/production-approval process. Merge or cherry-pick the fix back to `main` immediately. Reserve policy bypass for a documented, audited break-glass action followed by review.

## Target architecture

The [editable draw.io diagram](diagrams/architecture.drawio) shows traffic, data and deployment paths. The global external HTTPS load balancer is outside the VPC; Cloud SQL uses private service access to the VPC; Cloud Storage, Artifact Registry and Secret Manager are managed services accessed over Google APIs. The VPN and existing HTTPS entry point are retained. The worker's placement is illustrative; only replicate it if job handling is safe.

## Verification checklist

- [ ] API rollout and node drain do not send traffic to unready Pods.
- [ ] API replicas span zones, and remaining zones have capacity after one zone fails.
- [ ] Cloud SQL regional HA failover and application reconnection are tested.
- [ ] PITR to a selected timestamp is rehearsed and validated before cutover.
- [ ] Deleted/overwritten objects are restored by exact generation and checksum.
- [ ] A full recovery rehearsal with representative data meets the 15-minute RPO and two-hour RTO.
- [ ] Agent cleanup preserves active builds and Docker volumes; byte and inode alerts are active.
- [ ] Pipeline records immutable image digest and prior release metadata; rollback by digest is rehearsed.
- [ ] `main` branch policy and Production Environment approval are configured outside editable pipeline YAML.

## References

1. [GKE regional clusters](https://cloud.google.com/kubernetes-engine/docs/concepts/regional-clusters)
2. [Kubernetes probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) and [PodDisruptionBudgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
3. [Cloud SQL high availability](https://cloud.google.com/sql/docs/postgres/high-availability), [configure PITR](https://cloud.google.com/sql/docs/postgres/backup-recovery/configure-pitr) and [PITR restore](https://cloud.google.com/sql/docs/postgres/backup-recovery/pitr)
4. Cloud Storage [Object Versioning](https://cloud.google.com/storage/docs/object-versioning), [Soft Delete](https://cloud.google.com/storage/docs/soft-delete) and [restoring versions](https://cloud.google.com/storage/docs/using-versioned-objects)
5. [Docker pruning](https://docs.docker.com/engine/manage-resources/pruning/)
6. Artifact Registry [immutable tags](https://cloud.google.com/artifact-registry/docs/repositories/update-repo-settings) and [GKE image digests](https://cloud.google.com/kubernetes-engine/docs/concepts/about-container-images)
7. Azure DevOps Server [2022 release notes](https://learn.microsoft.com/en-us/azure/devops/server/release-notes/azuredevops2022), [approval checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals), [agent directories](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents) and [Build Artifacts task](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-build-artifacts-v1)
8. Google Cloud's [official legacy icon library](https://cloud.google.com/icons) supplies the service icons embedded in the diagram.
