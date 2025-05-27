# #Mustafar

# Summary

`mustafar` is the Synology NAS, doing NAS things on the network. In the context of this notebook, it is mainly used for media storage for [[Plex Server]]. This NAS was originally running my full media server setup, until it was moved to the NUC. The docker hosts remain dormant, in case I broke it all.
# Hardware

| Make           | Synology                  |
| -------------- | ------------------------- |
| Model          | DS918+                    |
| CPU            | Intel Celeron J3455       |
| CPU Clock Rate | 1.5GHz                    |
| CPU Cores      | 4                         |
| Memory         | 4 GB                      |
| Storage        | 4 TB (2x disks in RAID 1) |
| IP             | 10.0.0.23/24              |
| DNS            | `sarlacc`                 |
# File Hierarchy

**!! ---- IMPORTANT: Setup your hierarchy in a shared folder dedicated to your intended streaming content. Keep any personal pictures or documents in a separate folder, to avoid accidental modification ---- !!**

Home NAS server for use with media server / photos / documents. Since I want to use this for personal stuff as well as streaming, I have setup the directory structure to separate the two. See below for the preferred hierarchy of the shared folder `Data` from Synology

```sh
#Suggested folder layout
data # 'Data' nfs shared drive on the Synology. 
└── media
    ├── movies
    ├── music
    ├── books
    └── tv
```

## NFS Settings

Open up the Control Panel and go to `File Transfer Services`. Enable NFS, and check the box for NFS v4 if you plan to use it. 

Setup the following privileges on the folder in the Synology ui in the Control Panel:
`Shared Folders > edit > NFS Permissions`: 

| Client        | Privilege  | Squash     | Asynchronus | Non-priveleged ports | Cross-mount |
| ------------- | ---------- | ---------- | ----------- | -------------------- | ----------- |
| 10.0.33.33/24 | Read/Write | No Mapping | Yes         | Allowed              | Allowed     |

Also, dont forget to set the regular folder permissions in the Control Panel under:
`Shared Folders > edit > Permissions`: 

![[Pasted image 20250414173858.png]]

Pretend the name in the picture is "Plex"

Make sure that whatever user is assigned under the permissions in synology has a matching user id to the user that will be modifying the drive on linux. To find the uids for the users you have setup in synology, you can open a terminal and run this command:

```sh
#Read passwd file to see all users
cat /etc/passwd

#Print everything in this file up to the 3rd colon where uid is
awk -F: '{print $1 ":" $2 ":" $3}' /etc/passwd
```

Be sure to enable NFS and all that good stuff: 
https://kb.synology.com/en-us/DSM/tutorial/How_to_access_files_on_Synology_NAS_within_the_local_network_NFS

