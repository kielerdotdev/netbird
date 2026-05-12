# Rootful reverse-proxy image (`Dockerfile.rootful`)

This fork publishes a non-distroless image at `ghcr.io/kielerdotdev/netbird-reverse-proxy` so the NetBird reverse proxy can attach to **kernel WireGuard** on the host (via tooling such as nftables/iptables and root-level net admin), aiming to bypass the rough **~200 Mbps** ceiling commonly seen with **userspace WireGuard**.

The stock `netbirdio/reverse-proxy` image stays **distroless** and unchanged; builds use `proxy/Dockerfile`.

## Compatibility with upstream runtime

This image intentionally matches the distroless wrapper:

- **`ENTRYPOINT`**: `["/go/bin/netbird-proxy"]` (same absolute path so `NB_PROXY_*` behaviour matches `proxy/Dockerfile`)
- **`NB_PROXY_ADDRESS`**: default `:8443` (override in env as today)
- **Ports**: **`8443`** exposed
- **Paths**: **`/certs`** and **`/var/lib/netbird`** (writable at runtime)

## Build/runtime notes

- **Build stage** uses **`golang:1.25-alpine`** — this tracks `go.mod` and upstream proxy Docker tooling (templates that cited Go 1.23 will not compile this tree).
- **Runtime** is **`alpine:3.20`** with: `nftables`, `iptables`, `iproute2`, `wireguard-tools`, `ca-certificates`, `tzdata`.
- The process runs as **`root`** (explicit security trade-off vs upstream `USER netbird`).

## Docker Compose example

Swap the proxy service image and add capabilities/devices/volumes as needed:

```yaml
services:
  reverse-proxy:
    image: ghcr.io/kielerdotdev/netbird-reverse-proxy:latest
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
      - SYS_MODULE
      - SYS_RESOURCE
      - NET_RAW
    devices:
      - /dev/net/tun:/dev/net/tun
    volumes:
      - nb-proxy-certs:/certs
      - nb-proxy-data:/var/lib/netbird
      - /lib/modules:/lib/modules:ro
```

**Host requirement:** load the **`wireguard` kernel module** (and matching `/lib/modules` tree for that kernel).

## Automated builds (GitHub Actions)

`.github/workflows/rootful-proxy.yml` builds **`linux/amd64`** and **`linux/arm64`**, pushes to GHCR (`latest` and a tag matching the resolved NetBird version string).

Scheduled (`cron`) workflows are loaded from your **repository default branch** on GitHub. If `rootful-proxy` is not default, duplicate this workflow onto `main` or rely on **daily upstream merges** pushing to `rootful-proxy`, which retriggers **`on.push`**.

Upstream merge automation: `.github/workflows/sync-upstream.yml` merges `netbirdio/netbird` **`main`** into **`rootful-proxy`** daily (`workflow_dispatch` also). Optionally set Actions variable **`UPSTREAM_REPO`** at `OWNER/name` defaulting to **`netbirdio/netbird`** (use your own upstream fork if desired).

## Local smoke test

```bash
docker build -f proxy/Dockerfile.rootful -t test-proxy .
docker run --rm test-proxy nft --version
docker run --rm test-proxy iptables --version
docker run --rm test-proxy id    # uid=0(root)
```

(Requires Docker.)
