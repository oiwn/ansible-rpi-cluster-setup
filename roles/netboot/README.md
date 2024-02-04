Role Name
=========

Setting up Neetboot for raspberriy pies cluster.

Requirements
------------

Cluster of raspberry Pies with following configuration:

  - Raspberry Pi 5 with 1TB SATA SSD connected to USB3 port
  - 2 x Raspberry Pi 4B


Assuming the Raspberry Pi OS (Bookworm) used.

Prepare bootable SD Card for raspberry pi 5, with desktop invironment and 
hostname "raspb1.local", it will be fallback bootable device (if you will 
want to reset the cluster).

Prepare Raspberry Pi 4B image by your own: use Imager app to flash SD card with
hostname "raspb-img4b.local".

Use VNC client to connect to Raspberry Pi 5 desktop, and use Imager to flash
SSD disk with Raspberry Pi imager with hostname "raspb1s.local"

Role Variables
--------------


Dependencies
------------


Example Playbook
----------------

    - hosts: raspb1s.local
      roles:
         - { role: netboot  }

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
