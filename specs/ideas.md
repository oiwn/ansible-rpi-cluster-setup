# Ideas

## PoE HAT
- Tune fan curve: run CPU stress test, calibrate temp thresholds so fan stays off below 50°C
- Explore passive cooling with larger heatsink (drop PoE HAT fan entirely)
- Alternative: PoE splitter (PoE in → 5V USB-C out) — no HAT needed, full silence

## Storage
- Add NVMe HATs to workers, expand Garage into multi-node cluster
- S3 bucket backup to external location (rsync to NAS, or rclone to cloud)

## Networking
- PXE/netboot workers from gateway (drop SD cards entirely)
- mDNS setup for local name resolution without DNS server

## Monitoring
- Prometheus + Grafana stack via K3s
- Node Exporter on all hosts for CPU/temp/disk metrics
- Garage metrics endpoint

## Misc
- Tailscale for remote access to cluster
- Ansible playbook to wipe and re-provision a single worker
- Kubernetes Dashboard or k9s for cluster management
