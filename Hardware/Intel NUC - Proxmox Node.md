# Summary

The little NUC that could. `kepler` runs everything at the moment.

# Hardware

| Make    | Intel                                                   |
| ------- | ------------------------------------------------------- |
| Model   | NUC8i7BEH                                               |
| Memory  | 16 GB                                                   |
| Storage | 1 TB                                                    |
| CPU     | 8 x Intel(R) Core(TM) i7-8559U CPU @ 2.70GHz (1 Socket) |
| Kernel  | Linux 6.8.12-9-pve (2025-03-16T19:18Z)                  |

# Network

**Basic**

DNS is set to `sarlacc`, which is running pihole to perform network-wide adblocking. Due to DNS issues, I have set OpenDNS as the second server.  Will fix this, probably by moving pihole from proxmox. 

| DNS 1 | `sarlacc`        |
| ----- | ---------------- |
| DNS 2 | 208.67.222.220   |
| IP    | 10.0.0.2:8006/16 |
**Proxmox config**

`kepler` is configured with 2 VLANs currently, default and DMZ. Unfortunately this NUC only has one NIC, and as such the below setup is required to have both VLANs share the same physical port. 

| Name     | Type      | Autostart | VLAN aware | Ports/Slaves | CIDR         | Gateway  | VLAN Tag |
| -------- | --------- | --------- | ---------- | ------------ | ------------ | -------- | -------- |
| eno1     | Interface | No        | No         |              |              |          |          |
| vmbr0    | Bridge    | Yes       | Yes        | eno1         |              |          |          |
| vmbr0.1  | VLAN      | Yes       | No         |              | 10.0.0.2/16  | 10.0.0.1 | 1        |
| vmbr0.33 | VLAN      | Yes       | No         |              | 10.0.33.0/24 |          | 33       |

