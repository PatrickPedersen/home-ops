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
- No SMB. NFS for the cluster (storage VLAN) and for Linux end devices (`192.168.100.0/24`).

### Addresses

| Purpose                  | Address                                    | VLAN | Notes                                              |
| ------------------------ | ------------------------------------------ | ---- | -------------------------------------------------- |
| Management (web UI, SSH) | `192.168.99.10/24`, gateway `192.168.99.1` | 99   | same as Proxmox today                              |
| Storage (cluster NFS)    | `192.168.40.2/24`, no gateway              | 40   | unchanged, so the cluster only needs a path change |
| NFS for end devices      | `192.168.10.5/24` (pick a free one)        | 10   | end devices can't reach the unrouted storage VLAN  |

All three are VLAN interfaces on a LACP link aggregation of the two 10G ports, like `vmbr0` on Proxmox (`bridge-vids 10 20 40 99`). MTU 1500.

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
4. Reboot into TrueNAS. On the console, set the management interface to `192.168.99.10/24` so the web UI comes up where it was.
5. Shut down, **plug the WD disks back in**, boot. Check Storage → Disks lists both serials (`WX82DA1EC1RY`, `WX92DA1631PK`) before importing.

## 5. Configure TrueNAS

**Network** (Network → Interfaces):

- Link aggregation `bond0`: LACP, the two 10G ports, no address.
- VLAN 99 on `bond0`: `192.168.99.10/24`; global default gateway `192.168.99.1`, DNS `192.168.10.252`.
- VLAN 40 on `bond0`: `192.168.40.2/24`.
- VLAN 10 on `bond0`: `192.168.10.5/24`.
- System → General → GUI: bind the web UI to `192.168.99.10` only.

Test before going on: `ping 192.168.40.10` (k8s node) from the TrueNAS shell.

**Pool**: Storage → Import Pool → `vault`. It mounts at `/mnt/vault`.

> [!WARNING]
> **Don't click "Upgrade pool"** until everything has run fine for a few weeks. Upgrading enables features older ZFS (e.g. a Proxmox reinstall) can't import, which is your way back.

**Identity** (Credentials → Local Groups / Users):

- Group `media`, GID `100000`.
- User `media`, UID `100000`, primary group `media`, no password, login disabled.

**NFS** (Shares → NFS), one share per folder:

| Path                 | Networks                              | Mapall user / group | Read only |
| -------------------- | ------------------------------------- | ------------------- | --------- |
| `/mnt/vault/media`   | `192.168.40.0/24`, `192.168.100.0/24` | `media` / `media`   | no        |
| `/mnt/vault/website` | `192.168.40.0/24`, `192.168.100.0/24` | `media` / `media`   | no        |

Mapall makes every client act as `100000`, which is what the apps run as and owns the files. Desktops can then edit freely without matching UIDs, and remote root never acts as root on the pool.

NFS service settings: NFSv4 enabled, bind to `192.168.40.2` and `192.168.10.5`.

**UniFi firewall**: allow End Devices (`192.168.100.0/24`) → `192.168.10.5` TCP 2049.

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

## Later

- Wipe the NVMe disks and create a fast pool if wanted.
- Delete `/mnt/vault/backup/proxmox` once Minecraft has a new home or isn't needed.
- Scrub schedule and SMART tests (TrueNAS sets defaults; check Data Protection).
