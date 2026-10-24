# Proxmox → TrueNAS switch (pve01)

Runbook for reinstalling pve01 as TrueNAS and importing the existing `vault` pool. Everything the cluster runs has already moved off Proxmox; this only replaces the storage host.

## Facts this plan is based on

| Disk                                                                  | Holds                               | During the switch                        |
| --------------------------------------------------------------------- | ----------------------------------- | ---------------------------------------- |
| 2× WD Red 4 TB (`sdb`, `sdc`, serials `WX82DA1EC1RY`, `WX92DA1631PK`) | `vault` (mirror, 2.93T used)        | **Never select.** Imported afterwards.   |
| Samsung 850 EVO 250 GB (`sda`, serial `S3R0NF0J838419B`)              | old TrueNAS `boot-pool` leftovers   | TrueNAS boot disk                        |
| 2× Kingston 1 TB NVMe                                                 | `rpool` (Proxmox, all VM/LXC disks) | Left unused; wiped later for a fast pool |

- `media` and `website` are **folders** in the root dataset `vault`, not datasets. NFS shares folders fine.
- Active pool features all exist since OpenZFS 2.2, so any TrueNAS with ZFS ≥ 2.2 imports `vault`.
- Everything written to the share is owned by UID/GID `100000`.
- No SMB. NFS for the cluster (storage VLAN); `website` is also shared with Linux end devices (`192.168.100.0/24`), `media` is not.

### Addresses

| Interface                   | Address                                               | Purpose                                                                    |
| --------------------------- | ----------------------------------------------------- | -------------------------------------------------------------------------- |
| `enp7s0` (untagged VLAN 10) | `192.168.10.5/24`, **default gateway** `192.168.10.1` | NFS for desktops; the NAS's own outbound traffic (updates, B2 backup, NTP) |
| `vlan99` on `enp7s0`        | `192.168.99.10/24`                                    | Management only: web UI, SSH. Never serves NFS                             |
| `bond0` (untagged VLAN 40)  | `192.168.40.2/24`, no gateway, MTU 9000               | Cluster NFS. Unchanged, so the cluster only needs a path change            |

- `enp7s0` (onboard): its switch port has native VLAN 10 and VLAN 99 tagged. MTU 1500.
- `bond0` is LACP over the two 10G ports, MTU 9000; UniFi Jumbo Frames must be on. Its switch ports have VLAN 40 as **native (untagged)**, like the k8s node's, so the address sits directly on `bond0`. A tagged `vlan40` interface receives nothing there.
- Two routed addresses with one default gateway means replies to the management IP would leave via `enp7s0`'s VLAN 10, and the UDM's stateful firewall drops such asymmetric traffic. A source-based route sends everything **from** `192.168.99.10` out via `192.168.99.1` instead (see step 5).

## 1. Before (Proxmox still running)

### Back up what lives on the NVMe

Only the Minecraft VM is kept. Dump it onto `vault`, which survives the reinstall:

```sh
mkdir -p /vault/backup/proxmox
vzdump 115 --dumpdir /vault/backup/proxmox --mode stop --compress zstd
ls -lh /vault/backup/proxmox
```

Anything else on the system disk worth keeping (ISOs in `/var/lib/vz/template/iso`, `/root/network.log`) gets copied to `/vault/backup/proxmox` too.

### Delete migration leftovers (they contain secrets)

```sh
rm /root/{bookorbit.sql,jellyfin-migrate.tgz,qbit-migrate.tgz,rdtclient-db.tgz,seerr-migrate.tgz}
zfs destroy vault/subvol-101-disk-0
zfs destroy vault/subvol-111-disk-0
```

### Write down the network config

```sh
cat /etc/network/interfaces
ip -br link
```

Note which NIC names/MACs are the two 10G ports in the bond; TrueNAS will show the same MACs.

## 2. Stop the cluster apps that use NFS

From the workstation. A PV's NFS path can't be edited in place, so the volumes get deleted and recreated, and nothing may be using them:

```sh
for app in bazarr bookorbit jellyfin qbittorrent radarr rdtclient sonarr; do
  flux -n media suspend hr $app
  kubectl -n media scale deploy $app --replicas 0
done
flux -n default suspend hr games
kubectl -n default scale deploy games --replicas 0

kubectl get pods -A | grep -E 'bazarr|bookorbit|jellyfin|qbittorrent|radarr|rdtclient|sonarr|games'   # wait until none left
```

Everything else (Authentik, Seerr, Prowlarr, backups) keeps running.

## 3. Export the pool and shut down Proxmox

```sh
zpool export vault     # clean export: TrueNAS imports it without -f
zpool list             # vault must be gone
poweroff
```

If export says the pool is busy: `pct stop`/`qm stop` everything, then retry.

## 4. Install TrueNAS

1. **Unplug both WD Red disks** (SATA or power) with the machine off. The installer then can't see them, so a wrong pick can't wipe `vault`.
2. Boot the installer (current stable TrueNAS Community Edition).
3. Select `sda` (Samsung 850 EVO 250 GB). With the WD disks unplugged, the only other disks offered are the two NVMe.
4. Reboot into TrueNAS. Plug the onboard `enp7s0` into a switch port with native VLAN 10 and VLAN 99 tagged; it gets an address there (or set `192.168.10.5/24` on the console) to reach the web UI.
5. Shut down, **plug the WD disks back in**, boot. Check Storage → Disks lists both serials (`WX82DA1EC1RY`, `WX92DA1631PK`) before importing.

## 5. Configure TrueNAS

**Network** (Network → Interfaces). The installer's console setup is limited: give one port a temporary address on VLAN 99 (or plug the onboard `enp7s0` into a management port) to reach the web UI, then build the rest there. TrueNAS tests every network change and rolls it back unless confirmed.

- `enp7s0`: `192.168.10.5/24`, MTU 1500 (switch port: native VLAN 10, VLAN 99 tagged).
- VLAN `vlan99` on `enp7s0`: tag 99, `192.168.99.10/24`.
- Link aggregation `bond0`: LACP, members `enp4s0f0np0` + `enp4s0f1np1`, hash policy LAYER2+3, MTU 9000, `192.168.40.2/24`.
- Global configuration: default gateway `192.168.10.1`, nameserver `192.168.10.252`.
- System → Advanced → Init/Shutdown Scripts, type Command, when Post Init, so management replies go back through the management gateway:
  `ip route add default via 192.168.99.1 dev vlan99 table 99; ip rule add from 192.168.99.10 table 99`
- System → General → GUI: bind the web UI to `192.168.99.10` only. System → Services → SSH: bind to `vlan99` only.
- Proxmox disabled NIC offloading on both 10G ports (Intel X710, `i40e`) with `pre-up ethtool -K … tx off rx off tso off gso off gro off`. Not needed to get traffic flowing on TrueNAS; if NFS is slow or hangs, add it as an Init/Shutdown script (Command, Post Init): `for i in enp4s0f0np0 enp4s0f1np1; do ethtool -K $i tx off rx off tso off gso off gro off; done`

Test before going on, from the TrueNAS shell: `ping -c3 -M do -s 8972 192.168.40.10` (k8s node, full 9000-byte frames). If plain ping works but this doesn't, jumbo frames are off somewhere on the path.

**Pool**: Storage → Import Pool → `vault`. It mounts at `/mnt/vault`.

> [!WARNING]
> **Don't click "Upgrade pool"** until everything has run fine for a few weeks. Upgrading enables features older ZFS (e.g. a Proxmox reinstall) can't import, which is your way back.

**Identity** (Credentials → Local Groups / Users):

- Group `media`, GID `100000`.
- User `media`, UID `100000`, primary group `media`, no password, login disabled.

**NFS** (Shares → NFS), one share per folder:

| Path                 | Networks                              | Mapall user / group | Read only |
| -------------------- | ------------------------------------- | ------------------- | --------- |
| `/mnt/vault/media`   | `192.168.40.0/24` (cluster only)      | `media` / `media`   | no        |
| `/mnt/vault/website` | `192.168.40.0/24`, `192.168.100.0/24` | `media` / `media`   | no        |

Mapall makes every client act as `100000`, which is what the apps run as and owns the files. Desktops can then edit freely without matching UIDs, and remote root never acts as root on the pool.

NFS service settings: NFSv4 enabled, bind to `192.168.40.2` and `192.168.10.5` (never the management IP).

**UniFi firewall**: allow End Devices (`192.168.100.0/24`) → `192.168.10.5` TCP 2049. For the web UI from a desktop, also End Devices → `192.168.99.10` TCP 443.

## 6. Point the cluster at the new paths

Change the paths in git:

- `kubernetes/apps/media/media-nfs.yaml`: `/vault/media` → `/mnt/vault/media`
- `kubernetes/apps/default/games/app/website-nfs.yaml`: `/vault/website` → `/mnt/vault/website`

Delete the old volumes (their path is immutable), then push and let Flux recreate them:

```sh
kubectl -n media delete pvc media
kubectl -n default delete pvc website
kubectl delete pv media-nfs website-nfs
git push
flux reconcile ks cluster-apps -n flux-system --with-source
flux -n default reconcile ks games
kubectl get pv media-nfs website-nfs        # both Bound
```

Start the apps again:

```sh
for app in bazarr bookorbit jellyfin qbittorrent radarr rdtclient sonarr; do
  flux -n media resume hr $app
done
flux -n default resume hr games
```

## 7. Check

- `kubectl -n media exec deploy/radarr -- touch /data/media/.t && kubectl -n media exec deploy/radarr -- ls -ln /data/media/.t` shows owner `100000`; remove the file afterwards.
- Gatus all green; Jellyfin plays something; qBittorrent resumes seeding (it rechecks nothing if paths are unchanged inside the pod).
- From a desktop: `sudo mount -t nfs4 192.168.10.5:/mnt/vault/media /mnt/test`, create and delete a file.

## 8. Nightly off-site backup (restic → B2)

The irreplaceable folders (`Data`, `website`, `backup`, about 312 GB) go to the same B2 bucket as the cluster backups, under `vault/`. `media` is left out: it can be downloaded again, and would cost ten times as much to store.

The repository was created and first filled from Proxmox before the switch. TrueNAS only continues it.

TrueNAS has no package manager, so restic runs as a static binary from a root-only dataset on the pool. Not from the admin's home directory: TrueNAS mounts that `noexec`. restic 0.18.1 crashed (SIGSEGV while reading files) on TrueNAS 25.10; use a current release.

```sh
zfs create vault/restic
chmod 700 /mnt/vault/restic
cd /mnt/vault/restic
V=0.19.1   # check github.com/restic/restic/releases for the current one
curl -fsSLO https://github.com/restic/restic/releases/download/v$V/restic_${V}_linux_amd64.bz2
curl -fsSLO https://github.com/restic/restic/releases/download/v$V/SHA256SUMS
grep linux_amd64.bz2 SHA256SUMS | sha256sum -c   # must say OK
bunzip2 -c restic_${V}_linux_amd64.bz2 > restic && rm restic_${V}_linux_amd64.bz2 SHA256SUMS
chmod 700 restic
./restic version
```

Credentials, from 1Password (`backblaze`, `cluster-secrets`):

```sh
cat > /mnt/vault/restic/env <<'EOF'
export AWS_ACCESS_KEY_ID=<B2_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<B2_APPLICATION_KEY>
export RESTIC_PASSWORD=<RESTIC_PASSWORD>
export RESTIC_REPOSITORY=s3:https://<B2_ENDPOINT>/<B2_BUCKET>/vault
EOF
chmod 600 /mnt/vault/restic/env
```

The script:

```sh
cat > /mnt/vault/restic/backup.sh <<'EOF'
#!/bin/sh
# Nightly off-site backup of the irreplaceable folders. Run by a TrueNAS cron job.
set -eu
. /mnt/vault/restic/env
# Without a terminal restic is silent until the end; log progress once a minute.
export RESTIC_PROGRESS_FPS=0.016
R="/mnt/vault/restic/restic --cache-dir /mnt/vault/restic/cache"
$R backup --host vault /mnt/vault/Data /mnt/vault/website /mnt/vault/backup
# Pruning rewrites data in B2, so only once a week (Sunday).
if [ "$(date +%u)" = 7 ]; then
  $R forget --host vault --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
fi
EOF
chmod 700 /mnt/vault/restic/backup.sh
systemd-run --unit=restic-backup --property=Type=exec /mnt/vault/restic/backup.sh   # first run, detached from the terminal
journalctl -fu restic-backup                                                          # follow it
```

The first TrueNAS run reads all 312 GB again (new paths, new host name), but uploads almost nothing: restic deduplicates against what Proxmox already sent.

Schedule it in System → Advanced → Cron Jobs:

| Field                | Value                                                            |
| -------------------- | ---------------------------------------------------------------- |
| Command              | `/mnt/vault/restic/backup.sh`                                    |
| Run as               | `root`                                                           |
| Schedule             | daily, 04:00 (after the cluster backups at 03:00)                |
| Hide standard output | yes                                                              |
| Hide standard error  | **no**, so failures are mailed (needs System → Email configured) |

Check it now and then:

```sh
. /mnt/vault/restic/env && /mnt/vault/restic/restic snapshots --host vault
```

> [!IMPORTANT]
> `RESTIC_PASSWORD` is the only way to decrypt this backup. It's in 1Password, and must stay somewhere outside this machine too.

## 9. Metrics for Prometheus (ZFS and SMART)

TrueNAS can't install packages, but its Apps run Docker containers. Both exporters listen on the **storage IP only** (`192.168.40.2`), where Prometheus in the cluster scrapes them (`kube-prometheus-stack/app/scrapeconfig.yaml`). Nothing is exposed on VLAN 10 or management.

1. Apps → Configuration → Choose Pool: `Fast` (only needed once; app data and images live there).
2. Apps → Discover Apps → ⋮ → **Install via YAML**, name `exporters`:

    ```yaml
    services:
        # Host metrics incl. ZFS (pool state, ARC, I/O) from /proc/spl/kstat/zfs.
        node-exporter:
            image: quay.io/prometheus/node-exporter:v1.9.1
            command:
                - --path.rootfs=/host
                - --web.listen-address=192.168.40.2:9100
            network_mode: host
            pid: host
            volumes:
                - /:/host:ro,rslave
            restart: unless-stopped
        # Runs smartctl against every disk on each scrape (every 2 min).
        smartctl-exporter:
            image: quay.io/prometheuscommunity/smartctl-exporter:v0.14.0
            command:
                - --web.listen-address=192.168.40.2:9633
            network_mode: host
            privileged: true
            user: root
            restart: unless-stopped
    ```

3. Check from the TrueNAS shell:

    ```sh
    curl -s 192.168.40.2:9100/metrics | grep -m3 node_zfs_zpool_state
    curl -s 192.168.40.2:9633/metrics | grep -m3 smartctl_device_smart_status
    ```

4. In Prometheus (Status → Targets) both `nas01` targets should be up.

Renovate doesn't see these images; bump the tags by hand now and then.

## Later

- Wipe the NVMe disks and create a fast pool if wanted.
- Delete `/mnt/vault/backup/proxmox` once Minecraft has a new home or isn't needed.
- Scrub schedule and SMART tests (TrueNAS sets defaults; check Data Protection).
