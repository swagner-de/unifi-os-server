# UniFi OS Server

<a href="https://github.com/swagner-de/unifi-os-server/pkgs/container/unifi-os-server"><img src="https://img.shields.io/badge/ghcr.io-unifi--os--server-blue?logo=github&label=Container"></a>
<a href="https://github.com/swagner-de/unifi-os-server/actions/workflows/publish.yaml"><img src="https://img.shields.io/github/actions/workflow/status/swagner-de/unifi-os-server/publish.yaml?logo=githubactions&logoColor=white&label=Publish"></a>

Run [UniFi OS Server](https://blog.ui.com/article/introducing-unifi-os-server) directly in Docker or Kubernetes.

> The **UniFi OS Server is the new standard for self-hosting UniFi**, replacing the legacy UniFi Network Server. While the Network Server provided basic hosting functionality, it lacked support for key UniFi OS features like Organizations, IdP Integration, or Site Magic SD-WAN. With a fully unified operating system, UniFi OS Server now delivers the same management experience as UniFi-native–including CloudKeys, Cloud Gateways, and Official UniFi Hosting–and is fully compatible with Site Manager for centralized, multi-site control.
>
> <https://help.ui.com/hc/en-us/articles/34210126298775-Self-Hosting-UniFi>

# Installation

## Docker Compose

See [docker-compose.yaml](https://github.com/swagner-de/unifi-os-server/blob/main/docker-compose.yaml)

## Kubernetes

Install the Helm chart (published as an OCI artifact):

```bash
helm install unifi-os-server oci://ghcr.io/swagner-de/unifi-os-server \
  --namespace unifi --create-namespace
```

The chart runs in **privileged mode** (required by UniFi OS Server's internal
systemd). See the [chart README](https://github.com/swagner-de/unifi-os-server/blob/main/charts/unifi-os-server/README.md)
for security considerations, device adoption, ports, and how to supply
NetworkPolicies or extra volumes.

# Parameters

## Environment Variables

| Environment | Description |
|----|----|
| UOS_SYSTEM_IP | Hostname or IP for UniFi OS Server |
| HARDWARE_PLATFORM | Manually set hardware platform |

### UOS_SYSTEM_IP

Set UniFi OS Server hostname (recommended) or IP address for inform. To adopt device:

1. SSH into device with username/password: `ubnt`/`ubnt`
2. Set inform address:

   ```bash
   set-inform http://$UOS_SYSTEM_IP:8080/inform
   ```

### HARDWARE_PLATFORM

Overrides your detected hardware platform. Accepted values are: `synology`.

## Ports

| Protocol | Port | Direction | Usage |
|----|----|----|----|
| TCP | 11443 | Ingress | UniFi OS Server GUI/API |
| TCP | 5005 | Ingress | RTP (Real-time Transport Protocol) control protocol |
| TCP | 9543 | Ingress | UniFi Identity Hub |
| TCP | 6789 | Ingress | UniFi mobile speed test |
| TCP | 8080 | Ingress | Device and application communication |
| TCP | 8444 | Ingress | Secure Portal for Hotspot |
| UDP | 3478 | Both | STUN for device adoption and communication *(also required for Remote Management)* |
| UDP | 5514 | Ingress | Remote syslog capture |
| UDP | 10003 | Ingress | Device discovery during adoption |
| TCP | 28082 | Ingress | Device support files download |
| TCP | 5671 | Ingress | AQMPS |
| TCP | 8880 | Ingress | Hotspot portal redirection (HTTP) |
| TCP | 8881 | Ingress | Hotspot portal redirection (HTTP) |
| TCP | 8882 | Ingress | Hotspot portal redirection (HTTP) |

# How this image is built

This is a fork of [`lemker/unifi-os-server`](https://github.com/lemker/unifi-os-server)
with an automated, tested release pipeline:

- **Automated release tracking.** A scheduled workflow polls Ubiquiti's
  firmware API for new UniFi OS Server releases and opens a pull request that
  bumps the version. No manual version bumps.
- **Every release is boot-tested before publish.** When the release PR opens,
  a workflow builds the image for **both `linux/amd64` and `linux/arm64`**,
  boots it under systemd, waits for the system to reach a running state, and
  verifies the UniFi UI actually responds on port 443. The PR can only merge
  once these checks pass, so a broken build never reaches the registry.
- **One multiarch tag per version.** Publishing produces a single
  `:<version>` manifest (plus `:latest`) covering amd64 and arm64 — not
  separate per-arch tags.
- **Merge-gated publishing.** The image is built and pushed **only after** the
  release PR merges to `main`, never speculatively on every scheduled run.
- **A Helm chart ships alongside the image.** The
  [`charts/unifi-os-server`](https://github.com/swagner-de/unifi-os-server/tree/main/charts/unifi-os-server)
  chart is published as an OCI artifact to the same GHCR path and is
  version-bumped automatically by the release workflow (its `appVersion`
  tracks the UniFi OS Server version).

Images are published to
[`ghcr.io/swagner-de/unifi-os-server`](https://github.com/swagner-de/unifi-os-server/pkgs/container/unifi-os-server).

# Frequently Asked Questions

## How is the image assembled?

The base image is Ubiquiti's official UniFi OS Server, extracted directly from
the installer download (an ELF+ZIP polyglot that embeds an OCI image). On top
of that base, this repo layers [`uos-entrypoint.sh`](https://github.com/swagner-de/unifi-os-server/blob/main/uos-entrypoint.sh)
(originally from [lemker](https://github.com/lemker/unifi-os-server)),
which handles Docker/Kubernetes compatibility: persisting `UOS_UUID`, creating
the required `/usr/lib` metadata files, initializing log/data directories,
applying Synology overrides, and setting the system IP — all configurable via
environment variables. No Ubiquiti binaries are modified.

## Why does the container need specific settings for cgroup and tmpfs?

The underlying structure of UniFi OS Server runs every component as systemd services which requires access to the host `cgroup`.