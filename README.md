# ansible-rpi-cluster-setup

Ansible playbooks that build a 3-node Raspberry Pi k3s cluster from freshly
flashed SD cards: **1× Pi 5 8GB** acting as WiFi-to-Ethernet gateway, DHCP/DNS
server and k3s control plane (with a USB SSD for cluster state), plus **2× Pi 4B
8GB** PoE workers on an isolated `10.0.0.0/24` segment behind it.

This is a real, running home lab — every playbook here was used to build the
cluster it describes, including a deliberate plug-pull acceptance test. It is
**not** a production-grade setup: there is a single server node and a single
disk, and no fault tolerance, on purpose (see [Design choices](#design-choices)).

## Hardware

| Qty | Item | Role |
|-----|------|------|
| 1 | Raspberry Pi 5, 8GB | Gateway + k3s server |
| 2 | Raspberry Pi 4B, 8GB | k3s workers |
| 2 | Official Raspberry Pi PoE+ HAT | Powers the workers off the switch |
| 1 | TL-SG1005P PoE switch | Cluster LAN |
| 1 | 1TB Samsung USB SSD | k3s state + PVCs, on the Pi 5 |
| 3 | SD card | Boot media (all three nodes boot from SD) |

The workers take power *and* network from the switch — one cable each.

## Network

```
Internet ── home router
                │  WiFi
                │
       ┌────────┴──────────┐
       │  Pi 5  (gateway)  │  wlan0: DHCP from home LAN, NAT + ip_forward
       │                   │  eth0:  10.0.0.1/24 static, dnsmasq DHCP+DNS
       └────────┬──────────┘
                │  Ethernet
         TL-SG1005P switch (PoE)
                │
        ┌───────┴───────┐
   Pi 4B #1         Pi 4B #2
   10.0.0.2         10.0.0.3
```

The workers have no route to your home LAN and are not reachable from it
directly — everything reaches them by jumping through the gateway. `dnsmasq`
serves the `.cluster` domain, with fixed DHCP reservations pinning each worker
to its address by MAC.

## Prerequisites

- **Ansible core ≥ 2.21** on your workstation (developed on 2.21.1).
- Collections: `ansible-galaxy collection install -r requirements.yml`
  (`community.general` for `parted`/`filesystem`, `ansible.posix` for `mount`).
- Raspberry Pi OS 64-bit on all three nodes.
- An SSH key loaded in your agent. On macOS, if the key has a passphrase:
  `ssh-add --apple-use-keychain ~/.ssh/id_rsa`

## Quickstart

**1. Flash each Pi** with Raspberry Pi Imager: enable SSH with your public key,
set the hostname and username.

**2. Bootstrap passwordless sudo on each Pi.** Ansible's `become: true` needs
it, and the Imager default requires a password. Run once per node — this prompts
for the Pi's password:

```sh
ssh -t <user>@<pi-ip> 'echo "<user> ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/010_<user>-nopasswd > /dev/null && sudo visudo -cf /etc/sudoers.d/010_<user>-nopasswd'
```

Workers aren't on the cluster LAN yet at this point — put them on your home LAN
temporarily, or do this step over a directly attached keyboard.

**3. Fill in your inventory:**

```sh
cp inventory/hosts.example.yml inventory/hosts.yml
$EDITOR inventory/hosts.yml
```

`inventory/hosts.yml` is **gitignored** and holds every real value — MAC
addresses, your home-LAN addresses, your username. The template documents what
each variable is consumed by. Playbooks read these from inventory hostvars and
never hardcode them.

**4. Run the playbooks in order.** Each one ends with a verification block that
prints what it actually did, and each assumes the previous one succeeded:

```sh
ansible-playbook playbooks/00_ping.yml
ansible-playbook playbooks/01_gateway_network.yml
ansible-playbook playbooks/02_storage.yml
ansible-playbook playbooks/04_k3s_server.yml
ansible-playbook playbooks/05_k3s_workers.yml
ansible-playbook playbooks/06_headlamp.yml
```

| Playbook | What it does |
|----------|--------------|
| `00_ping.yml` | Connectivity gate; prints OS/kernel/interfaces |
| `01_gateway_network.yml` | eth0 `10.0.0.1/24` via systemd-networkd, dnsmasq DHCP+DNS (`*.cluster`), NAT out wlan0, `ip_forward` |
| `02_storage.yml` | SSD GPT+ext4 → `/mnt/k3s-storage`; cgroup/fsck boot flags and journald cap on **all** nodes (reboots) |
| `04_k3s_server.yml` | k3s server on the gateway, state on the SSD (reboots) |
| `05_k3s_workers.yml` | k3s agents on the workers, joining via the gateway API |
| `06_headlamp.yml` | Headlamp web UI on NodePort 30080 |
| `99_diagnose.yml` | Read-only diagnostics — safe to run any time |

After 01, the workers move to the switch and pick up their reserved addresses.
02 and 04 reboot nodes; that's intentional (the boot flags need it, and it
proves the setup survives a restart).

> **There is no `03`.** It was an SSH-key-distribution playbook, dropped once
> ProxyJump through the gateway turned out to cover every access path. The gap
> in the numbering is deliberate.

## Accessing the cluster

| Route | How | Setup |
|-------|-----|-------|
| kubectl on the gateway | `ssh <user>@<gateway> kubectl get nodes` | none |
| Headlamp in a browser | `http://<gateway-home-lan-ip>:30080`, choose **Token** login (playbook 06 prints one; reissue with `kubectl -n headlamp create token headlamp-admin --duration=87600h`) | none |
| kubectl from your laptop | `scp <user>@<gateway>:.kube/config ~/.kube/config` — the server endpoint is the gateway's home-LAN IP, already covered by a `tls-san` | one file |
| SSH to a worker | `ssh -J <user>@<gateway> <user>@10.0.0.2` | already baked into the inventory |

## Storage

```
/mnt/k3s-storage/k3s      k3s server state (sqlite, certs)  — --data-dir
/mnt/k3s-storage/volumes  local-path PVCs (default StorageClass)
/mnt/k3s-storage/backups  app-level backups
```

Mounted `nofail` with a device timeout, so a missing SSD never hangs boot.
Workers keep only the agent and container images on SD, with journald capped at
200M to limit write wear. Node labels: `storage=ssd` on the gateway,
`node-type=pi5|pi4` everywhere — pin stateful workloads to the SSD with
`nodeSelector: storage=ssd`.

## Design choices

- **No fault tolerance, deliberately.** One server node, one disk. Adding a
  second control-plane node or replicated storage to three Pis buys complexity,
  not resilience, at this scale.
- **local-path over Longhorn.** Single disk means replication has nowhere to go,
  and cross-node data access here is object-shaped rather than block-shaped.
- **dnsmasq owns DHCP, networkd owns the link.** systemd-networkd's built-in
  DHCP server is explicitly *not* used — running both caused a double-DHCP
  fight. Likewise NetworkManager is confined to wlan0 so it stops racing
  networkd for eth0.
- **Workers are compute-only.** All persistent state lives on the gateway's SSD.

## Power-outage behaviour

Designed for unattended recovery and tested by pulling the plug on the whole
cluster: ext4 journaling, `fsck.repair=yes`, a `nofail` mount, and every service
enabled at boot. Expect DHCP and WiFi back in ~1 min, the k3s server in ~2, and
workers rejoining by ~3. Verify with `kubectl get nodes` (3× Ready) and
`kubectl get pods -A`. If a node stays NotReady, `sudo systemctl restart k3s`
(or `k3s-agent` on a worker).

## Hardware quirks worth knowing

Three things cost real debugging time here; they may save you some.

- **Pi firmware injects `cgroup_disable=memory` ahead of `cmdline.txt`.** You
  will never see it in the file — only in `/proc/cmdline` — and it silently
  breaks k8s memory limits. The fix is to *append* `cgroup_enable=memory
  cgroup_memory=1` to `cmdline.txt`, since later tokens win. Verify with
  `/sys/fs/cgroup/cgroup.controllers` (must list `memory`) or an actual
  `systemd-run -p MemoryMax=…` test. **Do not check `/proc/cgroups`** — on
  v2-only kernels it never shows memory and will mislead you.
- **L-shaped SD card extender adapters oxidize.** Common in stacked cluster
  cases. Symptom is a boot failure reading `Unable to read partition as FAT` on
  a card that verifies as perfectly healthy elsewhere. Clean the contacts with
  isopropyl alcohol.
- **The Pi WiFi chip (CYW43455, Pi 4 and Pi 5) cannot use 5GHz DFS channels
  52–144.** If your router's 5GHz band sits on a DFS channel, the Pi simply will
  not see it — use the 2.4GHz band for the uplink.

## Security notes

This is a home lab on a trusted LAN, and the defaults reflect that. Before you
copy any of it somewhere less friendly:

- **Headlamp is served over plain HTTP** on NodePort 30080, reachable from your
  home LAN, and playbook 06 **prints a 10-year `cluster-admin` token to your
  terminal**. Anyone with that token owns the cluster. Shorten `--duration`, put
  it behind TLS, or don't expose the NodePort if that's not acceptable.
- Playbook 05 passes the k3s join token on a shell command line, so it is
  briefly visible in `ps` on the worker.
- **`99_diagnose.yml` output contains hardware serials and DHCP leases with MAC
  addresses.** Redact it before pasting into an issue or a gist.
- Passwordless sudo is enabled on every node as a prerequisite.

## Repo layout

```
playbooks/              numbered, run in order; 99 is read-only
inventory/
  hosts.example.yml     template — copy to hosts.yml (gitignored)
requirements.yml        Ansible collections
specs/overview.md       architecture deep-dive
specs/ideas.md          roadmap / things not built yet
CHANGELOG.md            what was built when, and what broke along the way
AGENTS.md               contract for AI coding agents working in this repo
```

## License

MIT — see [LICENSE](LICENSE).
