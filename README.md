# Ansible network automation

This repository is an Ansible project for managing Cisco switches over SSH. It is not a web application or long-running service. You "run the application" by executing Ansible playbooks against the devices in the inventory.

## What this project contains

- `inventory/hosts.yml`: list of switches and their management IPs
- `inventory/group_vars/cisco_switches/main.yml`: shared connection settings for Cisco IOS devices
- `inventory/group_vars/cisco_switches/vault.yml`: encrypted credentials file
- `playbooks/ping.yml`: verifies SSH access by running `show version`
- `playbooks/backup-config.yml`: backs up running configs
- `playbooks/save-config.yml`: saves running config to startup config
- `playbooks/vlans.yml`: creates or updates VLANs
- `playbooks/interfaces.yml`: applies interface settings
- `playbooks/interface-switchport.yml`: manages access and trunk port settings
- `playbooks/port-channels.yml`: manages EtherChannel and port-channel membership
- `playbooks/baseline.yml`: applies standard shared switch configuration
- `playbooks/users.yml`: manages local switch users
- `playbooks/facts.yml`: gathers operational state with read-only show commands
- `playbooks/monitoring.yml`: configures syslog, NTP, and SNMP
- `playbooks/aaa.yml`: applies AAA and TACACS/RADIUS related config lines
- `playbooks/compliance.yml`: gathers compliance and platform audit output
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
git clone https://github.com/alexbugheanu0/ansible-network.git
cd ansible-network
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
cd ansible-network
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

### 5. Apply switchport mode, VLAN, and trunk settings

Example:

```bash
ansible-playbook playbooks/interface-switchport.yml --ask-vault-pass -e '{
  "switchport_configs": [
    {
      "name": "GigabitEthernet1/0/10",
      "mode": "access",
      "access": {
        "vlan": 30
      }
    },
    {
      "name": "GigabitEthernet1/0/48",
      "mode": "trunk",
      "trunk": {
        "native_vlan": 99,
        "allowed_vlans": "10,20,30,99"
      }
    }
  ]
}'
```

### 6. Apply a baseline configuration

Example:

```bash
ansible-playbook playbooks/baseline.yml --ask-vault-pass -e '{
  "baseline_config_lines": [
    "service timestamps debug datetime msec",
    "service timestamps log datetime msec",
    "no ip http server",
    "no ip http secure-server",
    "ip domain-name lab.local",
    "ntp server 192.168.100.5"
  ]
}'
```

### 7. Manage local users

Example:

```bash
ansible-playbook playbooks/users.yml --ask-vault-pass -e '{
  "local_users": [
    {
      "name": "netadmin",
      "configured_password": "StrongPassword123!",
      "privilege": 15,
      "state": "present"
    }
  ]
}'
```

### 8. Collect operational facts

This is a read-only audit playbook for common troubleshooting output.

```bash
ansible-playbook playbooks/facts.yml --ask-vault-pass
```

### 9. Save the running configuration

Run this after successful changes so they survive a reboot.

```bash
ansible-playbook playbooks/save-config.yml --ask-vault-pass
```

### 10. Configure port-channels

Example:

```bash
ansible-playbook playbooks/port-channels.yml --ask-vault-pass -e '{
  "port_channel_configs": [
    {
      "name": "Port-channel10",
      "members": [
        {"member": "GigabitEthernet1/0/47", "mode": "active"},
        {"member": "GigabitEthernet1/0/48", "mode": "active"}
      ]
    }
  ]
}'
```

### 11. Configure syslog, NTP, and SNMP

Example:

```bash
ansible-playbook playbooks/monitoring.yml --ask-vault-pass -e '{
  "logging_global_config": {
    "hosts": [
      {"hostname": "192.168.100.50"}
    ],
    "trap": "informational",
    "logging_on": "enable"
  },
  "ntp_global_config": {
    "servers": [
      {"server": "192.168.100.5"}
    ],
    "logging": true
  },
  "snmp_server_config": {
    "contact": "Network Team",
    "location": "Main MDF",
    "communities": [
      {"name": "readonly", "ro": true}
    ]
  }
}'
```

### 12. Apply AAA and TACACS settings

Use variable-driven config lines so you can match your IOS syntax and authentication design.

```bash
ansible-playbook playbooks/aaa.yml --ask-vault-pass -e '{
  "aaa_config_lines": [
    "aaa new-model",
    "tacacs server ISE-1",
    " address ipv4 192.168.100.60",
    " key 7 070C285F4D06",
    "aaa group server tacacs+ TACACS-SERVERS",
    " server name ISE-1",
    "aaa authentication login default group TACACS-SERVERS local",
    "aaa authorization exec default group TACACS-SERVERS local if-authenticated"
  ]
}'
```

### 13. Run a compliance audit

This is a read-only playbook for software version, license, inventory, VLAN, EtherChannel, and neighbor checks.

```bash
ansible-playbook playbooks/compliance.yml --ask-vault-pass
```

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

### Change switchport mode and VLAN assignments

Pass a `switchport_configs` structure to `playbooks/interface-switchport.yml`:

```bash
ansible-playbook playbooks/interface-switchport.yml --ask-vault-pass -e '{
  "switchport_configs": [
    {
      "name": "GigabitEthernet1/0/12",
      "mode": "access",
      "access": {
        "vlan": 40
      }
    }
  ]
}'
```

### Apply common baseline settings across every switch

Pass a list of config lines to `playbooks/baseline.yml`:

```bash
ansible-playbook playbooks/baseline.yml --ask-vault-pass -e '{
  "baseline_config_lines": [
    "logging host 192.168.100.50",
    "snmp-server community readonly RO"
  ]
}'
```

### Add, change, or remove local users

Use `playbooks/users.yml` with a `local_users` list:

```bash
ansible-playbook playbooks/users.yml --ask-vault-pass -e '{
  "local_users": [
    {
      "name": "opsuser",
      "configured_password": "AnotherStrongPassword123!",
      "privilege": 15,
      "state": "present"
    }
  ]
}'
```

### Change EtherChannel members

Use `playbooks/port-channels.yml` with a `port_channel_configs` list:

```bash
ansible-playbook playbooks/port-channels.yml --ask-vault-pass -e '{
  "port_channel_configs": [
    {
      "name": "Port-channel20",
      "members": [
        {"member": "GigabitEthernet1/0/23", "mode": "active"},
        {"member": "GigabitEthernet1/0/24", "mode": "active"}
      ]
    }
  ]
}'
```

### Change syslog, NTP, or SNMP settings

Use `playbooks/monitoring.yml` and pass only the sections you want to change:

```bash
ansible-playbook playbooks/monitoring.yml --ask-vault-pass -e '{
  "logging_global_config": {
    "hosts": [
      {"hostname": "192.168.100.51"}
    ]
  }
}'
```

### Change AAA or TACACS configuration

Use `playbooks/aaa.yml` with an explicit config line list:

```bash
ansible-playbook playbooks/aaa.yml --ask-vault-pass -e '{
  "aaa_config_lines": [
    "aaa authentication login default local"
  ]
}'
```

## Recommended operating workflow

Use this order for routine changes:

1. Activate the virtual environment.
2. Confirm inventory and credentials are correct.
3. Run `playbooks/ping.yml`.
4. Run `playbooks/backup-config.yml`.
5. Run the change playbook such as `playbooks/vlans.yml`, `playbooks/interfaces.yml`, `playbooks/interface-switchport.yml`, `playbooks/port-channels.yml`, `playbooks/baseline.yml`, `playbooks/users.yml`, `playbooks/monitoring.yml`, or `playbooks/aaa.yml`.
6. Run `playbooks/save-config.yml`.
7. Run `playbooks/compliance.yml` or `playbooks/facts.yml` to verify the final state.
8. Review the result output for failures or changed devices.

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
