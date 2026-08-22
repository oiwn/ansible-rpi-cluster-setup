# Current Task Context

## State

Project revival. Existing playbooks (01–05) need cleanup and completion. Garage replaces Longhorn as the S3 object store. Playbooks were built for an earlier hardware revision — need adaptation for current setup.

## Current Hardware

- 1× RPi 5 8GB (gateway, WiFi, 2TB SSD)
- 2× RPi 4B 8GB + official PoE+ HAT (workers, PoE powered)
- TL-SG1005P switch

## Tasks

### 1. Hardware provisioning
- [ ] Flash RPi 5 with Raspberry Pi OS (64-bit), enable SSH, configure WiFi, set hostname `raspb1s`
- [ ] Flash both RPi 4Bs with Raspberry Pi OS (64-bit), enable SSH (password auth), set hostnames `raspb2`, `raspb3`
- [ ] Connect hardware: RPi 5 eth0 → switch, workers → switch PoE ports

### 2. Inventory
- [ ] Copy `inventory/hosts.default` → `inventory/hosts.yml`
- [ ] Fill real MAC addresses, IPs, ansible_user
- [ ] Verify SSH connectivity to all hosts before running playbooks

### 3. Playbook fixes
- [ ] Complete truncated `04_install_k3s.yml` (worker section missing after line 76)
- [ ] Resolve 04/05 overlap (decide merge or keep both)
- [ ] Strip all Longhorn references and files (`kuber/longhorn_ingress.yml`, `longhorn_setup.md`)
- [ ] Wire up or delete unused templates in `templates/`
- [ ] Fill empty group vars (`all.yml`, `gateway.yml`, `workers.yml`) with defaults

### 4. New playbook
- [ ] Create `playbooks/05_install_garage.yml` — deploy Garage single-node on gateway
  - Data directory: `/mnt/k3s-storage/garage`
  - Expose S3 API endpoint via K3s service
  - Default bucket and access key generation

### 5. Documentation
- [ ] Update `plan.md` or merge into `specs/overview.md`
- [ ] Write `README.md` with:
  - Hardware list
  - Quick-start guide (flash SDs → hosts.yml → run playbooks)
  - PoE HAT noise section (investigate `dtoverlay=rpi-poe` PWM fan control)

### 6. PoE HAT noise
- [ ] Investigate official RPi PoE+ HAT fan noise
  - Check if fan spins at boot (likely always-on by default)
  - Configure PWM control via `dtoverlay=rpi-poe` in `/boot/firmware/config.txt`
  - `dtparam=poe_fan_temp0=45000,poe_fan_temp0_hyst=5000` etc.
  - Target: silent at idle, gentle ramp under load

## Verification

After each playbook run:
1. Gateway: `ip addr show eth0` should show `10.0.0.1/24`, `sysctl net.ipv4.ip_forward` = 1
2. Workers: should receive DHCP lease from gateway, have internet access via NAT
3. K3s: `kubectl get nodes` on gateway shows all 3 nodes Ready
4. Garage: S3 API reachable at `http://<gateway>:3900`

## Active Blockers

None yet — hardware not yet flashed.
