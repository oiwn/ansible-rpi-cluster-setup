# Architecture Overview

Deep-dive on how the cluster is put together. For setup instructions see the
[README](../README.md).

## Nodes

| Inventory group | Model | Role | Storage |
|-----------------|-------|------|---------|
| `main` | Raspberry Pi 5 8GB | Gateway, DHCP/DNS, NAT, k3s server | 1TB USB 3.0 SSD + SD boot |
| `workers` | 2× Raspberry Pi 4B 8GB | k3s agents | SD card only |

Hostnames are yours to choose — they live in the gitignored
`inventory/hosts.yml` and flow into the playbooks as hostvars (the k3s
`--tls-san` is derived from `inventory_hostname`, DHCP reservations from
`mac_address`, and so on). All three nodes boot from SD; the SSD carries cluster
state only, so a dead SSD costs data but not bootability.

Workers are powered over PoE from the switch, so each takes a single cable.

## Network

```
Internet (home router)
    │
    │ WiFi — 2.4GHz (the Pi radio cannot use 5GHz DFS channels)
    │
Raspberry Pi 5  [group: main]
  wlan0 : DHCP from the home LAN, managed by NetworkManager
  eth0  : 10.0.0.1/24 static, managed by systemd-networkd
    │
    │ Ethernet
    │
TL-SG1005P switch (PoE)
    │
    ├── Pi 4B #1  10.0.0.2
    └── Pi 4B #2  10.0.0.3
```

- **Internal subnet:** `10.0.0.0/24`, domain `cluster`.
- **Routing:** `net.ipv4.ip_forward=1` plus an iptables `MASQUERADE` on wlan0,
  persisted through a small `iptables-restore` systemd unit.
- **Interface ownership is split on purpose.** systemd-networkd holds eth0;
  NetworkManager is told `unmanaged-devices=interface-name:eth0` so it keeps to
  wlan0. An earlier revision let both touch eth0 and they fought over it.
- **DHCP/DNS:** a single `dnsmasq` on eth0 — dynamic pool `.100–.150` so an
  unknown board can bootstrap, plus fixed reservations by MAC (generated from
  inventory) that pin the workers. networkd's own DHCP server is explicitly
  unused; running both produced a double-DHCP bug.
- **Resolution:** dnsmasq serves `*.cluster` and forwards everything else to
  `upstream_dns` from the inventory. The gateway points its own resolver at
  `127.0.0.1`, so it resolves cluster names too.

## Storage

```
1TB SSD  →  /mnt/k3s-storage   (ext4, label k3s-storage, nofail + device-timeout)
├── k3s/       k3s server state — sqlite datastore, certs   (--data-dir)
├── volumes/   local-path PVCs, the default StorageClass
└── backups/   app-level backups
```

`local-path-provisioner` is the default StorageClass, rooted on the SSD. Stateful
workloads pin to the gateway with `nodeSelector: storage=ssd`.

Longhorn and an object store (Garage) were both evaluated and dropped: with one
disk there is nothing to replicate to, and the cluster is explicitly not meant
to be fault tolerant. See [ideas.md](ideas.md) for what that would take.

**SD card wear** is managed by keeping persistent state off the workers
entirely and capping journald at 200M on every node.

### Boot flags

Applied to all three nodes by `02_storage.yml`:

- `cgroup_enable=memory cgroup_memory=1` — required for kubelet memory limits,
  and appended rather than edited in place because the Pi firmware injects
  `cgroup_disable=memory` ahead of `cmdline.txt`. Later tokens win.
- `fsck.repair=yes` — unattended recovery after an unclean shutdown.

## Node labels

| Label | Where | Purpose |
|-------|-------|---------|
| `storage=ssd` | gateway | Target for stateful workloads |
| `node-type=pi5` | gateway | Scheduling by board generation |
| `node-type=pi4` | workers | Scheduling by board generation |

## Software stack

| Layer | Component |
|-------|-----------|
| OS | Raspberry Pi OS 64-bit |
| Orchestration | k3s (v1.36.3+k3s1 at time of build) |
| Storage class | k3s local-path-provisioner |
| Link config | systemd-networkd (eth0) + NetworkManager (wlan0) |
| DHCP/DNS | dnsmasq |
| Web UI | Headlamp, NodePort 30080 |
| Provisioning | Ansible |

## Playbook flow

```
00_ping.yml             connectivity gate
01_gateway_network.yml  eth0 static, dnsmasq DHCP+DNS, NAT, ip_forward
02_storage.yml          SSD partition/format/mount, boot flags, journald caps
04_k3s_server.yml       k3s server on the gateway, state on the SSD
05_k3s_workers.yml      k3s agents join via the gateway API
06_headlamp.yml         Headlamp web panel
99_diagnose.yml         read-only diagnostics
```

Each runs against a fresh Raspberry Pi OS install, in order, and ends with a
verification block. `03` was an SSH-key playbook, dropped in favour of ProxyJump
through the gateway.

## Access model

There are no cluster-internal SSH keys. Everything reaches the workers by
jumping through the gateway (`ansible_ssh_common_args: -J …` in the inventory),
which keeps key material on the workstation and the workers unreachable from the
home LAN. kubectl works on the gateway directly, from a laptop via a copied
kubeconfig, or through Headlamp in a browser.
