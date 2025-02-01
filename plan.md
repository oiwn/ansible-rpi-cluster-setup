# Raspberry Pi Cluster Configuration Plan

## Hardware Layout
- Gateway Node (raspb1s): Raspberry Pi 5
  - 1TB SSD via USB 3.0
  - Dual network interfaces
  - Hosts databases and storage
- Worker Nodes: 2x Raspberry Pi 4B
  - Network connection via ethernet
  - Focus on running applications

## Initial Setup
### Gateway Node (raspb1s)
- Flash using Raspberry Pi Imager
  - Enable SSH with public key authentication
  - Configure WiFi credentials
  - Set hostname to raspb1s

### Worker Nodes
- Flash using Raspberry Pi Imager
  - Enable SSH with password auth initially
  - Set hostnames (raspb2, raspb3)
  - No WiFi configuration needed

## Network Architecture
### Gateway Configuration
- wlan0 (WiFi)
  - Internet connection via home router
  - DHCP from home network
- eth0 (Ethernet)
  - Static IP: 10.0.0.1
  - Gateway for worker nodes
  - Connected to network switch

### Worker Configuration
- eth0 only
  - IP from DHCP (10.0.0.2, 10.0.0.3)
  - Gateway points to raspb1s (10.0.0.1)
  - Internet access through gateway

## Network Management
### Core Components
- systemd-networkd
  - Interface configuration
  - Network routing
- dnsmasq
  - DHCP server
  - DNS forwarding
  - Local name resolution
  - Static IP assignments by MAC

### Network Features
- IP Forwarding enabled on gateway
- NAT for internet access
- Internal DNS resolution
- Fixed IP assignments for workers

## Storage Architecture
### Primary Storage (Longhorn)
- 1TB SSD on gateway node
- Managed by Longhorn for:
  - Block storage provisioning
  - Volume management
  - Future storage expansion
  - Backup capabilities

### Storage Strategy
- Minimize SD card I/O
- SSD for all persistent storage
- Longhorn manages storage pool
- Prepared for future NVMe expansion

## Workload Distribution
### Gateway Node (raspb1s)
- Database services
  - Direct access to SSD storage
  - Exposed via k3s services
- Network services (DHCP, DNS)
- Storage management (Longhorn)

### Worker Nodes
- Application workloads
- Connect to databases on gateway
- Stateless applications preferred
- K3s handles service discovery

## SSH Access Strategy
- Gateway Node (raspb1s)
  - Public key authentication only
  - Your personal key access
- Worker Nodes
  - Initially password auth
  - Switch to key-based auth
  - Keys managed from gateway

## Core Software Stack
- Operating System: Raspberry Pi OS
- Container Orchestration: K3s
- Storage: Longhorn
- Network: systemd-networkd + dnsmasq
- Database Hosting: Direct on raspb1s

## Future Expansion Path
- Add NVMe storage to workers
- Expand Longhorn storage pool
- Scale application workloads
- Maintain database centralization


