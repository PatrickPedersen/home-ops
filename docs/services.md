# Services and connection info

How apps reach each other, and where people reach them.

## App to app: use the cluster address

When one app connects to another (Radarr → qBittorrent, BookOrbit → Prowlarr, Seerr → Jellyfin, ...), use the in-cluster address:

```
http://<service>.<namespace>.svc.cluster.local:<port>
```

Within the same namespace the short form `http://<service>:<port>` also works.

Don't use the `*.patrickpedersen.tech` hostnames for this. The *arr UIs sit behind Authentik forward auth, and the cluster address skips Envoy, Cloudflare and DNS entirely.

| App            | Cluster address                                      | API key / credentials                                                                                                            |
| -------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Radarr         | `http://radarr.media.svc.cluster.local:7878`         | Settings → General → Security → API Key                                                                                          |
| Sonarr         | `http://sonarr.media.svc.cluster.local:8989`         | Settings → General → Security → API Key                                                                                          |
| Prowlarr       | `http://prowlarr.media.svc.cluster.local:9696`       | Settings → General → Security → API Key                                                                                          |
| Bazarr         | `http://bazarr.media.svc.cluster.local:6767`         | Settings → General → Security → API Key                                                                                          |
| qBittorrent    | `http://qbittorrent.media.svc.cluster.local:8080`    | WebUI username and password                                                                                                      |
| rdtclient      | `http://rdtclient.media.svc.cluster.local:6500`      | its own login                                                                                                                    |
| FlareSolverr   | `http://flaresolverr.media.svc.cluster.local:8191`   | none                                                                                                                             |
| VPN HTTP proxy | `http://qbittorrent.media.svc.cluster.local:8888`    | none; Gluetun's proxy, exits through ProtonVPN. In Prowlarr: indexer proxy `ProtonVPN`, applies to indexers tagged `vpn`         |
| Jellyfin       | `http://jellyfin.media.svc.cluster.local:8096`       | Dashboard → API Keys                                                                                                             |
| Seerr          | `http://seerr.media.svc.cluster.local:5055`          | Settings → General → API Key                                                                                                     |
| BookOrbit      | `http://bookorbit.media.svc.cluster.local:3000`      |                                                                                                                                  |
| Authentik      | `http://authentik-server.security.svc.cluster.local` |                                                                                                                                  |
| Authentik LDAP | `ak-outpost-ldap.security.svc.cluster.local:389`     | bind user `cn=ldap_bind,ou=users,dc=home,dc=patrickpedersen,dc=tech`, password in 1Password (`authentik` → `LDAP_BIND_PASSWORD`) |

The *arr API keys can also be read from the terminal:

```sh
kubectl -n media exec deploy/<app> -- sed -n 's:.*<ApiKey>\(.*\)</ApiKey>.*:\1:p' /config/config.xml
```

### Finding it for a new app

- The service is named after the app; the port is in the app's `helmrelease.yaml` under `service`.
- List everything: `kubectl get svc -A`

## People: hostnames

All on `patrickpedersen.tech`.

| Hostname                                 | Reachable from              | Login                  |
| ---------------------------------------- | --------------------------- | ---------------------- |
| `auth`                                   | internet (Cloudflare)       | Authentik              |
| `bookorbit`                              | internet (Cloudflare)       | Authentik (OIDC)       |
| `seerr`                                  | internet (Cloudflare)       | Jellyfin account       |
| `status`                                 | internet (Cloudflare)       | none (Gatus)           |
| `jellyfin`                               | internet (Pangolin) and LAN | Authentik (LDAP)       |
| `radarr`, `sonarr`, `prowlarr`, `bazarr` | LAN                         | Authentik forward auth |
| `qbittorrent`, `rdtclient`               | LAN                         | the app's own login    |
| `games`                                  | LAN                         | none                   |
| `flux-operator`, `hubble`                | LAN                         | none                   |

Which gateway a route uses decides who can reach it: `envoy-external` is published through Cloudflare, `envoy-internal` only resolves on the LAN (via k8s-gateway). Jellyfin is the exception: it is internal in the cluster but published by Pangolin (Newt), with target `jellyfin.media.svc.cluster.local:8096`.

## Access groups (Authentik)

| Group             | Gives                                              |
| ----------------- | -------------------------------------------------- |
| `media`           | Jellyfin login                                     |
| `media_admin`     | Jellyfin admin, and the *arr apps via forward auth |
| `bookorbit`       | BookOrbit login                                    |
| `bookorbit_admin` | BookOrbit login plus every `book_*` permission     |

Groups are defined in `kubernetes/apps/security/authentik/app/blueprints/`; membership is managed in the Authentik UI.
