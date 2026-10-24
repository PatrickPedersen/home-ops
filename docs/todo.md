# To do

Open work after the Proxmox → Talos/TrueNAS migration. Last reviewed 2026-10-19.

## 1. NAS stability (blocking)

The NAS reset itself on 2026-10-18 under restic load. The kernel logged an **uncorrected machine check in the CPU** (AMD Zen 2, `870f10`, bank 5 = execution unit, status `bea0000000000108`). restic crashed twice before that, with SIGSEGV at unrelated places in two different builds.

Hardware, read from the running system on 2026-10-19:

| Part      | Value                                                                                                         |
| --------- | ------------------------------------------------------------------------------------------------------------- |
| CPU       | AMD Ryzen 5 3600 (6 cores, 3.6 / 4.2 GHz boost), microcode `0x8701030`                                        |
| Mainboard | ASRock B550M Pro4, BIOS P3.20 (2023-09-27)                                                                    |
| RAM       | 2× 16 GB Corsair Vengeance LPX `CMK32GX4M2Z3600C18`, running at **2667 MT/s, 1.2 V (JEDEC, XMP off)**, no ECC |

RAM overclocking is ruled out. The machine check points at a CPU core; RAM errors wouldn't be logged without ECC either way.

- [x] Note the CPU model and mainboard.
- [x] RAM at JEDEC default (XMP/DOCP off).
- [x] BIOS update: P3.20 → **P4.00** (2026-08-19), microcode `0x8701030` → `0x8701035` (done 2026-10-19).
- [x] After the update: defaults loaded, PBO explicitly _Disabled_, everything else Auto (RAM stays 2667 MT/s / 1.2 V).
- [x] Load test with restic after the BIOS update (2026-10-19): **failed**. SIGSEGV after 25 min, inside the Go runtime's heap bookkeeping (`writeHeapBitsSmall`), i.e. corrupted memory. That's the third crash, each at a different place. The first two came after ~8 min, so the fixes improved it but didn't solve it. This time the NAS stayed up, there was **no MCE**, and no other process segfaulted. Silent corruption with no CPU error makes the RAM the prime suspect.
- [ ] Memtest86+, at least one full pass (overnight). Now the next step: random memory corruption points at the RAM just as much as at the CPU.
- [ ] If it still faults: test with Core Performance Boost off (stable → CPU degraded at boost voltage). Note the CPU number of every new MCE (same core each time → faulty core).
- [ ] If faults persist: power supply, then replace the CPU.

**Until the NAS is stable:**

- No scrub of `vault`, no `zpool upgrade`, no big write jobs.
- restic on TrueNAS stays paused.
- Optionally stop qBittorrent (the heaviest writer).

## 2. Off-site backup of `vault` (after section 1)

The B2 copy of `Data`, `website` and `backup` is from Proxmox on 2026-10-17 and is not being updated. See [truenas-migration.md](truenas-migration.md), step 8.

- [x] Add `export RESTIC_PROGRESS_FPS=0.016` to `/mnt/vault/restic/backup.sh` (after `. env`).
- [ ] First full run with restic 0.19.1 via `systemd-run` (also serves as the load test).
- [ ] If the CPU fault was confirmed: `restic check --read-data-subset=10%` on the repository.
- [ ] Cron job: daily 04:00, stderr not hidden.
- [ ] TrueNAS email alerts (System → Email), so cron failures are noticed.
- [ ] Delete the leftover `/home/truenas_admin/restic`.

## 3. TrueNAS follow-ups

- [ ] Check scrub schedule and SMART test tasks (after section 1).
- [ ] Don't upgrade `vault` for a few weeks; decide afterwards.
- [ ] Delete `/mnt/vault/backup/proxmox` once the Minecraft VM has run fine for a few weeks (38 GB, also in the B2 backup).
- [ ] Verify the management source route survives reboots: `ip rule | grep 99.10` and the web UI opens from a desktop.
- Note: the WD disks show "Apollo (Exported)" in the UI. Cosmetic (stale detection; ZFS sees only `vault`). **Never wipe them.**

## 4. Minecraft VM

- [x] `zfs destroy -r Fast/vms/convert` (working space from the conversion).
- [x] Autostart on.
- [ ] Periodic snapshot task on `Fast/vms` (daily, keep 14 days).
- [ ] Replication task `Fast/vms` → `vault/backup/vms` (daily).
- [ ] Off-site backup of the world directory (restic from inside the VM), if it matters.
- [x] Pangolin raw TCP:
    - `flags.allow_raw_resources: true` in `config/config.yml`
    - Traefik entrypoint `tcp-25565`
    - port `25565:25565` on the gerbil service
    - VPS firewall TCP 25565
    - Resource: Raw TCP 25565 → `<VM IP>:25565`
- [x] Optional: `mc.patrickpedersen.tech` as DNS-only record to the VPS.
- [ ] Test from outside (phone on mobile data).

## 5. Network and firewall (UniFi)

- [ ] Allow End Devices → `192.168.10.5` TCP 2049 (desktop NFS to `website`).
- [ ] Optional: End Devices → `192.168.99.10` TCP 443 (TrueNAS web UI).
- [ ] Set up logging/SNMP, then review the zones:
    - Servers ↔ Kubernetes is allow-all (temporary since 2026-10-17).
    - Storage VLAN 40 sits in the Server zone.
    - Must stay allowed: Technitium `192.168.10.252` → `192.168.30.22` UDP/TCP 53; Kubernetes (Newt) → Minecraft VM TCP 25565.
- [ ] Decide on the DMZ VLAN 20: nothing uses it any more (Minecraft moved to Pangolin).

## 6. Authentication

- [ ] MFA (TOTP or passkey) on the Authentik account: Authentik is public now.
- [ ] BookOrbit: after MFA, set `DISABLE_LOCAL_AUTH: "true"`. `patrick` is a superuser, so `siteadmin` isn't needed any more.
- [x] Bazarr: own login off. The UI only offers Basic/Form, so `auth.type: null` was set in `/config/config/config.yaml` (backup next to it as `config.yaml.bak-auth`). Saving Bazarr's General settings in the UI may set it back to Form; then repeat the edit. The API still requires its key.
- Optional: qBittorrent and rdtclient behind forward auth (two lines in `media/forward-auth.yaml`; they keep their own login too).

## 7. Backups and restore

- [ ] Test a CNPG restore (marked untested in [backup-and-restore.md](backup-and-restore.md)), e.g. Authentik into a scratch cluster.
- [x] Keep `RESTIC_PASSWORD` somewhere outside the cluster and the NAS as well.

## 8. Cluster

- [ ] openebs-hostpath volumes are **read-only from kubelet's view**. Containers get a writable mount, but kubelet can't chown/chmod them (`lchown …/volumes/kubernetes.io~local-volume/pvc-…: read-only file system` in `talosctl logs kubelet`). Effects:
    - `fsGroup` is never applied; apps only work because openebs creates the folders world-writable (`0777`).
    - `subPath` mounts fail with `failed to create subPath directory`. Worked around for Prometheus/Alertmanager with `disableMountSubPath: true`.
    - Fix on the Talos side: kubelet needs `/var/mnt/openebs/local` mounted read-write. Talos 1.14's `KubeletConfig` no longer has `extraMounts`, so find the current way (maybe a `UserVolumeConfig` for openebs, or the mount propagation of `/var/mnt`). Afterwards drop `disableMountSubPath` again.

## 9. Cleanup on the workstation

These contain password hashes / secrets:

- [x] `rm ~/bookorbit.sql`
- [x] `rm ~/Downloads/bazarr_backup_v1.6.2_2026.10.17_17.21.00.zip`
- [x] `rm ~/Downloads/{sonarr,radarr,prowlarr}_backup_*.zip` (migration backups with API keys and password hashes)

## Done (for reference)

- Talos node with SecureBoot/TPM, storage bond, Flux, VolSync + CNPG backups to B2 (verified nightly).
- Media stack, Seerr, BookOrbit and the games site migrated; lldap replaced by the Authentik LDAP outpost; *arr behind forward auth.
- Proxmox replaced by TrueNAS: `vault` imported, NFS for cluster (`192.168.40.2`, MTU 9000) and desktops (`192.168.10.5`), `Fast` NVMe pool with qBittorrent scratch and the Minecraft VM.
- Seerr points at the cluster services; Jellyfin uses QSV and trusts `10.42.0.0/16`.
