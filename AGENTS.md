# Agent Notes

## Privacy rule (IMPORTANT)

NEVER commit cluster-external data to git: MAC addresses, home-LAN IPs
(192.168.1.x), WiFi SSIDs/passwords, hardware serials, public keys of
persons. Real values live ONLY in `inventory/hosts.yml`, which is
gitignored on purpose (see `inventory/hosts.default` for the template).
Playbooks must take such values from inventory hostvars, never hardcode.

## Ansible setup for this repo

- Inventory: `inventory/hosts.yml` (template: `inventory/hosts.default`).
  main = k3s-rspb5m (WiFi uplink, external IP only in hosts.yml).
  workers = k3s-rspb4b1 @ 10.0.0.2, k3s-rspb4b2 @ 10.0.0.3 (eth0-only,
  reached via ProxyJump through main, ansible_ssh_common_args set in inventory).
- Gateway done (playbook 01, 2026-08-22): eth0 10.0.0.1/24 (networkd),
  NetworkManager owns wlan0 only (resolv 127.0.0.1 since playbook 04),
  dnsmasq DHCP+DNS (pool .100-.150, fixed reservations by MAC from inventory,
  domain=cluster), NAT via wlan0, ip_forward=1, upstream DNS from inventory
  var `upstream_dns`. NOPASSWD sudo bootstrapped on all 3 nodes.
- Storage done (playbook 02, 2026-08-25): SSD GPT+ext4 label k3s-storage at
  /mnt/k3s-storage (nofail), subdirs k3s/ volumes/ backups/, memory cgroup
  enabled + fsck.repair on all nodes, journald capped 200M.
- k3s done (playbooks 04/05, 2026-08-25): server on main (--data-dir on SSD,
  --node-ip 10.0.0.1, tls-san cluster name + WiFi IP), agents on workers;
  v1.36.3+k3s1, 3× Ready. Headlamp (playbook 06) at NodePort 30080, token
  login (SA headlamp-admin, cluster-admin).
- Playbooks run in numbered order: 00 ping gate, 01 gateway, 02 storage,
  04/05 k3s, 06 headlamp. 99 is read-only diagnostics. 03 (ssh keys) was
  dropped: ProxyJump through main covers all access.

## One-time bootstrap on a fresh Pi (before Ansible works)

1. SSH key must be in the macOS keychain agent (key has a passphrase):
   ```
   ssh-add --apple-use-keychain ~/.ssh/id_rsa
   ```
2. Passwordless sudo on the Pi (Ansible `become: true` needs it; Imager
   default requires a password). Run once, enters Pi password
   (replace user/host with the real ones, do not commit real values):
   ```
   ssh -t <user>@<pi-ip> 'echo "<user> ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/010_<user>-nopasswd > /dev/null && sudo visudo -cf /etc/sudoers.d/010_<user>-nopasswd'
   ```

## Hardware quirks discovered (2026-08)

- Pi firmware injects `cgroup_disable=memory` AHEAD of cmdline.txt (never
  visible in the file, only in /proc/cmdline). Fix: append
  `cgroup_enable=memory cgroup_memory=1` to cmdline.txt — later tokens win.
  VERIFY with /sys/fs/cgroup/cgroup.controllers (must list memory) or a
  systemd-run MemoryMax test; /proc/cgroups NEVER shows memory on v2-only
  kernels and is the wrong check.
- Cluster case uses L-shaped SD card extender adapters. Oxidized contacts
  cause "Unable to read partition as FAT" boot failures. Fix: clean contacts
  with isopropyl. Cards verified good via `diskutil verifyVolume` on Mac.
- Pi WiFi (CYW43455, Pi4/Pi5) cannot use 5GHz DFS channels 52-144.
  The home router's 5GHz band sits on a DFS channel -> use its 2.4GHz
  band for the uplink.
