# ansible-rpi-cluster-setup

Guides and ansible scripts to setup and manage toy raspberry pi cluster.

### Hardware:
   - 1x Raspberry Pi 5 (will act as gateway and main node)
   - 2x Raspberry Pi 4B (worker nodes)
   - 2x Raspberry Pi PoE+ HAT (extension of Rpi4B board to enable PoE)
   - 1x TL-SG1005P network switch with PoE (for 2x Pi 4B)
   - 1Tb Samsing SSD connected t0 Raspberry Pi 5 USB (main storage)
   - Cluster case ()
   - 3x SD Cards min 64Gb (or NVME)

PoE = Power On Ethernet.



Cluster configuration:

+ Raspberry Pi 5 8Gb with SATA SSD 1Tb connected to USB3 port
+ 2 x Raspberry Pi 4B 8Gb
+ All boards booted from SD cards


NEED TO DO:

+ split into the smaller testable parts 

Main:

+ generate locale

