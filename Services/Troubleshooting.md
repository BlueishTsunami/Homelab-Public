
# NAS NFS connection

Lots that can go wrong here, documenting some of the struggles. Note for most of this, my issues were due to the DMZ.
## "No route to host"

Resolution: DNS configuration was botched.

Getting this error while trying to mount the host. Double, triple, quadruple checked the following:

- Synology NAS permissions setup correctly per [[Synology NAS#NFS Settings]]
- Plex host has all the prerequisites per [[Host Setup#NFS Connection]]
- Checked all my firewall rules were setup properly per [[Network#**DMZ**]]

Went around in circles trying to figure this out, and then realized I was dumb. The proxmox host was using the host DNS settings, which pointed to my pihole in the default VLAN. Instead of mucking with the firewall, I opted to point the server to OpenDNS

```sh
sudo nano /etc/network/interfaces

# Update your interface with nameservers
iface eth0 inet static
    address 10.0.1.10
    netmask 255.255.255.0
    gateway 10.0.1.1
    dns-nameservers 208.67.222.222 208.67.220.220

# Check override file: 
sudo nano /etc/resolv.conf

# Input / modify these lines
nameserver 208.67.220.222
nameserver 208.67.220.220

# Restart networking service
sudo ifdown ens18
sudo restart networking

# Test
dig google.com
nslookup google.com
```

## Connections timing out

"mount.nfs: portmap query retrying: RPC: Timed Out"

Getting this error continuously when setting up NFS. Tried to do some things in `fstab`

```sh
# nofail = Dont stop boot process if mount fails
# _netdev = Waits for network to be up before mount
<nas_ip>:/volume1/Data /mnt/media nfs defaults,nofail,_netdev 0 0
```

Tried to force different versions, in case it was an issue with versioning. 

***
# Ansible / Keychain errors

one of the warnings I got while doing this: 

```sh
#This warning will show up if you dont 'chmod 600' the file
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for '/home/blu/.ssh/ansiblekey' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
```

Another warning i got later, fixed by assigning the correct permissions to the file

```sh
 * keychain 2.8.5 ~ http://www.funtoo.org
 * Starting ssh-agent...
 * Adding 1 ssh key(s): /home/blu/.ssh/ansiblekey
 * Error: Problem adding; giving up
```

```sh
# Fixing it
sudo chmod 600 ~/.ssh/ansiblekey
source .bashrc
```

***
# Proxmox CPU high usage

Started troubleshooting this one because the NUC got super loud when running plex. A quick search shows that proxmox by default sets all CPU governors to "performance". This eats up a lot of usage and power, and is not necessary for most of the services. 

**NOTE:** This troubleshooting is specific to your CPU, only do this if you have an intel CPU. Might need to do some different things for AMD. 

Firstly, lets check what everything is set to, and monitor the output: 

```sh
# Check which CPU governors are being used 
sudo cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
#Check available governors for your CPU
sudo cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_available_governors

#This command will poll your CPU every 2s for performance: 
watch "lscpu | grep MHz"

# Output: 
CPU(s) scaling MHz:                   85% # This number kept spiking up to 90%+
CPU max MHz:                          4500.0000
CPU min MHz:                          400.0000
```

We can see the CPU is struggling, although in proxmox the CPU usage is sitting at around 2%. Clearly something is up. Setting governors to powersaving mode to test this out. 

```sh
# Set the CPU governors, make sure to check available ones before setting. 
echo "powersave" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Test it out after changing
watch "lscpu | grep MHz"
# ---
CPU(s) scaling MHz:                   25% # Immediately dropped by a lot 
CPU max MHz:                          4500.0000
CPU min MHz:                          400.0000
```

Seems to have fixed it. After stress testing, performance seemed to have improved.
# Hardware Transcoding Woes
## GVT-g GPU Passthrough (Why you shouldn't use a VM)

**!--- WARNING! GVT-g is deprecated! do not use this! ---!**

https://pve.proxmox.com/pve-docs/pve-admin-guide.html

https://cetteup.com/216/how-to-use-an-intel-vgpu-for-plexs-hardware-accelerated-streaming-in-a-proxmox-vm/

https://github.com/intel/gvt-linux/wiki/GVTg_Setup_Guide

 To properly implement hardware transcoding using the iGPU in the intel NUC, have to enable GPU passthrough. Proxmox supports GVT-g for intel GPUs, used to virtualize the GPU for VMs. There are a few things to edit on the main node to enable this feature. Used a mix of the articles above to struggle through this: 

```sh
# Update your grub file
sudo nano /etc/default/grub
...
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on i915.enable_gvt=1"
...

# Update grub to apply the changes
update-grub
```

`i915` is the intel graphics driver used in the linux kernel

`intel_iommu` is "Input-Output Memory Management Unit". It is essentially a memory manager for hardware devices like GPUs, allowing for mapping across virtual devices while keeping everything securely isolated.

Reboot!

```sh
# ! ---- Reboot the node ---- !
# When booted back up, find your cpu:
lspci 

# Output from lspci showing the iGPU
00:02.0 VGA compatible controller: Intel Corporation CoffeeLake-U GT3e [Iris Plus Graphics 655] (rev 01)

# Verify its working
dmesg | grep -i gvt
# Should see this:
i915 0000:00:02.0: GVT: GVT-g device initialized

```

I did not in fact see that line, some ChatGPT troubleshooting: 

```sh
# Check i915 is loaded
lsmod | grep i915 
# Should see similar output, good here
i915                 2408448  2

# Check GVT-g support
find /sys/module/i915/parameters/enable_gvt
# If the file exists, GVT-g is compiled

# Check this one out too. IF you get a hit your good
modinfo i915 | grep gvt

# This one hit me with an error: 
sudo ls /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/
ls: cannot access '/sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/': No such file or directory
```

The error shows that GVT-g did not initialize at runtime. According to ChatGPT this means my kernel is too new.... This feature has been deprecated by intel, and newer kernels cannot support. Annoyingly, this puts me at a crossroads yet again: 

- Skip hardware transcoding and maintain sanity (not an option)
- Downgrade to an older kernel for GVT-g (no)
- Switch to an LXC for easier GPU access 
- Pass the full GPU to plex, and sacrifice console access and GPU on node 

Bottom 2 are the best options, but we already have an issue with the LXC, it cannot mount via NFS. This can be done in a really roundabout way, so plex server round 3...

## LXC GPU Passthrough (The hard way)

**NOTE:** This section requires rebooting the node potentially multiple times while updating configs. It is also not necessary, less secure, and a pain in the ass. See [[Plex Server#LXC Setup]]
### Security Considerations

Files on the host filesystem that are accessible by the `render` group could now be modified in the event of a container escape vulnerability. Few files belong to that group and this configuration is much more constrained than a privileged container, so risk is low.
### Mounting host GPU to Container

Double check you have installed all of the correct drivers on the host:

```sh 
sudo apt install -y intel-media-va-driver vainfo

# Run this to check out some GPU info
vainfo
```

Check on the GPU for the proxmox host

```sh
# Read the subuid and subgid, note output, you will need it later.
cat /etc/subuid
cat /etc/subgid

# View GPU device 
ls -l /dev/dri

drwxr-xr-x 2 root root         80 Apr 22 00:44 by-path
crw-rw---- 1 root video  226,   0 Apr 22 00:44 card0
crw-rw---- 1 root render 226, 128 Apr 22 00:44 renderD128
```

Should show ~`/dev/dri/card0`, `/dev/dri/renderD128`, etc. Note that the renderD128 device is a character device denoted by the 'c' at the beginning of its permission attributes. The device has "major number" `226`, and minor number `128`, accessible by the render group. 

Now we have to bind the device to the container:  

```sh
# On the proxmox host
sudo nano /etc/pve/lxc/<container_id>.conf
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

`cgroup2.devices.allow`	Allows the container to access the character device 226 (DRI). The ":\*" basically just says allow all numbers 
`mount.entry` Binds the host’s /dev/dri into the container’s /dev/dri.

Bind-mount into the LXC:

```sh
sudo nano /etc/pve/lxc/<container_id>.conf
...
mp0: /prox/node/path,mp=/LXC/path,ro=1 # allows container read-only
```

Reboot the container to set the mount, and run the following command on the container to ensure you can access `card0` and `render128`:

```sh
ls -l /dev/dri

drwxr-xr-x 2 root root         80 Apr 22 00:44 by-path
crw-rw---- 1 root video  226,   0 Apr 22 00:44 card0
crw-rw---- 1 root render 226, 128 Apr 22 00:44 renderD128

```

### Setting up Group mapping

https://gist.github.com/packerdl/a4887c30c38a0225204f451103d82ac5?permalink_comment_id=4494850

This one is tricky. I'll let this article explain it better than me. I recommend reading the link:

"LXC helps isolate containers by mapping a set of user and group ids on the host to a set of user and group ids in the container. Normally a set of ids outside the POSIX range of 0-65535 is used on the host to map container user and groups. By default, Proxmox has LXC configured to map host user and groups 100000-165535 to container user and groups 0-65535. In the event of a container escape exploit, the malicious user from the container would not have permissions to modify the host filesystem."

Because of the mapping done by proxmox node, we will need to do some manual group mapping to make sure everything works. 

```sh
# List all groups for the host, do this on both hosts:
getent group
# Look for these entries in the output. They will likely be different numbers: 
...
render:x:104: # You need this one from your host
plex:x:996: # And this one from your container
...
```

Update the `subgid` file on the node

```sh
# Edit the subgid file to authorize groupid mapping
sudo nano /etc/subgid

root:100000:65536 # Add root user if it doesn't exist
root:104:1 # Add in this line right here, for the render group
blu:165536:65536 
```

Map the ids to the container. 

The idea here is to map the `plex` group in the container to the `render` group in the proxmox node. This should allow for the container to access the GPU. 

```sh
# Edit the container config
sudo /etc/pve/lxc/<container_id>.conf

# Column 1: u/g define map for user or group ids
# Column 2: Range start for container
# Column 3: Range start for host
# Column 4: Length of range
# i.e., g 0 100000 996 = Map gids 100000-100997 on host to 0-995 in container
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 996
lxc.idmap: g 996 104 1 # This one is the key, plex group --> render group
lxc.idmap: g 997 100999 64539 # Do the math for your group, i.e 65536 - plex
```

Im going to explain this a little more because it hurt my head. Basically what is going on here are we are defining 3 group ranges and 1 user range to map: 

```sh
lxc.idmap: u 0 100000 65536 # Map ids from 0-65536 on  LXC to 100000-165536 on node
lxc.idmap: g 0 100000 996 # Map groups from 0-995 on LXC to 100000-100996 on node
lxc.idmap: g 996 104 1 # Map group 996 on  LXC to 104 on node - the important one
lxc.idmap: g 997 100999 64539 # Map groups 997-65536 on LXC to 100999-165536 on Node
```

Note: for this config, you basically want to make sure there is only one range that will overlap between node and LXC, the `plex` and `render` groups. This is achieved in line 3, while the rest just maps the remaining numbers. 

Finally, reboot the node to set the changes to gid mapping, and boot up the container. 

```sh
# This should output a bunch of gpu info
vainfo

# Sample Output, omitting profile lines
error: XDG_RUNTIME_DIR is invalid or not set in the environment.
error: cant connect to X server!
libva info: VA-API version 1.17.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_17
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.17 (libva 2.12.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 23.1.1 ()
vainfo: Supported profile and entrypoints
```


# "Get Deadline Exceeded" Loki errors

Started getting spammed with these errors: 

```
level=debug ts=2025-05-12T04:24:02.3626804Z caller=mock.go:153 msg=Get key=collectors/ring wait_index=8
level=debug ts=2025-05-12T04:24:02.462682018Z caller=mock.go:189 msg="Get - deadline exceeded" key=collectors/scheduler
level=debug ts=2025-05-12T04:24:02.462795287Z caller=mock.go:153 msg=Get key=collectors/scheduler wait_index=5
level=debug ts=2025-05-12T04:24:02.462826663Z caller=mock.go:189 msg="Get - deadline exceeded" key=collectors/compactor
level=debug ts=2025-05-12T04:24:02.462839299Z caller=mock.go:153 msg=Get key=collectors/compactor wait_index=6
level=debug ts=2025-05-12T04:24:02.462922458Z caller=mock.go:189 msg="Get - deadline exceeded" key=collectors/distributor
level=debug ts=2025-05-12T04:24:02.462939473Z caller=mock.go:153 msg=Get key=collectors/distributor wait_index=4
```

Annoying, because it clogs up the logging console. I could just set the parser to ignore these, but trying to fix instead. Seems to be related to the "ring store" in Loki: 

https://grafana.com/docs/loki/latest/get-started/hash-rings/

This seems to be related to me using the default `inmemory` ring configuration, probably due to me using pretty restricted resources for the machine. Regardless, the docs recommend to use the `memberlist` key-value store type so will be using that instead. Some updates to the loki config: 

```yaml
common:
  ring:
  kvstore:
    store: memberlist

memberlist:
  join_members:
    - 127.0.0.1:7946
```

This is essentially stating that the machine will be using a `memberlist` for its hash ring, and joins itself to the list of nodes. a bit dense but see here for some more info on the "gossip protocol" which is used by the ring in loki:
https://en.wikipedia.org/wiki/Gossip_protocol

This all seems to have fixed the output, a pretty dramatic difference in volumes is seen: 

![[Pasted image 20250512004306.png]]
# "missing sudo password" Ansible Errors

Once i finally got the basic ansible config working, I went and blew it up. To create something more flexible and future proof (aka overengineered), I split up the files and such:

```
blu@warpgate:~/ansible $ tree
.
├── ansible.cfg
├── convert-utf8.sh
├── docker-deploy-staging.yaml
├── hosts
├── host_vars
├── inventory
├── logs
├── playbooks
├── roles
│   ├── docker_deploy
│   │   ├── defaults
│   │   ├── tasks
│   │   └── vars
│   ├── vector-configs
│   │   ├── hosts
│   │   ├── sinks
│   │   ├── sources
│   │   └── transforms
│   └── vector_deploy
│       ├── files
│       ├── tasks
│       └── vector-configs
│           ├── hosts
│           ├── sinks
│           ├── sources
│           └── transforms

└── vault.yaml

```

As you can see it got a little crazy, but became a lot more flexible. See [[Ansible Control Node]] for setup details. After splitting all this up, I made a ton of random errors in the formatting of the toml files. 

One error that had me pulling hair for awhile was this: 

```sh
ansible-playbook  
Vault password:
PLAY [Deploy vector config] ******************************
TASK [Gathering Facts] ***********************************
fatal: [caladan]: FAILED! =>
  msg: Missing sudo password

PLAY RECAP ***********************************************
caladan     : ok=0    changed=0    unreachable=0    failed=1    s                                                                                                                                                             kipped=0    rescued=0    ignored=0
```

This "missing sudo password" had me checking the obvious sources: 

- Checked the `vault.yaml` file to verify password and formatting.
- Checked the `host_vars` file to ensure I had set the variable.
- Checked the `hosts` file to ensure I named and assigned the host.

No issues in any of these, troubleshot with the following playbook to print the password in the console. 

```yaml
# Debug script to print sudo pass, checking if things work. 
- name: Debug sudo password
  become: true
  become_method: sudo
  vars_files:
    - ~/ansible/vault.yaml
  hosts: caladan
  gather_facts: no
  tasks:
    - debug:
        var: ansible_become_pass
```

immediately was evident that the variable was not being set, so something was wrong with a config file. After rooting around for a while, found the answer: 

```yaml
# If you intended to do any loops like I did here, make sure you are
# setting the variables you include in the curlies. If you miss one, 
# your playbook will stop with errors.

- name: Deploy source configs
  loop: "{{ vector_sources }}"
  loop_control:
    loop_var: src_file
  copy:
    src: "vector-configs/sources/{{ src_file }}"
    dest: "{{ vector_config_dir }}/sources/{{ src_file }}"
    mode: '0644'
```

After all the tinkering, turns out the fix is to just not use the yaml file. For whatever reason it is breaking things. Will revisit this later.