# #newt & #newt-dmz
# Summary

Pangolin uses `newt`, a Wireguard tunnel client, to establish tunnels to the reverse proxy running `traefik`. One of these clients is required to establish a tunnel to the network. I placed it on its own lightweight LXC to simplify management. 

There are 2 newts, one sitting in each DMZ. They correspond to 2 sites in Pangolin, `Coruscant`and `Coruscant-DMZ`. Media services are largely separated to another VLAN since they will be the ones interfacing with the outside world. See [[Network#**VLANs**]] for more info.
# Resources

The only difference is the IP assignment, and the vlan tag. Note from [[Network]] that my vlan setup has vlan 33 on 10.0.33.0/24. 

|            | newt          | newt-dmz       |
| ---------- | ------------- | -------------- |
| memory     | 512 MB        | 512 MB         |
| storage    | 1 GB          | 1 GB           |
| processors | 1 core        | 1 core         |
| ip         | 10.0.0.199/24 | 10.0.33.199/24 |
| bridge     | vmbr0         | vmbr0          |
| vlan       |               | 33             |

# Install and Setup

Setting up a new newt client is dead easy. Follow the [[Host Setup#Install and setup]] to get going. Make sure your [[Pangolin VPS Reverse Proxy]] has been setup. 

Logging into pangolin and creating a new site will give you a block of code for use with any connecting newt client. Looks similar to below. Copy this somewhere, it will only show it once: 

```sh
#These details are from the doc, you will need to generate your own:
./newt \
--id 31frd0uzbjvp721 \
--secret h51mmlknrvrwv8s4r1i210azhumt6isgbpyavxodibx1k2d6 \
--endpoint https://example.com
```

I created a basic script to just run this in a new tmux session with the terminal open: 

```sh
#!/bin/bash

new-session -d -s newt 'newt --id 31frd0uzbjvp721 --secret h51mmlknrvrwv8s4r1i210azhumt6isgbpyavxodibx1k2d6 --endpoint https://example.com';
tmux split-window;
tmux a;
```
