# **VLANs**

Setup 3 VLAN's, IP ranges match numbers for convenience of remembering. Likely going to make VLAN 23 for trusted devices, but default works fine for now:

| ID  | Name            | IP Range     |
| --- | --------------- | ------------ |
| 1   | Default         | 10.0.0.0/24  |
| 13  | Guestnet        | 10.0.13.0/24 |
| 33  | DMZ - Media Net | 10.0.33.0/24 |
## **Port tagging**

| Host                 | VLANs     | Notes                                                          |
| -------------------- | --------- | -------------------------------------------------------------- |
| Gateway              | 33, 13, 1 | Its the gateway                                                |
| `kepler` (proxmox)   | 33, 1     | Host requires DMZ access due to hosting containers in the DMZ  |
| Main Desktop         | 33, 13, 1 | Main desktop VLANs are triggered when needed for mgmt.         |
| `warpgate` (ansible) | 33, 1     | Ansible host has specific firewall rules setup for DMZ access. |
| `mustafar` (NAS)     | 1         | NAS does not directly connect to the DMZ.                      |
| IOT                  | 13        | IOT devices are assigned to the guest network for separation.  |

# **DNS**

Forwarding to `sarlacc` pihole server for DNS. May switch this out for adguard home, also need to troubleshoot as it doesn't seem to be working 100%. 
# **DMZ**

VLAN 33 is the DMZ net. This encompasses 10.0.33.0/24 and contains the media services (currently just streaming). These services will be accessed externally and shared with friends, so placing them in the DMZ made the most sense. 

## **Why a DMZ?** 

Having a DMZ will make me feel more warm and fuzzy about opening up a server to the internet with no auth (although plex should be secure enough.) for more info on the plex deployment, see 

Unifi has had an ACL implemented with an implicit deny between VLAN 33 and any other local network. There were rules put in place to allow NFS ports between `mustafar` and `fhloston` / `jumbotron`. Tested this and seems to be working. 

| Name               | Action | Type      | Protocol | Source     | Port | Destination           | Port         |
| ------------------ | ------ | --------- | -------- | ---------- | ---- | --------------------- | ------------ |
| NFS Plex           | Accept | LAN Local | TCP/UDP  | 10.0.33.33 | -    | 10.0.0.23             | 111,892,2049 |
| DMZ implicit block | Block  | Local     | All      | DMZ (33)   | Any  | Default(1), guest(13) | Any          |

Also placed in a rule to allow my desktop to reach anything on the DMZ network as the main management device. Will disable this rule when management is not needed: 

| Name         | Action | Type  | Protocol | Source          | Port | Destination | Port |
|--------------|--------|-------|----------|-----------------|------|-------------|------|
| Desktop MGMT | Allow  | Local | All      | DESKTOP-RDH12UV | Any  | DMZ         | Any  

Also placed a rule similar to the above for ansible host `fleet-controller`

| Name          | Action | Type       | Protocol | Source          | Port | Destination | Port |
| ------------- | ------ | ---------- | -------- | --------------- | ---- | ----------- | ---- |
| fleet-ansible | Allow  | IP Address | All      | fleet-commander | Any  | 10.0.33.33  | 22   |
| *             | *      | *          | *        | *               | *    | 10.0.33.199 | 22   |
