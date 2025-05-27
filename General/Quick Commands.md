# General Linux

## Network

```sh
# Restart ssh to apply settings
sudo systemctl restart ssh
sudo systemctl restart sshd

# File holding all the network config for linux
sudo nano /etc/network/interfaces

# Restart networking too if you need to
sudo systemctl restart networking.service

# Check system networking services
systemctl list-units --type=service | grep -i network
```

## Drive

```sh

#Grabbing id for swap to use for fstab entry to make permanent
sudo blkid /dev/sda2

#Shows free disk space
df -h

#Shows storage blocks
lsblk

#Resize filesystem to fit sda1
sudo resize2fs /dev/sda1

#Make and enable swap
sudo mkswap /dev/sda2
sudo swapon /dev/sda2
sudo swapon --show

```

## File

```sh
#listing all files in a directory and output to file:
find /mnt/media/media -type f -exec echo "put {}"\; > movies.txt
```

## User

```sh
#Read passwd file to see all users
cat /etc/passwd

#Search & print everything in this file up to the 3rd colon where uid is
awk -F: '{print $1 ":" $2 ":" $3}' /etc/passwd

# Add a user
useradd -m blu #Create user and generate home directory
usermod -aG sudo blu #Add blu to the sudo group
usermod -u 1030 blu #(Optional) This is to add to match synology permission
passwd blu #Set passwd
su blu

```

****
# Ansible

```sh
#Run playbook
ansible-playbook update-playbook.yaml -e @vault.yaml --ask-vault-password

#Run playbook with specific host
ansible-playbook update-playbook.yaml -l pangolin -e @vault.yaml --ask-vault-password

#Ping check all hosts
ansible all -m ping -e @vault.yaml -v --ask-vault-password
```

****
# Fail2Ban
#Fail2Ban
```sh
#Dry run and status check
sudo fail2ban-client -d 

#Statuseses
sudo fail2ban-client status #shows jail
sudo fail2ban-client status sshd #shows detailed sshd jail stats 
sudo systemctl status fail2ban #systemctl status

#Check journal for fail2ban entries
sudo journalctl -xeu fail2ban

#Restart fail2ban service
sudo systemctl restart fail2ban

```

***
# NFS

```sh
# Find all active nfs mounts
mount | grep nfs
```