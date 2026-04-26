# UniFi OS Server

<a href="https://github.com/buggerman/unifi-os-server/pkgs/container/unifi-os-server"><img src="https://img.shields.io/badge/dynamic/regex?url=https%3A%2F%2Fgithub.com%2Fbuggerman%2Funifi-os-server%2Fpkgs%2Fcontainer%2Funifi-os-server&search=(%3Fs)%3Cspan%5B%5E%3E%5D*%3E%5Cs*Total%5Cs%2Bdownloads%5Cs*%3C%2Fspan%3E.*%3F%3Ch3%5B%5E%3E%5D*%3E%5Cs*(%5B0-9%5D%5B0-9.%2C%5D*%5Cs*%5BKM%5D%3F)%5Cs*%3C%2Fh3%3E&replace=%241&logo=github&label=Downloads&cacheSeconds=3600"></a>
<a href="https://github.com/buggerman/unifi-os-server/actions/workflows/build-image.yaml"><img src="https://img.shields.io/github/actions/workflow/status/buggerman/unifi-os-server/build-image.yaml?logo=githubactions&logoColor=white&label=Actions"></a>

Run [UniFi OS Server](https://blog.ui.com/article/introducing-unifi-os-server) directly in Docker or Kubernetes.

> The **UniFi OS Server is the new standard for self-hosting UniFi**, replacing the legacy UniFi Network Server. While the Network Server provided basic hosting functionality, it lacked support for key UniFi OS features like Organizations, IdP Integration, or Site Magic SD-WAN. With a fully unified operating system, UniFi OS Server now delivers the same management experience as UniFi-native–including CloudKeys, Cloud Gateways, and Official UniFi Hosting–and is fully compatible with Site Manager for centralized, multi-site control.
>
> <https://help.ui.com/hc/en-us/articles/34210126298775-Self-Hosting-UniFi>

# Installation

## Docker Compose

See [docker-compose.yaml](https://github.com/buggerman/unifi-os-server/blob/main/docker-compose.yaml)

## TrueNAS SCALE

TrueNAS SCALE runs Docker natively. Two networking approaches are available depending on your preference.

### Option 1: Bridge with Port Mapping

The container shares the TrueNAS host IP and exposes each required port explicitly. Use the standard [docker-compose.yaml](https://github.com/buggerman/unifi-os-server/blob/main/docker-compose.yaml) as a starting point.

### Option 2: macvlan (recommended)

Assigns the container its own dedicated IP address on your network. No port mapping is required — the container is directly reachable at its IP on all ports, and behaves like a separate physical device.

Replace `enp131s0` with your TrueNAS host's physical network interface and adjust the subnet, gateway, and IP address to match your network.

```yaml
networks:
  unifi_network:
    driver: macvlan
    driver_opts:
      parent: enp131s0
    ipam:
      config:
        - subnet: 192.168.0.0/16
          gateway: 192.168.0.1

services:
  unifi-os-server:
    cap_add:
      - NET_RAW
      - NET_ADMIN
    cgroup: host
    container_name: unifi-os-server
    environment:
      - UOS_SYSTEM_IP=192.168.1.10
      - TZ=Europe/London
    image: ghcr.io/buggerman/unifi-os-server:latest
    networks:
      unifi_network:
        ipv4_address: 192.168.1.10
    restart: unless-stopped
    tmpfs:
      - /run:exec
      - /run/lock
      - /tmp:exec
      - /var/lib/journal
      - /var/opt/unifi/tmp:size=64m
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
      - /path/to/persistent:/persistent
      - /path/to/var-log:/var/log
      - /path/to/data:/data
      - /path/to/srv:/srv
      - /path/to/var-lib-unifi:/var/lib/unifi
      - /path/to/var-lib-mongodb:/var/lib/mongodb
      - /path/to/etc-rabbitmq-ssl:/etc/rabbitmq/ssl
```

> **Note:** With macvlan, the TrueNAS host cannot communicate with the container directly over the macvlan interface. This is a kernel-level limitation. If you need host-to-container access, add a macvlan shim on the host:
>
> ```bash
> ip link add macvlan0 link enp131s0 type macvlan mode bridge
> ip addr add 192.168.1.11/32 dev macvlan0
> ip link set macvlan0 up
> ip route add 192.168.1.10/32 dev macvlan0
> ```
>
> Use any unused IP address for the shim (`192.168.1.11` above). This does not persist across reboots — add it to a startup script if needed.

### ZFS Dataset

If you store container data on a ZFS dataset, verify it is not read-only before starting the container:

```bash
zfs get readonly <pool>/<dataset>
```

If the `VALUE` is `on`, disable it:

```bash
zfs set readonly=off <pool>/<dataset>
```

A `SOURCE` of `temporary` means the property was set in the current boot session and will clear on reboot, but the container will fail to start until it is explicitly cleared. This can happen after certain ZFS operations such as snapshot rollbacks.

## Kubernetes

See [kubernetes](https://github.com/buggerman/unifi-os-server/tree/main/kubernetes)

Deployment example uses [ingress-nginx](https://github.com/kubernetes/ingress-nginx) for the ingress and [longhorn](https://github.com/longhorn/longhorn) for storage.

Your ingress controller must be modified to accept extra ports. For example, `ingress-nginx` Helm values:

```yaml
tcp:
  5005: "unifi/unifi-os-server-rtp-svc:5005" # Optional
  9543: "unifi/unifi-os-server-id-hub-svc:9543" # Optional
  6789: "unifi/unifi-os-server-mobile-speedtest-svc:6789" # Optional
  8080: "unifi/unifi-os-server-communication-svc:8080"
  8443: "unifi/unifi-os-server-network-app-svc:8443" # Optional
  8444: "unifi/unifi-os-server-hotspot-secured-svc:8444" # Optional
  11084: "unifi/unifi-os-server-site-supervisor-svc:11084" # Optional
  5671: "unifi/unifi-os-server-aqmps-svc:5671" # Optional
  8880: "unifi/unifi-os-server-hotspot-redirect-0-svc:8880" # Optional
  8881: "unifi/unifi-os-server-hotspot-redirect-1-svc:8881" # Optional
  8882: "unifi/unifi-os-server-hotspot-redirect-2-svc:8882" # Optional
udp:
  3478: "unifi/unifi-os-server-stun-svc:3478"
  5514: "unifi/unifi-os-server-syslog-svc:5514" # Optional
  10003: "unifi/unifi-os-server-discovery-svc:10003"
```

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
| TCP | 8443 | Ingress | UniFi Network Application GUI/API |
| TCP | 8444 | Ingress | Secure Portal for Hotspot |
| UDP | 3478 | Both | STUN for device adoption and communication *(also required for Remote Management)* |
| UDP | 5514 | Ingress | Remote syslog capture |
| UDP | 10003 | Ingress | Device discovery during adoption |
| TCP | 11084 | Ingress | UniFi Site Supervisor |
| TCP | 5671 | Ingress | AQMPS |
| TCP | 8880 | Ingress | Hotspot portal redirection (HTTP) |
| TCP | 8881 | Ingress | Hotspot portal redirection (HTTP) |
| TCP | 8882 | Ingress | Hotspot portal redirection (HTTP) |

# Frequently Asked Questions

## What is the difference between images?

The `uosserver` image is provided by UniFi, extracted from the installation binary. The `unifi-os-server` image provides better compatibility for Docker and Kubernetes with directory fixes and configuration through environment variables.

## Why does the container need specific settings for cgroup and tmpfs?

The underlying structure of UniFi OS Server runs every component as systemd services which requires access to the host `cgroup`.
