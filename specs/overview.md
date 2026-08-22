# Project Overview

## Hardware Layout

| Node | Model | Role | Storage |
|------|-------|------|---------|
| `raspb1s.local` | Raspberry Pi 5 8GB | Gateway + K3s server | 2TB SSD via USB 3.0 |
| `raspb2.local` | Raspberry Pi 4B 8GB | K3s worker | SD card |
| `raspb3.local` | Raspberry Pi 4B 8GB | K3s worker | SD card |

- 2× Official RPi PoE+ HAT (on workers)
- TL-SG1005P network switch with PoE
- All boards boot from SD cards

## Network Architecture

```
Internet (home router)
    |
    | WiFi
    |
Raspberry Pi 5 (raspb1s)
  wlan0: DHCP from home LAN
  eth0: 10.0.0.1/24 (static)
    |
    | Ethernet
    |
TL-SG1005P Switch (PoE)
    |
    ├── RPi 4B #1 (raspb2) 10.0.0.2
    └── RPi 4B #2 (raspb3) 10.0.0.3
```

- **Gateway (RPi 5)**: NAT + IP forwarding between wlan0 and eth0
- **DHCP/DNS**: dnsmasq on eth0, static leases by MAC for workers
- **Internal subnet**: `10.0.0.0/24`

## Storage Architecture

```
2TB SSD (/mnt/k3s-storage)
├── local-path/          ← K3s local-path-provisioner PVCs
└── garage/              ← Garage S3-compatible object store
```

- SSD formatted as ext4, mounted at `/mnt/k3s-storage`
- K3s local-path-provisioner for stateful workload PVCs
- Garage provides S3 API for object storage (AGPLv3, single-binary, ARM-native)

## Software Stack

| Layer | Component |
|-------|-----------|
| OS | Raspberry Pi OS (64-bit) |
| Container orchestration | K3s |
| Object storage (S3) | Garage |
| Network management | systemd-networkd + dnsmasq |
| Provisioning | Ansible |

## Playbook Flow

```
01_prepare_gateway.yml    — Network setup, DHCP, NAT, IP forwarding
02_prepare_storage.yml    — SSD partition, format, mount
03_ssh_access.yml         — SSH key generation for cluster
04_install_k3s.yml        — K3s server on gateway + agents on workers
05_install_garage.yml     — Garage S3 on gateway
```

Playbooks run sequentially on fresh RPi OS installations.
