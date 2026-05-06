---
purpose: runnable verification recipe for the JuiceFS-backed tether deployment ("C''" check)
audience: operator running this deployment for the first time
applies_to: a freshly applied stack from wiki/deployment/manifests/
status: v0.1 — every step is a concrete kubectl invocation, no hand-waving
---

# C'' verification — JuiceFS-backed tether daemon

> "C''" = the consistency-and-availability slice that the v1 backlog flagged.
> Concretely: two daemon replicas writing to the same JuiceFS-backed home
> directory, observing each other's session JSONL + audit log without
> coordination beyond the filesystem itself. **NOT cross-pod attach** —
> see Limitation #1 in `juicefs-pv.md`.

## Pre-flight

```sh
NS=<NAMESPACE>

# Stack present?
kubectl -n $NS get sc juicefs-tether
kubectl -n $NS get pvc tether-data
kubectl -n $NS get sts tether
kubectl -n $NS get pods -l app=tether -o wide

# PVC bound, both replicas Running?
kubectl -n $NS wait --for=condition=Ready pod/tether-0 --timeout=120s
kubectl -n $NS wait --for=condition=Ready pod/tether-1 --timeout=120s
```

If anything above fails, stop here — the rest assumes a healthy 2-replica
StatefulSet with the JuiceFS PVC bound RWX.

## Step 1 — mount sanity

Both replicas should see the same `/data` mount and the dirs the
entrypoint created.

```sh
kubectl -n $NS exec tether-0 -- ls -ld /data /data/.tether /data/.claude/projects
kubectl -n $NS exec tether-1 -- ls -ld /data /data/.tether /data/.claude/projects
```

**PASS criteria**:
- Both pods print the same inode for each path:
  ```sh
  kubectl -n $NS exec tether-0 -- stat -c '%i %n' /data/.tether
  kubectl -n $NS exec tether-1 -- stat -c '%i %n' /data/.tether
  # → identical inode number
  ```
- `/data/.tether` mode is `drwx------` (0700).

**FAIL → debug**: if inodes differ, the CSI driver is mounting two
distinct filesystems. Check the JuiceFS Secret values are identical
across replicas.

## Step 2 — cross-replica JSONL visibility

Each replica writes a sentinel JSONL "envelope" file under
`~/.claude/projects/<bucket>/<sid>.jsonl`. The other replica must see
it within ≤ 5s (JuiceFS metadata-cache TTL is 1s with the
`entry-cache=1` mount option from `00-storageclass.yaml`).

```sh
SID0=$(uuidgen)
SID1=$(uuidgen)
BUCKET="-data-tether-verify"   # any string-safe bucket name

kubectl -n $NS exec tether-0 -- sh -c "
  mkdir -p /data/.claude/projects/$BUCKET
  printf '{\"sid\":\"$SID0\",\"from\":\"tether-0\",\"ts\":%s}\n' \$(date +%s) \
    > /data/.claude/projects/$BUCKET/$SID0.jsonl
"

kubectl -n $NS exec tether-1 -- sh -c "
  mkdir -p /data/.claude/projects/$BUCKET
  printf '{\"sid\":\"$SID1\",\"from\":\"tether-1\",\"ts\":%s}\n' \$(date +%s) \
    > /data/.claude/projects/$BUCKET/$SID1.jsonl
"

sleep 3

# Each replica must see BOTH files:
kubectl -n $NS exec tether-0 -- ls -la /data/.claude/projects/$BUCKET/
kubectl -n $NS exec tether-1 -- ls -la /data/.claude/projects/$BUCKET/

# Content cross-check — replica-0 reads what replica-1 wrote, and vice versa.
kubectl -n $NS exec tether-0 -- cat /data/.claude/projects/$BUCKET/$SID1.jsonl
kubectl -n $NS exec tether-1 -- cat /data/.claude/projects/$BUCKET/$SID0.jsonl
```

**PASS criteria**: each `cat` prints the JSON written by the *other*
replica. Both `ls` outputs list two files with the same byte sizes.

## Step 3 — survivor-replica visibility after pod kill

The hard claim: kill replica-0, replica-1 must still see all files
replica-0 wrote.

```sh
# Snapshot replica-1's view BEFORE the kill.
kubectl -n $NS exec tether-1 -- ls /data/.claude/projects/$BUCKET/ | sort > /tmp/before.txt

# Kill replica-0. We use --force --grace-period=0 to simulate a hard crash,
# not a clean SIGTERM — that's the more honest C'' test.
kubectl -n $NS delete pod tether-0 --grace-period=0 --force

# replica-0 will be recreated by the StatefulSet. We don't wait for it
# here — the question is what replica-1 sees while replica-0 is gone.

kubectl -n $NS exec tether-1 -- ls /data/.claude/projects/$BUCKET/ | sort > /tmp/after.txt

diff /tmp/before.txt /tmp/after.txt
```

**PASS criteria**: `diff` is empty. Replica-1's view of the JSONL
directory is identical before and after replica-0 went down.

**FAIL → debug**: if `after.txt` is missing files, the JuiceFS
metadata engine is not durable. Most common cause: Redis without AOF
fsync, lost the WAL on the kill. Set
`appendonly yes` + `appendfsync everysec` on the metadata Redis.

Wait for replica-0 to come back before continuing:

```sh
kubectl -n $NS wait --for=condition=Ready pod/tether-0 --timeout=120s

# replica-0 should also see what replica-1 wrote during its absence:
kubectl -n $NS exec tether-0 -- cat /data/.claude/projects/$BUCKET/$SID1.jsonl
```

## Step 4 — `lock.log` append-only invariant

The audit log (`~/.tether/.../lock.log`, see
`tether/internal/lock/doc.go:35`) MUST be append-only. We verify by
recording the file's inode + size before and after a write, and
ensuring the inode is stable while the size grows monotonically.

> Note: in v0.1, the daemon does not yet persist `lock.log` itself
> (out-of-scope marker in `internal/lock/doc.go`). We simulate the
> daemon's writer here so the FILESYSTEM-side invariant is verified
> independently of when the daemon code lands.

```sh
LOG=/data/.tether/lock.log

kubectl -n $NS exec tether-0 -- sh -c "
  : > $LOG
  chmod 0600 $LOG
"

# Record initial inode and size.
INO0=$(kubectl -n $NS exec tether-0 -- stat -c '%i' $LOG)
SIZE0=$(kubectl -n $NS exec tether-0 -- stat -c '%s' $LOG)

# Append from BOTH replicas (simulates the daemon writing audit events).
for i in 1 2 3 4 5; do
  kubectl -n $NS exec tether-0 -- sh -c "echo 'evt-0-$i' >> $LOG"
  kubectl -n $NS exec tether-1 -- sh -c "echo 'evt-1-$i' >> $LOG"
done

INO1=$(kubectl -n $NS exec tether-0 -- stat -c '%i' $LOG)
SIZE1=$(kubectl -n $NS exec tether-0 -- stat -c '%s' $LOG)

echo "inode before: $INO0   inode after: $INO1   (must be equal)"
echo "size before:  $SIZE0  size after:  $SIZE1   (after must be > before)"
```

**PASS criteria**:
- `INO0 == INO1` (no in-place rewrite — that would create a new inode).
- `SIZE1 > SIZE0` (monotonic growth).
- `wc -l $LOG` shows 10 lines (5 from each replica). Note: byte-level
  interleaving without `O_APPEND` coordination is undefined and is
  what the future leader-election ticket addresses; for the
  filesystem invariant check, line count is sufficient.

```sh
kubectl -n $NS exec tether-0 -- wc -l $LOG  # → 10
```

## Step 5 — file-mode round-trip

JuiceFS must preserve POSIX modes. The 0600 on `lock.log` and 0700 on
`.tether/` are the load-bearing ones (the daemon refuses to start if
these are world-readable, per the security stance in
`tether/internal/agent/attach_socket.go`).

```sh
kubectl -n $NS exec tether-0 -- stat -c '%a %n' /data/.tether
kubectl -n $NS exec tether-0 -- stat -c '%a %n' /data/.tether/lock.log

kubectl -n $NS exec tether-1 -- stat -c '%a %n' /data/.tether
kubectl -n $NS exec tether-1 -- stat -c '%a %n' /data/.tether/lock.log
```

**PASS criteria**:
- `.tether` → mode `700`
- `lock.log` → mode `600`
- Both replicas report the same mode (proves the mode is on the
  filesystem, not pod-local cache).

## Step 6 — clean up the verification artifacts

```sh
kubectl -n $NS exec tether-0 -- rm -rf /data/.claude/projects/$BUCKET
kubectl -n $NS exec tether-0 -- rm -f /data/.tether/lock.log
```

## Verdict matrix

| Step | What it proved |
|---|---|
| 1 | JuiceFS PVC mounted on both replicas as the same filesystem (same inode root). |
| 2 | Cross-replica JSONL visibility within metadata-cache TTL. |
| 3 | Surviving replica's view is durable across a forced pod kill (metadata engine is HA). |
| 4 | `lock.log` is append-only at the filesystem level; size grows, inode stable. |
| 5 | POSIX mode bits round-trip cleanly through JuiceFS. |
| (n/a — limitation) | Cross-pod **attach** is NOT verified here. The attach socket lives on `emptyDir`, not on JuiceFS, by design. Cross-pod attach is the future Phase-11+ cross-network WT path. |

If all of 1–5 pass, the C'' slice is green for v0.1. Scale the
StatefulSet to `replicas=1` per `juicefs-pv.md` Limitation #2 before
handing the deployment to real users.

## Troubleshooting cheat sheet

| Symptom | Likely cause |
|---|---|
| PVC stuck `Pending` | StorageClass / Secret namespace mismatch, or JuiceFS CSI driver not installed in the cluster. |
| Both pods `CrashLoopBackOff` | Daemon can't write `/data` — check `fsGroup: 65532` is honoured by the CSI mount; or `<JUICEFS_META_URL>` is unreachable. |
| Step 2 `cat` returns empty | metadata-cache miss. Reduce `entry-cache` mount option, or wait > TTL between write and read. |
| Step 3 `diff` non-empty | Metadata engine lost data on the kill. Enable Redis AOF (`appendonly yes`, `appendfsync everysec`). |
| Step 4 inode changed | Some component is rewriting `lock.log` in-place (likely the operator's own helper script, not the daemon). Audit `kubectl exec` history. |
| Step 5 mode is `666` or `777` | JuiceFS mount missed the `juicefs/mount-options` parameters. Re-check `00-storageclass.yaml`. |
