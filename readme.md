# Ansible notes

Ansible automates IT tasks with an agentless architecture, requiring no additional software on target nodes, and it operates using a simple, human-readable YAML syntax for defining tasks. A key feature is its idempotency, ensuring that operations produce the same results when executed multiple times, thus providing consistent and reliable automation outcomes.

It requires no software installation on the targets (agentless architecture)

## Installation + Tests setup

Local machine: `sudo apt-get install ansible`

Test server: set up virtualbox machine with ubuntu server

- Hostname: vhome_server
- User: pk
- Pass: vpass100
- **Use bridged network**
- Enable / install openssh

Enable ssh on the home server:

```shell
sudo apt update
sudo apt install openssh-server
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
sudo systemctl enable ssh # start ssh service at boot
sudo systemctl start ssh # start service immediately for the first time without reboot
sudo systemctl status ssh # check if service is running
sudo ufw allow ssh # allow ssh connections in firewall (ubuntu)
```

Test ssh connection (from host):

```ssh pk@192.168.178.45```

## Create ssh key file

So we don't have to log in manually, we can create a ssh key file for authentication:

On host:

```shell
ssh-keygen \
-o \ # Enforce the use of the new OpenSSH format for the key
-a 100 \ # Number of rounds of the key derivation function to use, making it more secure
-t ed25519 \ # Specify the type of key to create, in this case, Ed25519
-f ~/.ssh/vhome_server \ # The filename of the key file
-C "vhome@example.com" # A comment to identify the key, usually an email address
```
- empty passphrase

Copy the key to the home server:

`ssh-copy-id -i ~/.ssh/vhome_server pk@192.168.178.45`

Test passwordless ssh login again with

```ssh pk@192.168.178.45```


## Inventory

### hosts

Create an inventory (hosts) file of the infrastructure.

In the hosts file, the **hosts** (= **targets** = **nodes**) are defined.

[Different formats can be used, we use a hosts.ini](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#inventory-basics-formats-hosts-and-groups)

hosts.ini
```ini
[local]
127.0.0.1 ansible_connection=local

[vhome] # group name
vhome_server ansible_host=192.168.178.45 ansible_user=pk ansible_connection=ssh ansible_ssh_private_key_file=~/.ssh/vhome_server
```

### config

Create `ansible.cfg`

```ini
[defaults]
INVENTORY = hosts.ini

[ssh_connections]
pipelining = true # speeds up connections
```

Test ansible connection(s):

`ansible all -m ping`

## Tasks / playbooks

- Playbook format: yaml
- a **playbook** consists of an ordered list of "**plays**"
- each **play** consists of one of more **tasks**
- each play consists of *at least*:
  - name (optional but should always be included)
  - the target node(s) (= hosts = targets)
  - at least one task to execute
  - tasks are ansible built-ins or from plugins
- each **task** calls an ansible **module**
- all playbooks, plays and tasks run in order from top to bottom
- each play and task can (and should) have a name

We create a base task playbook that does the following on the target system(s) (=nodes) 
- updates all packages 
- installs essential software
- enable passwordless sudo for the login user
- disable password authentication for ssh
- enable public key only authentication

Note: all names / directories are arbitrary used here, the playbooks can be placed wherever within the main directory (the one containing the hosts definitions)

- Create a `tasks` folder in the folder where the previous config files were created.
- Create a `essential.yml` file - this is a playbook

```yaml
- name: Update packages # first play
  # apt is an ansible built in command
  # the following are parameters for the ansible apt command, not the node's apt program
  # update_cache=yes: run apt-get update first
  # force_apt_get=yes: use apt-get instead of apt
  # cache_valid_time=3600: update apt cache if older than 3600 seconds
  # Note that the following "style" would also work:
  # apt: update_cache=yes force_apt_get=yes cache_valid_time=3600
  apt:
    update_cache: yes
    force_apt_get: yes
    cache_valid_time: 3600
    # the meaning of state: latest depends on the ansible command (here: apt)
    # here, it updates all packages to the latest state;
    state: latest
  # to only update to the latest 

- name: Install essential packages
  package:
    name:
      - vim
      - neofetch
      - htop
      - tmux
      - speedtest-cli
    state: latest # install and update to the latest state available
```