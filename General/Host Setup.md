# Install and setup

This may vary, but pretty much the standard for every linux host setup that i do in my homelab. first things first: 

- **(Optional) If it is a VM, Enable xterm.js for better console!**
  https://pve.proxmox.com/wiki/Serial_Terminal
	- Go into "Hardware" in proxmox and add a serial port named "serial0"
	- In the "Console" dropdown top right, you should be able to pick xterm.js.
	- Reboot the host to apply. 
	- Update the `grub` file to use the interface:
	
```sh
# !!WARNING!! Messing up your grub file can brick your system. Be careful. 
nano /etc/default/grub

# The below line should exist but be blank, update with this value: 
...
GRUB_CMDLINE_LINX="quiet console tty0 console=ttys0,115200"
...

# Update grub with the changes, reboot
update-grub
reboot
```
***
## Basic Setup

- Remote into your newly created host
- Run updates, get common tools: 

```sh
apt install sudo # Why do i need to install this wtf
apt install net-tools # Optional but nice. netstat, arp, etc.
apt install tmux # Optional
apt install nfs-common # For any hosts requiring mount to NAS
apt update
apt dist-upgrade
apt autoremove
apt clean
```

- Create new user "blu". For any hosts that you are mounting to NAS via NFS, assign UID matching appropriate user in synology. See [[Synology NAS]] for more info. 

```sh
useradd -m blu #Create user and generate home directory
usermod -aG sudo blu #Add blu to the sudo group
usermod -u 1030 blu #(Optional) This is to add to synology group
passwd blu #Set passwd
su blu
```

## Docker

Run the docker setup script: 
https://docs.docker.com/compose/gettingstarted/

![[lxsetup-docker.txt]]
## Quality of life

Updating the default shell to `bash` and default editor to `nano`

```sh
chsh -s /bin/bash <user> # Sets user login shell to bash 
export EDITOR=nano  # make default editor nano
sudo update-alternatives --config editor # Prompts to pick default editor

```

## Network Setup

If you are setting manual ips, using vlans, etc, you might have to edit `/etc/network/interfaces`. On containers this can be edited through the proxmox ui easily, but in VMs its a bit trickier. Make sure you have the right bridge assigned under the Hardware tab in Proxmox, and tag your desired vlan as well. 

Once you have the interface setup in Proxmox, go into the host and open up the file

```sh
sudo nano /etc/network/interfaces

#------------------------------------------------------------
#Sample output, ens18 is what we care about:

# Loopback interface
auto lo
iface lo inet loopback

# Primary network interface
allow-hotplug ens18
iface ens18 inet static # Set DHCP to static 
 address 10.0.33.33/24 # Desired IP and subnet.
 netmask 255.255.255.0 
 gateway 10.0.33.1 # If using Unifi, use the gateway assigned to your vlan.
 dns-domain sarlacc.lan # Optional (pihole)
 dns-nameservers 10.0.0.201 # Optional (pihole)
#------------------------------------------------------------

# Run this after config to apply settings and restart network
sudo ifdown ens18 # Bring down the main network interface to reset it
sudo systemctl restart networking
```

## Basic SSH configs

- mess with some ssh settings for host security. More advanced setup later:

```sh
sudo nano /etc/ssh/sshd_config 

# Set the following parameters in sshd_config: 

Port XXXX # Custom ssh port
PermitRootLogin to No # Allow root login -> no
MaxAuthTries 6 # Max login tries
MaxSessions 10 # Max ssh sessions
PasswordAuthentication no #(Optional) disable password login for SSH. 

PubkeyAuthentication yes # (Optional) For using an ssh key
AuthorizedKeysFile      .ssh/authorized_keys # (Optional) For using an ssh key


# Restart ssh to apply settings
sudo systemctl restart ssh
sudo systemctl restart sshd

# Restart networking too if you need to
sudo systemctl restart networking.service
## Shell environment stuff
```

***
# SSH Setup

Setting up `ssh` keys for the host, and optionally `keychain`. This step is optional, and only required if you want to use ssh keys to connect securely to your host. 

Use `keychain` in order to simplify auth and key management. This is only really worth it for hosts that will be initiating a ton of ssh sessions for automation (ex. [[Ansible Control Node]])

```sh
#Optional, create key. I prefer to do this in puttygen.
ssh-keygen -t eddsa -b 521 -C "ansible-control-node"

#Create directory and authorized_keys file, if it doesn't exist
mkdir -p /home/blu/.ssh && touch /home/blu/.ssh/authorized_keys

#Open up the file and add the public key from puttygen
nano /home/blu/.ssh/authorized_keys

# Assign permissions
sudo chmod 700 ~/.ssh
sudo chmod 600 ~/.ssh/authorized_keys
sudo chmod 600 ~/.ssh/ansiblekey # chmod any other keys you add.
```

A common issue when setting this up is file permissions. Make sure your key files are owned by the user that they are intended for, and that the permissions match the above. 

Puttygen steps: 
- generate a key in puttygen, pick your encryption and options.
- export private key. Use `Conversions` tab up top to export in a specific format, choose OpenSSH format.
- copy over public key from puttygen output into a file named `<<key>>.pub` . 
- **`ssh-agent`** is running and properly loaded with your keys.
## Keychain

```sh
#Using keychain for ssh mgmt
sudo apt install keychain

#Start ssh-agent using keychain. Starts ssh-agent, loads SSH key, and exports environmental variables. 
eval "$(keychain --eval ~/.ssh/ansiblekey)"

#Adding SSH private key to keychain. Enter passphrase once, and will be cached for the session.
ssh-add ~/.ssh/ansiblekey

#To check loaded keys
ssh-add -l

#to copy to other hosts:
ssh-copy-id -i ~/.ssh/ansiblekey.pub blu@10.0.33.33

#If you changed the ssh port: 
ssh-copy-id -i ~/.ssh/ansiblekey.pub -p <XXXX> blu@10.0.33.33
```

To automatically add your key to the keychain on startup, add the following to your `~/.bashrc` file

```sh
eval "$(keychain --eval ~/.ssh/ansiblekey)"
```



***
# NFS Connection

See [[Troubleshooting#NAS NFS connection troubleshooting]] for the struggles.

If you dont have any VLANs in your way, setting up your NFS connection is not too bad at all. Make sure you setup your [[Synology NAS]] permissions correctly. **It is very important that your UID or GID of the connecting user in Linux has a matching UID or GID to the host for which you assign permissions on the NAS**

If you happen to be using VLANs, its a bit of a pain. I had endless issues with this one to start, and it all boiled down to the firewall rules and permissions between the devices. 

Firstly, on the NAS, find the UID of whatever user you assigned permissions in the GUI

```sh 
#On the NAS
#Print everything in this file up to the 3rd colon where uid is
awk -F: '{print $1 ":" $2 ":" $3}' /etc/passwd
```

Now on the client that will be connecting to the share

```sh
#On the client
sudo apt install nfs-common #Dont forget this 
mkdir /path/to/mount #I use /mnt/media
sudo usermod -u 1030 blu #Assign UID if needed

#Mount it
sudo mount -t nfs <insert_ip_here>:/NAS/path /path/to/mount
```

If all went well, you should be mounted. Run a quick `ls` to check. You are gonna wanna try to `touch` some test files in the directory to check permissions before you get too far. This mount is a one time thing, so to make it permanent, add an entry to `/etc/fstab`

```sh
# Add this entry to fstab on the last line
<insert_ip_here>:/NAS/path /path/to/mount nfs defaults 0 0

# Run this afterward to apply
sudo systemctl daemon-reload
```

If you have permissions issues trying to `touch` files, make sure to check your NAS permissions and also check this: 

```sh
#Print everything in this file up to the 3rd colon where uid is
awk -F: '{print $1 ":" $2 ":" $3}' /etc/passwd
```

Now if you hate yourself like me and you are being stared down by VLANs, there's extra work to do on the firewall side. See [[Network#**DMZ**]] for more info, but here is the rule:

| Name            | Action | Type      | Protocol | Source         | Port | Destination | Port         |
| --------------- | ------ | --------- | -------- | -------------- | ---- | ----------- | ------------ |
| NFS Media Mount | Accept | LAN Local | TCP/UDP  | `Media Server` | -    | `NAS`       | 111,892,2049 |
***
# Vector Agent Setup

For linux, install it via apt: 

```bash
curl --proto '=https' --tlsv1.2 -sSfL https://sh.vector.dev | bash
```

for compose:

```yaml
  vector:
    image: timberio/vector:0.46.1-debian # No latest tag for vector
    container_name: vector
    volumes:
      - ./config/vector/vector-config.yaml:/etc/vector/vector.yaml # Vector Config
      - /var/log:/var/log:ro # Host Logs
      - /var/lib/docker/containers:/var/lib/docker/containers:ro # Docker Logs
      - /mnt/logs/vector:/vector-data # Vector Logs
    depends_on:
      - loki
    restart: unless-stopped
    networks:
      - monitoring
```



# Fail2Ban setup

Fail2ban is a public project that scans log files, and bans ips that attempt too many logins. Needless to say this is a good idea to place on any remotely exposed hosts. Currently running on `rishi-station`, the [[Pangolin VPS Reverse Proxy]], as it is the only exposed device currently. 

Just run this single command. Its that easy. Smooth.

```sh
# --- Install steps on rishi-station, debian 12, vol1 edition2 ---

# Install it
sudo apt install fail2ban
```

TL;DR, here is the fix: 

```sh
# Insert jail here
sudo nano /etc/fail2ban/CECOT.local

# Input the following
[sshd]
enabled = true
backend = systemd # Logs through systemd instead of log files
```

Of course not smooth, immediate errors when running which I expected. tried to run the status command, saw the error, and then checked the status. It is truncated, but we can see that it is `Loaded: loaded, enabled, present`. We also see it shows `Active: failed`:

```sh
#Check status
sudo systemctl status fail2ban

× fail2ban.service - Fail2Ban Service
     Loaded: loaded (/lib/systemd/system/fail2ban.service; enabled; preset: ena>
     Active: failed (Result: exit-code) since Mon 2025-04-14 15:46:39 EDT; 1min>
   Duration: 133ms
       Docs: man:fail2ban(1)
    Process: 4081025 ExecStart=/usr/bin/fail2ban-server -xf start (code=exited,>
   Main PID: 4081025 (code=exited, status=255/EXCEPTION)
        CPU: 118ms

Apr 14 15:46:39 rishi-station systemd[1]: Started fail2ban.service - Fail2Ban S>
Apr 14 15:46:39 rishi-station fail2ban-server[4081025]: 2025-04-14 15:46:39,523>
Apr 14 15:46:39 rishi-station fail2ban-server[4081025]: 2025-04-14 15:46:39,535>
Apr 14 15:46:39 rishi-station fail2ban-server[4081025]: 2025-04-14 15:46:39,536>
Apr 14 15:46:39 rishi-station systemd[1]: fail2ban.service: Main process exited>
Apr 14 15:46:39 rishi-station systemd[1]: fail2ban.service: Failed with result >
```

Clearly not ideal, checking for issues in logs:

```sh
# Doing a 'dry run' and printing issues: 
sudo fail2ban-client -d 

# Saw this near top of output: 
2025-04-14 15:53:35,721 fail2ban.jailreader [4084266]: WARNING Have not found any log file for sshd jail
```

Have to build a jail to send all the bad ssh's to:

```sh
# Insert jail here
sudo nano /etc/fail2ban/jail.local

# Input the following
[sshd]
enabled = true
backend = systemd # Logs through systemd instead of log files

# Basic fail2ban status
sudo fail2ban-client status
Status
|- Number of jail:      1
`- Jail list:   sshd`

# sshd status
sudo fail2ban-client status sshd

Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
```

Jail is built, looks like its working, now just restart the service and run another status check: 

```sh
# Restart and status
sudo systemctl restart fail2ban
sudo systemctl status fail2ban

● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/lib/systemd/system/fail2ban.service; enabled; preset: enabled)
     Active: active (running) since Mon 2025-04-14 16:05:23 EDT; 4s ago
       Docs: man:fail2ban(1)
   Main PID: 4089203 (fail2ban-server)
      Tasks: 5 (limit: 2314)
     Memory: 15.4M
        CPU: 162ms
     CGroup: /system.slice/fail2ban.service
             └─4089203 /usr/bin/python3 /usr/bin/fail2ban-server -xf start

Apr 14 16:05:23 rishi-station systemd[1]: Started fail2ban.service - Fail2Ban Service.
Apr 14 16:05:23 rishi-station fail2ban-server[4089203]: 2025-04-14 16:05:23,308 fail2ban.co>
Apr 14 16:05:23 rishi-station fail2ban-server[4089203]: Server ready

```