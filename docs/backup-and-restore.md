# Backup and restore

Everything on the node's local disk (`openebs-hostpath`) is backed up to Backblaze B2.
Media on the NFS share (`/vault/media`) is **not** covered here. Protecting it is the job of ZFS on the storage server.

| What                    | Tool                              | Schedule                     | Where in B2                   | Retention                    |
| ----------------------- | --------------------------------- | ---------------------------- | ----------------------------- | ---------------------------- |
| App volumes (`media/*`) | VolSync (restic)                  | daily 03:00                  | `<bucket>/volsync/<app>/`     | 7 daily, 4 weekly, 3 monthly |
| Authentik Postgres      | CloudNativePG barman-cloud plugin | continuous WAL + daily 03:00 | `<bucket>/cnpg/authentik-pg/` | 30 days                      |

Gatus is intentionally not backed up, since its uptime history isn't worth keeping.

## Where things are

- VolSync component: `kubernetes/components/volsync/`. Each app's `ks.yaml` adds it with
  `postBuild.substitute: { APP, VOLSYNC_UID, VOLSYNC_GID }`. The PVC must be named the same as the app.
- App UIDs: `100000` for most media apps (owner of the existing media files), `1000` for `prowlarr` and `seerr`.
- Postgres backup: `kubernetes/apps/security/authentik/app/{objectstore,scheduledbackup,cluster}.yaml`.

### Secrets (1Password, vault `Kubernetes`)

| Item              | Fields                                                                        |
| ----------------- | ----------------------------------------------------------------------------- |
| `backblaze`       | `B2_KEY_ID`, `B2_APPLICATION_KEY`, `RESTIC_PASSWORD`                          |
| `cluster-secrets` | `B2_BUCKET`, `B2_ENDPOINT` (no `https://`, **no leading or trailing spaces**) |

> [!IMPORTANT]
> Keep a copy of `RESTIC_PASSWORD` **outside** the cluster as well. Without it the VolSync backups cannot be decrypted, and in a full rebuild you need it before 1Password Connect is running.

After changing a value in 1Password, the in-cluster copy only refreshes on its interval. Force it:

```sh
kubectl -n <namespace> annotate externalsecret cluster-secrets force-sync=$(date +%s) --overwrite
flux -n <namespace> reconcile ks <app>
```

A stale `cluster-secrets` shows up in Flux as `variable not set (strict mode): "B2_..."`.

## Checking backups

```sh
# VolSync: LAST SYNC / NEXT SYNC per app
kubectl -n media get replicationsource

# Result and restic summary of the latest run
kubectl -n media get replicationsource <app> -o jsonpath='{.status.latestMoverStatus.result}{"\n"}'
kubectl -n media get replicationsource <app> -o jsonpath='{.status.latestMoverStatus.logs}'

# Postgres: WAL archiving and backups
kubectl -n security get cluster authentik-pg -o json \
  | jq -r '.status.conditions[] | select(.type=="ContinuousArchiving") | "\(.status) \(.reason)"'
kubectl -n security get backup
```

`ContinuousArchiving` must be `True ContinuousArchivingSuccess`. While it fails, Postgres keeps its WAL files on the local 5Gi volume until the upload works again, and the database stops if that volume fills.

### Run a backup now

```sh
# VolSync: setting `manual` to a new value triggers a run
T=manual-$(date +%s)
kubectl -n media patch replicationsource <app> --type merge -p "{\"spec\":{\"trigger\":{\"manual\":\"$T\"}}}"

# wait until the run is done: lastManualSync equals $T
kubectl -n media get replicationsource <app> -o jsonpath='{.status.lastManualSync}{"\n"}'

# then remove the manual trigger again
kubectl -n media patch replicationsource <app> --type json -p '[{"op":"remove","path":"/spec/trigger/manual"}]'
```

> [!WARNING]
> While `spec.trigger.manual` is set, the nightly schedule is **paused**. Always remove it afterwards. Flux won't do it for you, because it doesn't remove fields that `kubectl` added.

```sh
# Postgres
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: authentik-pg-manual-$(date +%Y%m%d%H%M)
  namespace: security
spec:
  cluster:
    name: authentik-pg
  method: plugin
  pluginConfiguration:
    name: barman-cloud.cloudnative-pg.io
EOF
```

If a VolSync run failed and keeps retrying with backoff, delete its job to start over right away:
`kubectl -n media delete job volsync-src-<app>`.

## Restore an app volume (VolSync)

Tested on 2026-10-17 with Prowlarr: file-identical restore, SQLite `integrity_check: ok`.

1. **Stop the app** so nothing writes to the volume during the restore. This also covers a disaster rebuild: let Flux create the app (and its empty PVC), then stop it.

    ```sh
    flux -n media suspend hr <app>      # otherwise Flux/Helm may scale it back up
    kubectl -n media scale deploy <app> --replicas 0
    ```

2. **Restore the latest backup into the app's PVC.** Use the app's UID: `100000`, or `1000` for prowlarr/seerr.

    ```sh
    APP=<app>; UID_=<uid>
    cat <<EOF | kubectl apply -f -
    apiVersion: volsync.backube/v1alpha1
    kind: ReplicationDestination
    metadata:
      name: ${APP}-restore
      namespace: media
    spec:
      trigger:
        manual: restore-$(date +%s)
      restic:
        repository: ${APP}-volsync-secret
        copyMethod: Direct
        destinationPVC: ${APP}
        cacheStorageClassName: openebs-hostpath
        cacheAccessModes: ["ReadWriteOnce"]
        cacheCapacity: 1Gi
        moverSecurityContext:
          runAsUser: ${UID_}
          runAsGroup: ${UID_}
          fsGroup: ${UID_}
        # Optional: restore an older snapshot instead of the latest
        # restoreAsOf: "2026-10-15T00:00:00Z"   # newest snapshot at or before this time
        # previous: 1                           # skip N newest snapshots
    EOF
    ```

3. **Wait for it** and check the result:

    ```sh
    kubectl -n media get replicationdestination ${APP}-restore \
      -o jsonpath='{.status.latestMoverStatus.result}{"\n"}'      # expect: Successful
    kubectl -n media get replicationdestination ${APP}-restore \
      -o jsonpath='{.status.latestMoverStatus.logs}' | grep "restoring snapshot"
    ```

4. **Clean up and start the app:**

    ```sh
    kubectl -n media delete replicationdestination ${APP}-restore
    flux -n media resume hr ${APP}
    kubectl -n media scale deploy ${APP} --replicas 1
    ```

> [!NOTE]
> `copyMethod: Direct` restores **on top of** the PVC's current contents. Files that exist in the volume but not in the snapshot are left in place. For a clean restore into a non-empty volume, empty it first with a temporary pod that mounts the PVC.

### Jellyfin

Jellyfin mounts its PVC through the subfolders `data`, `etc` and `log`. Kubernetes fails to create those on a **fresh** volume (`failed to create subPath directory`). The backup contains them, so restoring with the steps above recreates them. If you ever start Jellyfin on an empty volume instead, create the three folders first (owner `100000`).

### Restore test without touching the app

To check a backup without stopping anything, restore into a throwaway PVC instead of the app's own:

```sh
kubectl -n media apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: <app>-restore-test, namespace: media }
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: openebs-hostpath
  resources: { requests: { storage: 1Gi } }
EOF
```

Then run step 2 with `destinationPVC: <app>-restore-test`. Compare the files with a pod that mounts both PVCs **read-only**, for example `PRAGMA integrity_check;` on a copy of the SQLite file. Delete the `ReplicationDestination` and the test PVC afterwards.

## Browse backups with the restic CLI

Useful to list snapshots or fetch a single file without Kubernetes. The values come from the 1Password items above.

```sh
export AWS_ACCESS_KEY_ID=<B2_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<B2_APPLICATION_KEY>
export RESTIC_PASSWORD=<RESTIC_PASSWORD>
export RESTIC_REPOSITORY=s3:https://<B2_ENDPOINT>/<B2_BUCKET>/volsync/<app>

restic snapshots
restic ls latest
restic restore latest --target ./restore --include /data/config.xml
```

Inside a snapshot, the volume's contents are under `/data`.

## Restore the Authentik database (CloudNativePG)

> [!WARNING]
> **Not tested yet.** Only VolSync restores have been tested. Try this on a copy before you need it for real.

CloudNativePG restores by bootstrapping a **new** cluster from the backup. You can't restore into a running cluster.

1. Stop Authentik and delete the broken cluster:

    ```sh
    flux -n security suspend ks authentik
    kubectl -n security scale deploy authentik-server authentik-worker --replicas 0
    kubectl -n security delete cluster authentik-pg
    ```

2. In `kubernetes/apps/security/authentik/app/cluster.yaml`, replace `bootstrap.initdb` with a recovery from the existing archive. Give the new cluster's own archive a **new `serverName`**, because CloudNativePG refuses to write WAL into an archive that already holds another cluster's history:

    ```yaml
    spec:
        bootstrap:
            recovery:
                source: backup
                # recoveryTarget:            # optional point-in-time recovery
                #   targetTime: "2026-10-17 12:00:00+00"
        externalClusters:
            - name: backup
              plugin:
                  name: barman-cloud.cloudnative-pg.io
                  parameters:
                      barmanObjectName: authentik-pg
                      serverName: authentik-pg # where the existing backups are
        plugins:
            - name: barman-cloud.cloudnative-pg.io
              isWALArchiver: true
              parameters:
                  barmanObjectName: authentik-pg
                  serverName: authentik-pg-v2 # new archive path for the restored cluster
    ```

3. Commit, push, then `flux -n security resume ks authentik`. Wait for `kubectl -n security get cluster authentik-pg` to report a healthy state, then scale Authentik back up (or let Flux do it).

4. The `authentik-pg-app` secret (database password) is recreated by CloudNativePG and Authentik reads it from there. `AUTHENTIK_SECRET_KEY` comes from 1Password and must be the **same** as before.

Keep `bootstrap.recovery` and the new `serverName` in git afterwards. `bootstrap` only runs when a cluster is created, so it does nothing on an existing cluster.
