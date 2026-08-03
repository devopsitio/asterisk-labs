# Asterisk Labs

Ansible-based lab project for installing and testing Asterisk on virtual machines.

The current main lab target is:

- Ubuntu 24.04 VM
- Asterisk 22.5.2 built from source
- DAHDI Linux, DAHDI tools and LibPRI
- repeatable lab workflow with VM snapshots, Git and Ansible logs

This repository is for training and lab work. It is not a production-ready PBX deployment.

## Project structure

```text
.
├── ansible.cfg
├── bin/
│   ├── a
│   └── ap
├── inventories/
│   └── lab/
│       ├── hosts.example.yml
│       └── hosts.yml
├── playbooks/
│   ├── lab00-base.yml
│   └── lab01b-install-asterisk-source-ubuntu.yml
├── roles/
│   ├── common/
│   └── asterisk_source_ubuntu/
└── logs/
```

`logs/` is created locally and ignored by Git.

## Control machine requirements

The control machine is the computer where Ansible is installed and from where playbooks are executed.

Install required tools:

```bash
sudo apt update
sudo apt install -y ansible git openssh-client
```

Check Ansible:

```bash
ansible --version
```

Clone the repository:

```bash
git clone git@github.com:YOUR_USER/asterisk-labs.git
cd asterisk-labs
```

## Prepare target VM

Create a clean Ubuntu VM.

Recommended VM settings:

```text
OS: Ubuntu Server 24.04
CPU: 2 cores or more
RAM: 2 GB or more
Disk: 20 GB or more
Network: Bridged
IP: static IP or DHCP reservation
Example IP: 192.168.1.50
```

On the target VM, install minimal required packages:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y openssh-server sudo python3
sudo systemctl enable --now ssh
```

Create or use a regular user for Ansible.

Make sure the user can run sudo:

```bash
sudo whoami
```

Expected output:

```text
root
```

For a lab VM, passwordless sudo is convenient:

```bash
sudo visudo
```

Example for user `user`:

```text
user ALL=(ALL) NOPASSWD:ALL
```

Do not use passwordless sudo blindly on production servers.

After the VM is updated and SSH works, create the first clean VM snapshot:

```text
00-clean-ubuntu-24.04
```

## SSH access

On the control machine, create an SSH key if needed:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_asterisk_lab -C "asterisk-lab"
```

Copy the key to the target VM:

```bash
ssh-copy-id -i ~/.ssh/id_asterisk_lab.pub user@192.168.1.50
```

Check SSH access:

```bash
ssh -i ~/.ssh/id_asterisk_lab user@192.168.1.50
```

## Inventory setup

Copy the example inventory:

```bash
cp inventories/lab/hosts.example.yml inventories/lab/hosts.yml
```

Edit the inventory:

```bash
vim inventories/lab/hosts.yml
```

Example:

```yaml
all:
  children:
    asterisk:
      hosts:
        asterisk_lab_ubuntu_24:
          ansible_host: 192.168.1.50
          ansible_user: user
          ansible_become: true
          ansible_ssh_private_key_file: ~/.ssh/id_asterisk_lab
          asterisk_hostname: asterisk-ubuntu-24
```

Check connection:

```bash
ansible all -m ping
```

Or with the logging wrapper:

```bash
bin/a all -m ping
```

Expected result:

```text
pong
```

## Running playbooks

Use the wrapper scripts from `bin/` instead of running Ansible directly.

Ad-hoc command:

```bash
bin/a all -m ping
```

Run a playbook:

```bash
bin/ap playbooks/lab00-base.yml --limit asterisk_lab_ubuntu_24 --diff
```

### Lab 00: base OS setup

```bash
bin/ap playbooks/lab00-base.yml --limit asterisk_lab_ubuntu_24 --diff
```

This prepares the target VM with common base packages.

After a successful run, create a VM snapshot:

```text
lab00-base-<git-short-hash>
```

### Lab 01b: install Asterisk from source on Ubuntu

```bash
bin/ap playbooks/lab01b-install-asterisk-source-ubuntu.yml --limit asterisk_lab_ubuntu_24 --diff
```

This playbook builds and installs:

- DAHDI Linux
- DAHDI tools
- LibPRI
- Asterisk

The Asterisk version is configured in:

```text
roles/asterisk_source_ubuntu/defaults/main.yml
```

Example:

```yaml
asterisk_version: "22.5.2"
```

For Asterisk `22.5.2`, the tarball is downloaded from the Asterisk `old-releases` directory.

## Verify Asterisk

After installation, check the target VM:

```bash
ssh -i ~/.ssh/id_asterisk_lab user@192.168.1.50
```

Then run:

```bash
sudo asterisk -V
sudo systemctl status asterisk --no-pager
sudo asterisk -rx "core show version"
```

Enter the Asterisk CLI:

```bash
sudo asterisk -rvvv
```

Exit the CLI:

```text
exit
```

## Logs

The wrapper scripts save logs automatically into:

```text
logs/
```

Each run log includes:

- start time
- command
- Git branch
- Git commit
- Git status
- Ansible output
- exit code

Example:

```bash
bin/ap playbooks/lab01b-install-asterisk-source-ubuntu.yml \
  --limit asterisk_lab_ubuntu_24 \
  --diff
```

Log files are ignored by Git.

## Secrets

Do not commit secrets.

Keep provider credentials and local SIP configs outside Git.

Ignored examples:

```text
.env
*.env
*.local.conf
*.key
*.pem
*.crt
secrets/
logs/
captured/
```

For SIP provider credentials, use a local file on the VM, for example:

```text
/etc/asterisk/pjsip_zadarma.local.conf
```

This file must not be committed.

## Notes

This project intentionally keeps the workflow simple:

1. Prepare a clean VM.
2. Create a snapshot.
3. Run Ansible from the control machine.
4. Verify Asterisk.
5. Create another snapshot.
6. Commit changes to Git.

The goal is repeatable Asterisk lab work without turning the course into unnecessary CI/CD complexity.
