I was originally running this on Proxmox, but moved to a dedicated Raspberry Pi 4. If I intend to run updates on the Proxmox node, it would make sense to do so from outside of it. 

# Hardware

https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/

| Make    | Raspberry Pi                                                        |
| ------- | ------------------------------------------------------------------- |
| Model   | 4B                                                                  |
| Memory  | 4 GB                                                                |
| Storage | 128 GB                                                              |
| CPU     | Broadcom BCM2711, Quad core Cortex-A72 (ARM v8) 64-bit SoC @ 1.8GHz |
| Kernel  | Debian Linux 6.12 for Raspberry Pi 2712                             |
# Network

Nothing too exciting here. This host sits on the main network, and is tagged for VLAN 33 to allow for management of the DMZ VLAN. 

I intend to use this pi for VPN tunnel endpoints as well, but need to play with Docker VLAN config to get it working. 