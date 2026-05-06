---
purpose: K8s deployment recipe for the tether daemon backed by a JuiceFS PV (v0.1 dogfood)
audience: operators deploying tether to a K8s cluster
applies_to: tether daemon (Go, cmd/tether) running as a StatefulSet
spec: tether/docs/specs (Phase-9 / Phase-11 integration points referenced inline)
status: v0.1 shape + operator-fillable. NOT production-hardened. See §"Production hardening follow-ups".
sources: ["v1 backlog: JuiceFS PV deployment manifest + C'' verification"]
---

# tether on K8s — JuiceFS-backed PV (v0.1)

> Single-slice deliverable. The **shape** of the deployment + the **C''
> verification recipe**. Concrete cluster-specific values are
> placeholders — the operator fills them before `kubectl apply`.

## TL;DR

- StatefulSet, replicas=2 for the C'' multi-replica check, `replicas=1` once
  you've confirmed mount sanity.
- Each pod mounts a JuiceFS-backed PVC at `/data` and runs the tether
  daemon with `--projects-dir=/data/claude/projects` and
  `--attach-socket=/run/tether/attach.sock`.
- `~/.tether/` state (incl. `lock.log`) is rooted at `/data/tether/` via the
  `HOME=/data` env var so all of the daemon's "home-rooted" paths land on
  JuiceFS without code changes.
- The **attach socket is local-pod only** — JuiceFS does NOT carry Unix
  domain sockets across replicas. Cross-pod attach is a Phase-11+ feature
  (cross-network WT path); not delivered here.

## Operator must fill (before apply)

| Placeholder | What it is | Suggested source |
|---|---|---|
| `<JUICEFS_VOLUME_NAME>` | JuiceFS filesystem name | `juicefs format ...` you ran on the cluster |
| `<JUICEFS_META_URL>` | Metadata-engine connection URL | Redis HA: `redis://:<pw>@<sentinel-host>:26379/1?master=<master-name>`; TiKV: `tikv://<pd1>:2379,<pd2>:2379,<pd3>:2379/jfs` |
| `<S3_ENDPOINT>` | Object-store endpoint | `https://s3.us-east-1.amazonaws.com` / `https://<account>.r2.cloudflarestorage.com` / `http://minio.minio.svc:9000` |
| `<S3_BUCKET>` | Bucket holding chunks | created out-of-band; tether does not auto-create |
| `<S3_ACCESS_KEY_ID>` / `<S3_SECRET_ACCESS_KEY>` | Object-store creds | put in a Secret, **not** in the ConfigMap |
| `<TETHER_IMAGE>` | Container image ref | `ghcr.io/piaobeizu/tether:<sha>` (built from the `tether` repo's Dockerfile; not in scope here) |
| `<NAMESPACE>` | K8s namespace | e.g. `tether-dev` |
| `<STORAGE_CLASS>` | Name of the JuiceFS StorageClass | matches `metadata.name` in `00-storageclass.yaml` |
| `<PVC_SIZE>` | PVC size | `50Gi` is fine for v0.1 dogfood; JuiceFS treats this as a quota hint |
| `<JUICEFS_CSI_DRIVER>` | CSI driver name | `csi.juicefs.com` (default) |

If any of those are unset when you run `kubectl apply`, the StatefulSet
will land in `Pending` (PVC unbound) or `CrashLoopBackOff` (daemon unable
to dial meta) — both are loud failure modes, not silent corruption.

## Why this shape

**Why a StatefulSet (not a Deployment)** — the daemon writes per-session
state under `~/.claude/projects/<bucket>/<sid>.jsonl` and per-daemon state
at `~/.tether/lock.log`. Stable pod identity (`tether-0`, `tether-1`) is
what lets the C'' verification reason about "which replica wrote which
file". With a Deployment + rolling update, the post-roll pod identity is
disconnected from the pre-roll one, which makes "kill replica-0, verify
replica-1 sees its files" meaningless.

**Why a single PVC mounted on all replicas (not one PVC per replica)** —
the whole point of the C'' check is *shared* state. Each replica has its
own pod-local scratch (the attach socket, /tmp), but JSONL session files
and `lock.log` MUST be visible from all replicas. JuiceFS supports
`ReadWriteMany` natively, which is the access mode we need. (Standard
block PV would force `ReadWriteOnce` and the second replica would never
schedule.)

**Why `HOME=/data` instead of patching tether code** — the daemon's
defaults already resolve socket / projects paths from `$HOME`
(`cmd/tether/main.go:65–68`, `internal/agent/attach_socket.go:55`,
`internal/skill/install.go:18`). Setting `HOME=/data` reroutes
`~/.claude/projects` → `/data/.claude/projects` and `~/.tether/` →
`/data/.tether/` without a code change. The attach socket is then
explicitly redirected to `/run/tether/attach.sock` (an `emptyDir`) via the
`--attach-socket` flag because the socket is the one path that **must
not** live on JuiceFS.

## Metadata-engine choice (v0.1 dogfood)

| Engine | Why for v0.1 | Why not for prod |
|---|---|---|
| **Redis with Sentinel HA** ✅ chosen | Lowest operational surface area (1 chart, well-understood failover). Sufficient for ≤ 100M files, single-tenant. | No horizontal scale; sentinel split-brain still possible; AOF fsync tuning is the operator's problem. |
| TiKV | Horizontally scalable, strong consistency. | Operationally heavy for a v0.1 dogfood — 3 PD + 3 TiKV pods minimum. Defer until we hit Redis ceilings. |
| MySQL / Postgres | Well-known. | JuiceFS metadata is hot-path; relational engines are ~3× slower than Redis on small-file ops, which is the tether daemon's profile (lots of tiny JSONL appends). |

**Decision**: Redis HA + Sentinel for v0.1. We do **not** ship the Redis
manifest in this slice — the assumption is the operator already has a
Redis service available (managed Redis on the cloud, or `bitnami/redis`
chart). The connection URL goes into `<JUICEFS_META_URL>`.

If you have nothing — `bitnami/redis` chart with `architecture=replication`
+ `sentinel.enabled=true` is the v0.1 baseline.

## Chunk-storage choice (v0.1 dogfood)

| Backend | When to pick |
|---|---|
| **MinIO (in-cluster)** ✅ default for first dogfood | Self-contained; no cloud bill; one Helm install. |
| Cloudflare R2 | Egress-free; good if the dogfood already has a CF account. Requires an R2 access-key pair. |
| AWS S3 | Default for prod-like dogfood on AWS. Use a dedicated bucket per environment. |
| GCS via the S3-compat shim | Works but adds a hop; only if the cluster is on GKE and you want native IAM. |

**Decision**: MinIO if no cloud creds are at hand (operator declares with
`<S3_ENDPOINT>=http://minio.minio.svc:9000`), otherwise S3 / R2. The
ConfigMap doesn't care — JuiceFS treats them identically through the
S3-compatible API.

## Files in this slice

```
wiki/deployment/
├── juicefs-pv.md                       (this file)
├── c-prime-prime-verification.md       (the C'' runnable recipe)
└── manifests/
    ├── 00-storageclass.yaml            JuiceFS CSI StorageClass + Secret schema
    ├── 10-configmap.yaml               tether daemon flags + env
    ├── 20-pvc.yaml                     RWX PVC bound to the StorageClass
    └── 30-statefulset.yaml             tether daemon, replicas=2 by default
```

Apply order is the filename prefix.

## Limitations (be loud about these)

1. **Attach socket is pod-local.** `attach.sock` is a Unix domain socket;
   it lives on `emptyDir` (`/run/tether/`). Replica-1 cannot attach to
   replica-0's session by reading JuiceFS. The cross-pod attach path is
   the future cross-network WT bridge (Phase 11+), not JuiceFS. The
   verification recipe explicitly DOES NOT test cross-pod attach.

2. **Two writers to the same `lock.log` is undefined.** v0.1 deliberately
   runs `replicas=2` only for the storage round-trip check. Real workloads
   should pin to `replicas=1` until the daemon learns leader election
   (separate Phase-12 ticket).

3. **JuiceFS CSI driver assumed already installed.** This slice does NOT
   install `juicefs-csi-driver` — it consumes it. See
   https://juicefs.com/docs/csi/getting_started/ for the install side.

4. **No backup.** JuiceFS metadata loss = filesystem loss. Operators must
   schedule `juicefs dump` against the metadata engine before declaring
   this production. v0.1 dogfood does not block on this.

5. **No quota enforcement at the K8s level.** PVC `<PVC_SIZE>` is a hint;
   JuiceFS reports filesystem-level usage but K8s won't evict on overflow.

## Production hardening follow-ups (not in this slice)

- [ ] Leader-elected single-writer for `lock.log` (replicas > 1 safe).
- [ ] PodDisruptionBudget so `kubectl drain` doesn't take both replicas.
- [ ] NetworkPolicy gating the attach socket exposure (when the future
      cross-network WT path lands).
- [ ] Backup job: `CronJob` running `juicefs dump` to a separate bucket.
- [ ] Metadata-engine HA verified with chaos testing (kill the Redis
      master mid-write).
- [ ] TiKV migration path documented if Redis hits the file-count
      ceiling.
- [ ] Mount the projects dir read-only inside any non-daemon sidecars.

## See also

- `c-prime-prime-verification.md` — the runnable recipe
- `manifests/` — the YAML this doc describes
- tether daemon flags: `tether/cmd/tether/main.go:60-97`
- attach-socket default path: `tether/internal/agent/attach_socket.go:55`
- skills pool default: `tether/internal/skill/install.go:18`
- audit-log out-of-scope marker: `tether/internal/lock/doc.go:35`
