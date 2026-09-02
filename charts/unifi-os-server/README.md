# unifi-os-server

Self-hosted UniFi OS Server Helm chart — UniFi Network with Organizations, IdP,
and Site Magic SD-WAN support.

## Important: Security Context

This chart runs in **privileged mode**. Standard Kubernetes security primitives
do **not** apply:

- `runAsNonRoot` — container runs systemd, requires root
- `readOnlyRootFilesystem` — systemd and services need writable paths everywhere
- `allowPrivilegeEscalation: false` — requires privilege escalation
- `capabilities.drop: [ALL]` — needs full capability set
- `seccompProfile: RuntimeDefault` — systemd syscalls would be blocked

UniFi OS Server runs every component as systemd services internally, which
requires host cgroup access (`hostPath /sys/fs/cgroup`) and tmpfs mounts
(`/run`, `/run/lock`). This is an inherent requirement of the upstream project
and cannot be worked around.

**Recommendation:** Run this in an isolated namespace. Restrict traffic with a
NetworkPolicy supplied via `extraManifests` (see below).

## Install

```bash
helm install unifi-os-server oci://ghcr.io/swagner-de/charts/unifi-os-server
```

## Device Adoption

After deployment, set the inform URL on your UniFi devices:

```bash
ssh admin@<device-ip>
set-inform http://<UOS_SYSTEM_IP>:8080/inform
```

By default, `UOS_SYSTEM_IP` is the pod IP. Override `uosSystemIP` if you use a
LoadBalancer or external DNS name for device communication.

## Ports

The `main` service exposes these ports (all on by default — remove any you don't
need under `service.main.extraPorts`):

| Port  | Protocol | Purpose                          |
|-------|----------|----------------------------------|
| 443   | TCP      | GUI and API (HTTPS)              |
| 8080  | TCP      | Device inform / communication    |
| 5005  | TCP      | RTP control                      |
| 9543  | TCP      | UniFi Identity Hub               |
| 6789  | TCP      | Mobile speed test                |
| 8444  | TCP      | Secure Portal for Hotspot        |
| 28082 | TCP      | Device support files             |
| 5671  | TCP      | AQMPS                            |
| 8880  | TCP      | Hotspot portal redirect (HTTP)   |
| 8881  | TCP      | Hotspot portal redirect (HTTP)   |
| 8882  | TCP      | Hotspot portal redirect (HTTP)   |
| 5514  | UDP      | Remote syslog                    |

The `stun` service (LoadBalancer) exposes:

| Port  | Protocol | Purpose                          |
|-------|----------|----------------------------------|
| 3478  | UDP      | STUN (required for remote mgmt)  |
| 10003 | UDP      | Device discovery                 |

`443`, `8080`, `3478`, and `10003` are the minimum required for adoption and
management; the rest are optional feature ports.

## Custom resources (NetworkPolicy, etc.)

Supply arbitrary manifests via `extraManifests`:

```yaml
extraManifests:
  - apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: unifi-os-server-ingress
    spec:
      podSelector:
        matchLabels:
          app.kubernetes.io/name: unifi-os-server
      policyTypes: [Ingress]
      ingress:
        - from:
            - ipBlock:
                cidr: 10.0.0.0/8
```

## Extra volumes (e.g. TLS cert injection)

```yaml
persistence:
  extraVolumes:
    - name: tls
      secret:
        secretName: unifi-os-server-tls
      mounts:
        - path: /data/unifi-core/config/unifi-core.crt
          subPath: tls.crt
          readOnly: true
```
