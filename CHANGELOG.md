# Changelog

## 2026-08-25 — Base cluster infra: SSD storage, k3s, Headlamp

- `02_storage.yml` (replaces old 02): SSD GPT+ext4 (label `k3s-storage`)
  mounted at `/mnt/k3s-storage` with `nofail` + device-timeout; subdirs
  `k3s/ volumes/ backups/`; `cgroup_enable=memory cgroup_memory=1` +
  `fsck.repair=yes` on all nodes; journald capped 200M.
  Gotcha found: Pi firmware injects `cgroup_disable=memory` AHEAD of
  cmdline.txt — appended flags win; verify via cgroup.controllers (NOT
  /proc/cgroups, which never shows memory on v2-only kernels). Memory limits
  verified enforced end-to-end (MemoryMax test) on all 3 nodes.
- `04_k3s_server.yml` (replaces old 04): k3s server on gateway — data-dir
  on SSD, `--node-ip=10.0.0.1` (fixes old `ansible_default_ipv4`-picks-wlan0
  bug), tls-san cluster name + WiFi IP, local-path default StorageClass
  rooted on SSD, node labels `storage=ssd`/`node-type=pi5|pi4`; gateway
  resolver → 127.0.0.1 (gateway now resolves `*.cluster`). v1.36.3+k3s1.
- `05_k3s_workers.yml` (replaces old 05): agents at 10.0.0.2/.3 join via
  gateway API; 3× Ready, system pods green.
- `06_headlamp.yml`: Headlamp web panel at NodePort 30080, token login
  (`--incluster` flag was removed upstream; token-login mode used instead).
  Verified reachable from home LAN.
- Housekeeping: removed stale `02/03/04/05` playbooks, longhorn docs,
  orphaned dhcp-era templates, empty group_vars, empty `kuber/`;
  `99_diagnose.yml` extended with k3s/storage sections; README rewritten
  (bootstrap, playbook order, access tiers, storage layout, power-outage
  runbook); AGENTS.md state refreshed.
- Plug-pull acceptance test PASSED: full-cluster power outage → unattended
  recovery to 3× Ready, 8 pods Running, Headlamp up; SSD clean, reserved
  DHCP leases re-acquired, no k3s journal corruption.
- Storage decisions: k3s local-path (not Longhorn — single disk, no fault
  tolerance wanted, cross-node data access is object-shaped); stateful
  workloads pin to gateway via `nodeSelector: storage=ssd`; workers are
  compute-only (agent/images on SD, capped logs).

## 2026-08-22 — Gateway network (playbook 01)

- `01_gateway_network.yml`: eth0 10.0.0.1/24 via systemd-networkd;
  NetworkManager owns wlan0 only (explicit unmanaged-eth0 config — fixes
  old playbook's NM/networkd fight); dnsmasq DHCP+DNS (dynamic pool
  .100-.150 + fixed MAC reservations from inventory, domain=cluster; no
  built-in networkd DHCP server — fixes old double-DHCP bug); NAT via
  wlan0; ip_forward=1; upstream DNS from inventory var `upstream_dns`.
- Workers discovered via dnsmasq log-dhcp, pinned to reserved 10.0.0.2/.3.
- Access model: SSH ProxyJump through gateway (cluster-internal SSH keys
  dropped as unnecessary); NOPASSWD sudo bootstrapped on all nodes.
- Privacy rule established: MACs, serials, home-LAN IPs, WiFi SSIDs live
  only in gitignored `inventory/hosts.yml`; playbooks take values from
  hostvars.
