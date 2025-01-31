Role Name
=========

Main Rpi board has 2 boot option 1) from SD card 2) from SSD
First option used as backup, if something happened it's always possible to boot
from SD card as fallback  option.

This playbook implement the following:

  - Update rpi1 and install required software
  - Assign static IP address
  - Bridge traffic from wifi to LAN port

Requirements
------------


Role Variables
--------------


Dependencies
------------


Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: raspb1.local
      roles:
         - { role: prepare_rpi1 }

License
-------

BSD

Author Information
------------------

