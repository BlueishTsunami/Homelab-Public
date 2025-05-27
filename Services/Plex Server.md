# #jumbotron

# Summary

`jumbotron` is living in the DMZ by itself running Plex Media Server. Plex is installed through the Linux package downloaded from the site. The documentation kind of sucks on Plex and was not the easiest to dig through. `jumbotron` is using `newt-dmz` to tunnel to the VPS running pangolin, exposing port 32400 with no authentication. This really does nothing besides make the service route through the VPS rather than my home ip. See the DMZ section for more info.

Started this one in an LXC, but there's a few issues here: 

- Unprivileged LXC's cannot mount via NFS
- Privileged LXC's can mount via NFS, but are less secure. 
- Even though privileged containers can mount, they boot with a blank `fstab` file

All of these complications made me just switch to a VM. Issues with a VM: 
- Newer Kernels do not support GVT-g for intel GPU virtualization

Then I switched back to an LXC, unprivileged this time. 

## To DMZ or not to DMZ

Why you probably don't need DMZ. skip the rest of this section if you aren't using one:
https://words.filippo.io/how-plex-is-doing-https-for-all-its-users/

*I tried so hard, and got so far, but in the end, it doesn't even matter*. If you're all about sunk cost and you're gonna get the DMZ working even if its relatively pointless, some extra considerations during setup. On initial install, the documentation points you to 

`https://<server_ip>:32400/web`. 

This will not work if you are on a different network than the plex host, i.e. your desktop computer on another VLAN. The least painful way will be to assign some desktop host to the DMZ VLAN and complete the setup. I did this with a small VM running in proxmox, but you can just do this with your desktop host too. 

The only other way to do this is to setup an ssh tunnel between the hosts to forward port 32400 over. I could not get this one to work, probably firewall rule issues: 

```sh
# https://support.plex.tv/articles/200288586-installation/
# Using PuTTy on Windows 

- **Gateway**: ip.address.of.server
- **Source Port**: 8888
- **Destination**: 127.0.0.1:32400

Once you have the SSH tunnel set up:

1. Open a browser window
2. Type `http://127.0.0.1:8888/web` into the address bar
```

## To Tunnel or not to Tunnel

Just when you thought the crisis was over, here we go again. The two options are:

1. Open up and forward port 32400 on home firewall to `jumbotron`
	- Objectively easier solution
	- Probably secure enough, as long as plex is updated
	- Exposes port 32400 on your home IP (pokin holes)
	
2. Use `rishi-station` to tunnel all the traffic via Traefik
	- You get to use your own domain and justify sunk costs
	- Probably a bit more secure routing traffic via encrypted tunnel
	- Does not require firewall rules
	- Exposes the service on your VPS instead of home network.
	- Does not require you to enable "External Access" on the server.

Going with option 2 for these purposes, mainly coming down to slightly better perceived security, as well as eliminating firewall rules and keeping the table clean. This is also easy enough to setup so might as well get the most out of the VPS. 

The only requirement for this one if you are using a DMZ is to ensure you have a newt client running somewhere on the network, and a DMZ site setup to your subnet in pangolin. 

# Hardware

Made this one a bit beefier to handle the transcoding loads, which were redlining the CPU at 4 cores. 

| Type       | LXC Debian 12     |
| ---------- | ----------------- |
| memory     | 4 GB (4 GB Swap)  |
| storage    | 32 GB             |
| processors | 4 core (1 socket) |
| ip         | 10.0.33.33/24     |
| bridge     | vmbr0             |
| vlan       | 33                |

# Install and Setup

So fresh plex LXC, do the usual [[Host Setup]], and run this command

```sh
# On the LXC...
# List all groups for the host
getent group | grep plex

plex:x:996:blu # You need this one from your container, note the number
```

Then power down while we update configs. Make sure you know the number of the plex group from the above command.

Mount the proxmox node to your storage using [[Host Setup#NFS Connection]]. 

Then check some things on the node:

```sh
# On the proxmox node... 
# Get drivers, and get vainfo 
sudo apt install -y intel-media-va-driver vainfo
vainfo

sudo nano /etc/pve/lxc/<container_id>.conf

# Add these lines
mp0: /prox/node/path,mp=/LXC/path,ro=1 # Allows container read-only
dev0: /dev/dri/renderD128,gid=996 # Allow container GPU passthrough

# Reboot the node, and start up the container to test. 
```

That's it. You should now have passthrough. See [[Troubleshooting#Hardware Transcoding Woes]] as a reminder of why you should read things. 

```sh
# Download the plex img and get it onto your server
sudo dpkg -i plex_image_file.iso

# Add Plex repo to trusted apt file for easier updates
sudo apt install curl
sudo apt install gpg
curl https://downloads.plex.tv/plex-keys/PlexSign.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/plex.gpg
```

Thats pretty much it. To test if hardware transcoding is working, run the command below or check out this link for instructions in the ui
https://support.plex.tv/articles/115002178853-using-hardware-accelerated-streaming/#toc-5

```sh
# Find any instance of "Hardware Transcode" in logs
cat /var/lib/plexmediaserver/Library/Application\ Support/Plex\ Media\ Server/Logs/Plex\ Media\ Server.log | grep -i "hardware"
```

With 4 concurrent streams going all transcoding, I am barely scratching the resources on my NUC, and its silent. 
# Networking

Pretty straightforward. If you are using pangolin, setup a resource pointing to `https://<plex_ip>:32400` without authentication. If you are using a DMZ, you will need another [[Newt Clients]] and setup a DMZ site in pangolin.
## Plex web settings for tunneling

- Go to Settings > External Access and DISABLE External access, it is not required for this setup. 
- Go to Settings > Network and add your custom server URLs: 

```sh
# Set 2 custom URLs. INCLUDE HTTPS AND PORT!!
# One for your domain on 443 for the tunnel, 
# One with local ip on port 32400 for local access.  
https://plexserver.domain:443,https://10.0.33.33:32400
```

- Set secure connections from Preferred to Required
- Disable Plex Relay 

| NOTE: If you have issues with your connection appearing as "indirect", it is most likely due to the secure connections setting, or the custom server URL setting. I had a lot of issues at first, and it was due to these configs. If all went well, it should look like the pic on the right when signing into app.plex.tv from a remote host: | ![[Pasted image 20250416150435.png]] |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |


# Troubleshooting and lessons learned

I hate how easy this was in the end. See  [[Troubleshooting#Hardware Transcoding Woes]]

