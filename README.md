# ansible-rpi-cluster-setup

Ansible playbooks for a 3-node Raspberry Pi k3s cluster:
1× RPi 5 8GB (gateway, WiFi uplink, SSD) + 2× RPi 4B 8GB (PoE workers).

Real IPs/MACs live **only** in `inventory/hosts.yml` (gitignored — copy from
`inventory/hosts.default`). See `AGENTS.md` for the privacy rule.

## One-time bootstrap per Pi (before Ansible works)

1. Flash with Raspberry Pi Imager: enable SSH (public key), set hostname/user.
2. Mac: `ssh-add --apple-use-keychain ~/.ssh/id_rsa` (key has a passphrase).
3. On each Pi (enters Pi password once):
   ```
   ssh -t <user>@<pi-ip> 'echo "<user> ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/010_<user>-nopasswd > /dev/null && sudo visudo -cf /etc/sudoers.d/010_<user>-nopasswd'
   ```

## Playbooks (run in order, each gated on the previous)

```
00_ping.yml          connectivity gate
01_gateway_network.yml  eth0 10.0.0.1/24, dnsmasq DHCP+DNS (*.cluster), NAT
02_storage.yml       SSD GPT+ext4 -> /mnt/k3s-storage, cmdline/journald fixes
04_k3s_server.yml    k3s server on gateway (state on SSD)
05_k3s_workers.yml   k3s agents on workers
06_headlamp.yml      Headlamp web panel (NodePort 30080)
99_diagnose.yml      read-only cluster diagnostics
```

Run: `ansible-playbook playbooks/NN_name.yml` from the repo root.

## Access tiers

| Tier | How | Setup |
|------|-----|-------|
| kubectl on gateway | `ssh <user>@<gateway> kubectl get nodes` | none |
| Headlamp in browser | `http://<gateway-wifi-ip>:30080`, login = token (printed by playbook 06; reissue: `kubectl -n headlamp create token headlamp-admin --duration=87600h`) | none |
| kubectl from Mac | `scp <user>@<gateway>:.kube/config ~/.kube/config` (server endpoint = gateway WiFi IP, already tls-san'd) | one file |

Worker SSH: `ssh -J <user>@<gateway> <user>@10.0.0.2` (ProxyJump; also baked
into `inventory/hosts.yml`).

## Storage layout (SSD on gateway)

```
/mnt/k3s-storage/k3s      k3s server state (etcd/sqlite, certs)
/mnt/k3s-storage/volumes  local-path PVCs (default StorageClass)
/mnt/k3s-storage/backups  app-level backups
```

Workers run agent + images on SD (cgroup memory enabled on all nodes;
journald capped at 200M). Node labels: `storage=ssd` (gateway),
`node-type=pi5|pi4`.

## Power-outage runbook

Design (no fault tolerance, tested by pulling the plug on purpose once):
ext4 journaling + `fsck.repair=yes` + `nofail` mount + all services enabled.
After power returns, expect: DHCP/wifi ~1min, k3s server ~2min, workers
joining ~3min. Verify: `kubectl get nodes` → 3× Ready, `kubectl get pods -A`.
If a node is NotReady: `sudo systemctl restart k3s` (or k3s-agent on workers).
