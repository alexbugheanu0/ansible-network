# Ansible network automation

This repository is an Ansible project for managing Cisco switches over SSH. It is not a web application or long-running service. You "run the application" by executing Ansible playbooks against the devices in the inventory.

## What this project contains

- `inventory/hosts.yml`: list of switches and their management IPs
- `inventory/group_vars/cisco_switches/main.yml`: shared connection settings for Cisco IOS devices
- `inventory/group_vars/cisco_switches/vault.yml`: encrypted credentials file
- `playbooks/ping.yml`: verifies SSH access by running `show version`
- `playbooks/backup-config.yml`: backs up running configs
- `playbooks/vlans.yml`: creates or updates VLANs
- `playbooks/interfaces.yml`: applies interface settings
- `requirements.yml`: required Ansible collections
- `ansible.cfg`: local Ansible defaults for this repo

## Devices currently added

- `SW-CORE-01` `192.168.100.11` `Cisco Catalyst 9300-48P`
- `SW-DIST-01` `192.168.100.12` `Cisco Catalyst 3850-48T`
- `SW-ACC-01` `192.168.100.13` `Cisco Catalyst 2960X-48FPS-L`
- `SW-ACC-02` `192.168.100.14` `Cisco Catalyst 9200L-24P-4G`

## Step-by-step installation

### 1. Open the project folder

```bash
cd /home/alex/network-scripts/ansible-network
```

### 2. Activate the Python virtual environment

This repository already includes `ansible-venv`.

```bash
source ansible-venv/bin/activate
```

If you ever need to recreate it from scratch:

```bash
python3 -m venv ansible-venv
source ansible-venv/bin/activate
pip install --upgrade pip ansible
```

### 3. Install the required Ansible collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 4. Create the encrypted credentials file

```bash
cp inventory/group_vars/cisco_switches/vault.example.yml inventory/group_vars/cisco_switches/vault.yml
```

Edit `inventory/group_vars/cisco_switches/vault.yml` and replace the placeholder value:

```yaml
vault_ansible_password: "your-switch-ssh-password"
```

Then encrypt it:

```bash
ansible-vault encrypt inventory/group_vars/cisco_switches/vault.yml
```

### 5. Review the shared connection settings

Current shared settings are in `inventory/group_vars/cisco_switches/main.yml`:

- `ansible_connection: ansible.netcommon.network_cli`
- `ansible_network_os: cisco.ios.ios`
- `ansible_user: alex`
- `ansible_password: "{{ vault_ansible_password }}"`

If your switch username is not `alex`, update `ansible_user` before running playbooks.

### 6. Review the inventory

Open `inventory/hosts.yml` and confirm the switch names and `ansible_host` IP addresses are correct for your environment.

## How to run the project

Every time you work with this repo:

```bash
cd /home/alex/network-scripts/ansible-network
source ansible-venv/bin/activate
```

### 1. Test connectivity first

This is the safest first command. It connects to each switch and runs `show version`.

```bash
ansible-playbook playbooks/ping.yml --ask-vault-pass
```

### 2. Back up running configurations

Run this before making changes.

```bash
ansible-playbook playbooks/backup-config.yml --ask-vault-pass
```

Ansible will create backup files in its default backup location for `ios_config` runs.

### 3. Apply VLAN changes

Example:

```bash
ansible-playbook playbooks/vlans.yml --ask-vault-pass -e '{
  "vlan_definitions": [
    {"vlan_id": 10, "name": "USERS"},
    {"vlan_id": 20, "name": "VOICE"}
  ]
}'
```

What this does:

- ensures VLAN 10 exists and is named `USERS`
- ensures VLAN 20 exists and is named `VOICE`

### 4. Apply interface changes

Example:

```bash
ansible-playbook playbooks/interfaces.yml --ask-vault-pass -e '{
  "interface_configs": [
    {"name": "GigabitEthernet1/0/1", "description": "Office-PC-01", "enabled": true}
  ]
}'
```

What this does:

- updates interface `GigabitEthernet1/0/1`
- sets the description to `Office-PC-01`
- ensures the interface is enabled

## How to make changes safely

### Change device IPs or add more switches

Edit `inventory/hosts.yml`.

Example:

```yaml
all:
  children:
    cisco_switches:
      hosts:
        SW-NEW-01:
          ansible_host: 192.168.100.20
          device_model: Cisco Catalyst 9200
```

After editing the inventory, rerun:

```bash
ansible-playbook playbooks/ping.yml --ask-vault-pass
```

### Change the login user

Edit `inventory/group_vars/cisco_switches/main.yml`:

```yaml
ansible_user: your-username
```

### Change the SSH password

Edit the encrypted vault file:

```bash
ansible-vault edit inventory/group_vars/cisco_switches/vault.yml
```

### Change which VLANs are deployed

You have two common options:

1. Pass values with `-e` on the command line for one-off changes.
2. Store variables in a file later if you want repeatable change sets.

Current example for one-off VLAN changes:

```bash
ansible-playbook playbooks/vlans.yml --ask-vault-pass -e '{
  "vlan_definitions": [
    {"vlan_id": 30, "name": "SERVERS"},
    {"vlan_id": 40, "name": "GUEST"}
  ]
}'
```

### Change interface settings

Pass a new `interface_configs` structure:

```bash
ansible-playbook playbooks/interfaces.yml --ask-vault-pass -e '{
  "interface_configs": [
    {"name": "GigabitEthernet1/0/10", "description": "Printer", "enabled": true}
  ]
}'
```

## Recommended operating workflow

Use this order for routine changes:

1. Activate the virtual environment.
2. Confirm inventory and credentials are correct.
3. Run `playbooks/ping.yml`.
4. Run `playbooks/backup-config.yml`.
5. Run the change playbook such as `playbooks/vlans.yml` or `playbooks/interfaces.yml`.
6. Review the result output for failures or changed devices.

## Useful commands

Show the inventory Ansible sees:

```bash
ansible-inventory --graph
```

Show the resolved host variables:

```bash
ansible-inventory --host SW-CORE-01
```

Check the Ansible version in the virtual environment:

```bash
ansible --version
```

## Notes

- The inventory is currently set for direct privilege level 15 access, so no enable secret is required.
- `host_key_checking = False` is set in `ansible.cfg`, so SSH host key prompts are disabled for this repo.
- If a playbook reports authentication or timeout errors, check reachability, username, password, and whether SSH is enabled on the switch.
