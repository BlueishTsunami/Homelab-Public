# #warpgate

# Summary

`warpgate` is a Raspberry Pi 4B being used for device management. Primarily running ansible for automating management of devices, and will likely incorporate ntfy to send push notifications for issues. Also may use for newt endpoints. 

See [[Rpi 4 - Ansible host]] for more details on the hardware. 

# Install and Setup

Just like everything else, run through linux host setup. 

From here, need to start installing ansible. Already have python3 installed, so: 

```sh
# !--- Make sure you switch away from root! ---!

sudo apt install pip
sudo apt install pipx
sudo pipx ensurepath #Auto-adds PATH variable

#This did not work to add PATH, so had to run the following: 
export PATH=$PATH:~/.local/bin

#Had to update the .bashrc file to fix

#Installing full ansible package because why not
pipx install --include-deps ansible

#For updates later: 
pipx upgrade --include-injected ansible

#To install dependency: 
pipx inject ansible <dependency>

```


## SSH Setup

Follow [[Host Setup#SSH Setup]] using `keychain` in order to simplify auth and key management. 

```sh
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

- **Private key** (`ansiblekey`) is in OpenSSH format.
- **Public key** (`ansiblekey.pub`) is copied directly from PuTTYgen’s output, ready to use.

## Ansible Setup:

```sh
#Setup hosts file
mkdir ~/ansible
nano ansible/hosts
```

The host file contains groupings of hosts that we want to manage. Groups are designated with brackets like`[host-group]` and help to organize the hosts. Playbooks can be run against groups. 

Another note with the hosts file is that there are ansible parameters that can be added, such as if ssh has been changed to a non-standard port. See more here
https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#connecting-to-hosts-behavioral-inventory-parameters

```sh
#List your groups and hosts, with your ssh ports specified if needed.
[DMZServers]
10.0.33.33 ansible_port=<insert_ssh_port>

[TrustedServers]
<insert_ips>

[VPS]
<insert_ips>

```

Once the host file has been setup, you can refer to all hosts at once by using 'all' with ansible. See this in use below to perform an SSH test against all hosts in the host file:

```sh
#Me trying to debug the Synology
ANSIBLE_DEBUG=1 ansible 10.0.0.23 -i ~/ansible/hosts -m ping -v

#ping all
ansible all -i ~/ansible/hosts -m ping -v
```

Now that all the hosts are connecting properly, create a config file: 

```ini
[defaults]
inventory = ~/ansible/hosts
remote_user = blu
host_key_checking = False
retry_files_enabled = False
timeout = 30
stdout_callback = yaml
forks = 10
log_path = /home/blu/ansible/logs/update-playbook.log

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False
```

`inventory = ~/ansible/hosts`
Sets the host inventory file for ansible

`host_key_checking = False`
Disable SSH host key fingerprint checking for easier automation

`retry_files_enabled = False`
Disables `.retry` file creation when a playbook fails. This keeps the folder clean since I'm not using this.  

`deprecation_warnings = False`
Can enable this to disable deprecation warnings.

`timeout = 30`
Sets how long Ansible waits for a response when connecting to a host.

`stdout_callback = yaml`
outputs everything in yaml

`forks = 10`
Sets how many hosts Ansible will manage **in parallel**. 5 is default, 10 is commonly used.

`log_path` 
Sets the log file for the update playbook

`[Privilege escalation]`
`become = True`
ells Ansible to **run tasks with privilege escalation** (like `sudo`).

`become_method = sudo`
Specifies **how** to escalate privileges. We use sudo.

`become_ask_pass = True
Tells Ansible **to prompt** for a sudo password when becoming another user.

ansible_become_pass={{ ansible_pass_fhloston }}

created a vault file in ansible to store all my passwords: 

```sh
#Create vault, enter password, etc
ansible-vault create vault.yaml

#File contents
jumbotron_sudo_password: XXXXXXX
kepler_sudo_password: XXXXXXX
newt_sudo_password: XXXXXXX
newtdmz_sudo_password: XXXXXXX
sarlacc_sudo_password: XXXXXXX
pangolin_sudo_password: XXXXXXX
```

Time to make host variables for the vault passes: 

```sh
mkdir -p ~/ansible/host_vars

#In the host_vars directory, make a file for each hostname and input: 
ansible_become_pass: "{{ fhloston_sudo_password }}"

```

Now to edit the `/ansible/hosts` file to use the hostnames assigned to each:

```ini
[DMZServers]
jumbotron ansible_host=<server_ip>
newtdmz ansible_host=<server_ip>

[TrustedServers]
newt ansible_host=<server_ip>
sarlacc ansible_host=<server_ip>
kepler ansible_host=<server_ip>

[VPS]
panoglin ansible_host=<server_ip>

```

# Ansible Usage

Now that its finally setup hopefully, we can test it out and run a ping to make sure we didnt break everything. 

```sh
#Ping against group 'all', using vault.yaml ansible vault, asking for password.
ansible all -m ping -e @vault.yaml -v --ask-vault-password
```

Making the first playbook: 

```yml
# update_playbook.yml
---
- name: Run apt update on all hosts
  hosts: all
  become: true
  become_method: sudo
  vars_files:
    - ~/ansible/vault.yaml  # Include the Vault file containing the sudo password
  tasks:
    - name: Update apt package list
      apt:
        update_cache: yes

    - name: Upgrade all packages
      apt:
        upgrade: dist

    - name: Autoremove old packages # No apt package for autoremove
      shell: apt autoremove -y

    - name: Clean up cached packages # No apt package for clean
      shell: apt clean -y
```

and running it against a single host to test: 

```yml
ansible-playbook update-playbook.yaml -l pangolin -e @vault.yaml --ask-vault-password

#Sample output:
PLAY [Run apt update on all hosts] **********************************************************************
TASK [Gathering Facts] **********************************************************************
ok: [pangolin]
TASK [Update apt package list] **********************************************************************
ok: [pangolin]
TASK [Upgrade all packages] **********************************************************************
ok: [pangolin]
TASK [Autoremove old packages] **********************************************************************
changed: [pangolin]
TASK [Clean up cached packages] **********************************************************************
changed: [pangolin]
PLAY RECAP *********************************************************************
pangolin  : ok=5 changed=2 unreachable=0 failed=0 skipped=0 rescued=0  ignored=0
```

***
# Playbook Creation

Started this by creating a playbook for testing. First made a `playbooks` folder in the `ansible` project folder. Then created a new playbook:

```yml
# docker-deploy-staging.yaml

- name: Stage Docker app
  hosts: Staging # This is a group for staging hosts
  become: true

  tasks:
    - name: Ensure app directory exists
      file:
        path: "{{ staging_path }}" # Path to create a staging env
        state: directory
        mode: '0755'

    - name: Copy docker-compose file to staging # Self explanatory
      command:
        cmd: cp "{{ app_path }}/docker-compose.yaml" "{{ staging_path }}/docker>

    - name: Pull latest images 
      community.docker.docker_compose_v2:
        project_src: "{{ staging_path }}" # Pull to the staging path
        pull: always
```

# Directory Structure

The tree for my ansible directory. Setup to have separation between functions and playbooks, and allow for modular updates and CI/CD implementation. Will update the docs with more info soon.

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
# Quick commands

Some quick ansible commands since they are wordy

```sh
# Run playbook
ansible-playbook update-playbook.yaml -e @vault.yaml --ask-vault-password

# Run playbook with specific host
ansible-playbook update-playbook.yaml -l pangy -e @vault.yaml --ask-vault-password

# Ping check all hosts
ansible all -m ping -e @vault.yaml -v --ask-vault-password
```