# Ansible Network Automation Guide

This guide explains how to set up the repository, what each playbook does, and how to run playbooks safely against your Cisco switches.

## 1. What this project is

This repository is an Ansible-based network automation project for Cisco IOS switches.

It is used to:

- connect to switches over SSH
- audit switch state
- apply repeatable configuration changes
- back up and restore configurations
- avoid manual device-by-device CLI work

It is not a daemon or web app. You use it by running playbooks from the terminal.

## 2. Project structure

- `inventory/hosts.yml`
  Purpose: list of switches and their management IP addresses

- `inventory/group_vars/cisco_switches/main.yml`
  Purpose: shared connection settings for all Cisco switches

- `inventory/group_vars/cisco_switches/vault.yml`
  Purpose: encrypted password file

- `playbooks/`
  Purpose: operational tasks such as backup, VLANs, interfaces, users, monitoring, ACLs, routes, restore, and audits

- `requirements.yml`
  Purpose: required Ansible collections

- `ansible.cfg`
  Purpose: repo-local Ansible defaults

## 3. Prerequisites

Before using the repo, make sure you have:

- Python 3 installed
- SSH reachability to the switches
- a valid switch username
- the switch SSH password
- privilege level 15 access or equivalent permissions

## 4. Initial setup

### Step 1: Clone the repository

```bash
git clone https://github.com/alexbugheanu0/ansible-network.git
cd ansible-network
```

### Step 2: Activate the virtual environment

This repository already includes `ansible-venv`.

```bash
source ansible-venv/bin/activate
```

If you need to recreate it:

```bash
python3 -m venv ansible-venv
source ansible-venv/bin/activate
pip install --upgrade pip ansible
```

### Step 3: Install required Ansible collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### Step 4: Create the encrypted credentials file

```bash
cp inventory/group_vars/cisco_switches/vault.example.yml inventory/group_vars/cisco_switches/vault.yml
```

Edit the file and replace the placeholder:

```yaml
vault_ansible_password: "your-switch-password"
```

Encrypt it:

```bash
ansible-vault encrypt inventory/group_vars/cisco_switches/vault.yml
```

### Step 5: Review connection settings

Current shared settings:

```yaml
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: cisco.ios.ios
ansible_user: alex
ansible_password: "{{ vault_ansible_password }}"
```

If your username is different, update:

`inventory/group_vars/cisco_switches/main.yml`

### Step 6: Review inventory

Open `inventory/hosts.yml` and verify:

- switch names
- management IPs
- device models

## 5. Normal operating workflow

Use this order for most changes:

1. Activate the virtual environment.
2. Verify inventory and credentials.
3. Run `playbooks/ping.yml`.
4. Run `playbooks/backup-config.yml`.
5. Run the intended change playbook.
6. Run `playbooks/save-config.yml`.
7. Run `playbooks/compliance.yml` or `playbooks/facts.yml`.

## 6. How to run playbooks

Each session normally starts like this:

```bash
cd ansible-network
source ansible-venv/bin/activate
```

Most playbooks are run like this:

```bash
ansible-playbook playbooks/<playbook-name>.yml --ask-vault-pass
```

If a playbook needs variables, pass them with `-e`:

```bash
ansible-playbook playbooks/<playbook-name>.yml --ask-vault-pass -e '{
  "some_variable": "some_value"
}'
```

## 7. Playbook reference

### `playbooks/ping.yml`

What it does:

- tests SSH access
- confirms Ansible can connect to the switches
- runs `show version`

When to use it:

- before any change
- after updating inventory
- after changing credentials

Run:

```bash
ansible-playbook playbooks/ping.yml --ask-vault-pass
```

### `playbooks/backup-config.yml`

What it does:

- backs up the running configuration from each switch

When to use it:

- before all configuration changes
- before maintenance windows

Run:

```bash
ansible-playbook playbooks/backup-config.yml --ask-vault-pass
```

### `playbooks/save-config.yml`

What it does:

- saves running-config to startup-config when changes were made

When to use it:

- after successful change playbooks

Run:

```bash
ansible-playbook playbooks/save-config.yml --ask-vault-pass
```

### `playbooks/vlans.yml`

What it does:

- creates VLANs
- updates VLAN names

Variable used:

- `vlan_definitions`

Run:

```bash
ansible-playbook playbooks/vlans.yml --ask-vault-pass -e '{
  "vlan_definitions": [
    {"vlan_id": 10, "name": "USERS"},
    {"vlan_id": 20, "name": "VOICE"}
  ]
}'
```

### `playbooks/interfaces.yml`

What it does:

- changes interface descriptions
- enables or disables interfaces

Variable used:

- `interface_configs`

Run:

```bash
ansible-playbook playbooks/interfaces.yml --ask-vault-pass -e '{
  "interface_configs": [
    {"name": "GigabitEthernet1/0/1", "description": "Office-PC-01", "enabled": true}
  ]
}'
```

### `playbooks/interface-switchport.yml`

What it does:

- sets access or trunk mode
- assigns access VLANs
- sets native VLANs
- sets allowed VLANs on trunks

Variable used:

- `switchport_configs`

Run:

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

### `playbooks/port-channels.yml`

What it does:

- creates or updates EtherChannel membership
- applies LACP or channel-group modes

Variable used:

- `port_channel_configs`

Run:

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

### `playbooks/baseline.yml`

What it does:

- applies generic shared configuration lines
- useful for timestamps, domain-name, SSH hardening, syslog basics, banners, and platform defaults

Variable used:

- `baseline_config_lines`

Run:

```bash
ansible-playbook playbooks/baseline.yml --ask-vault-pass -e '{
  "baseline_config_lines": [
    "service timestamps debug datetime msec",
    "service timestamps log datetime msec",
    "no ip http server",
    "no ip http secure-server",
    "ip domain-name lab.local"
  ]
}'
```

### `playbooks/users.yml`

What it does:

- creates local users
- updates local users

Variable used:

- `local_users`

Run:

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

### `playbooks/monitoring.yml`

What it does:

- configures syslog
- configures NTP
- configures SNMP

Variables used:

- `logging_global_config`
- `ntp_global_config`
- `snmp_server_config`

Run:

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

### `playbooks/aaa.yml`

What it does:

- applies AAA, TACACS, or RADIUS related IOS config lines

Variable used:

- `aaa_config_lines`

Run:

```bash
ansible-playbook playbooks/aaa.yml --ask-vault-pass -e '{
  "aaa_config_lines": [
    "aaa new-model",
    "tacacs server ISE-1",
    " address ipv4 192.168.100.60",
    " key 7 070C285F4D06",
    "aaa group server tacacs+ TACACS-SERVERS",
    " server name ISE-1",
    "aaa authentication login default group TACACS-SERVERS local"
  ]
}'
```

### `playbooks/acls.yml`

What it does:

- creates or updates ACLs
- supports standard and extended ACL models

Variable used:

- `acl_definitions`

Run:

```bash
ansible-playbook playbooks/acls.yml --ask-vault-pass -e '{
  "acl_definitions": [
    {
      "afi": "ipv4",
      "acls": [
        {
          "name": "MGMT_ONLY",
          "acl_type": "extended",
          "aces": [
            {
              "sequence": 10,
              "grant": "permit",
              "protocol": "ip",
              "source": {
                "address": "192.168.100.0",
                "wildcard_bits": "0.0.0.255"
              },
              "destination": {
                "any": true
              }
            }
          ]
        }
      ]
    }
  ]
}'
```

### `playbooks/management-svi.yml`

What it does:

- creates or updates management VLAN interfaces
- sets descriptions and admin state
- assigns Layer 3 IP addresses

Variables used:

- `management_svi_interfaces`
- `management_svi_l3_configs`

Run:

```bash
ansible-playbook playbooks/management-svi.yml --ask-vault-pass -e '{
  "management_svi_interfaces": [
    {"name": "Vlan100", "description": "Management VLAN", "enabled": true}
  ],
  "management_svi_l3_configs": [
    {
      "name": "Vlan100",
      "ipv4": [
        {"address": "192.168.100.2/24"}
      ]
    }
  ]
}'
```

### `playbooks/static-routes.yml`

What it does:

- configures IPv4 or IPv6 static routes

Variable used:

- `static_route_configs`

Run:

```bash
ansible-playbook playbooks/static-routes.yml --ask-vault-pass -e '{
  "static_route_configs": [
    {
      "address_families": [
        {
          "afi": "ipv4",
          "routes": [
            {
              "dest": "0.0.0.0/0",
              "next_hops": [
                {"forward_router_address": "192.168.100.1"}
              ]
            }
          ]
        }
      ]
    }
  ]
}'
```

### `playbooks/facts.yml`

What it does:

- collects common operational show output
- useful for troubleshooting and verification

Run:

```bash
ansible-playbook playbooks/facts.yml --ask-vault-pass
```

### `playbooks/compliance.yml`

What it does:

- collects version, license, inventory, VLAN, EtherChannel, and neighbor information
- useful for audits and post-change validation

Run:

```bash
ansible-playbook playbooks/compliance.yml --ask-vault-pass
```

### `playbooks/rollback-restore.yml`

What it does:

- restores a saved config file back to the switch
- saves the restored result if changes occur

Variable used:

- `restore_config_file`

Run:

```bash
ansible-playbook playbooks/rollback-restore.yml --ask-vault-pass -e '{
  "restore_config_file": "backup/SW-CORE-01_config.cfg"
}'
```

Important:

- use only known-good restore files
- treat this as a high-impact operation

### `playbooks/stack-members.yml`

What it does:

- shows stack member state
- helps you inspect current switch numbering and stack details

Run:

```bash
ansible-playbook playbooks/stack-members.yml --ask-vault-pass
```

### `playbooks/stack-renumber.yml`

What it does:

- sends explicit renumber commands to switch stacks
- can optionally trigger reload

Variables used:

- `confirm_stack_renumber`
- `stack_renumber_commands`
- `stack_reload_after_renumber`

Run:

```bash
ansible-playbook playbooks/stack-renumber.yml --ask-vault-pass -e '{
  "confirm_stack_renumber": true,
  "stack_renumber_commands": [
    "switch 2 renumber 1"
  ],
  "stack_reload_after_renumber": true
}'
```

Important:

- use only in a maintenance window
- verify current stack state first with `playbooks/stack-members.yml`

### `playbooks/site-baseline.yml`

What it does:

- runs a larger workflow in sequence
- backup
- baseline changes
- VLAN and interface changes
- routing and monitoring changes
- user and ACL changes
- save config
- compliance verification

When to use it:

- when you want one orchestrated run instead of multiple separate playbook commands

Run:

```bash
ansible-playbook playbooks/site-baseline.yml --ask-vault-pass
```

Note:

- only pass variables that you actually intend to apply in that run

## 8. Common change examples

### Add a new switch to inventory

Edit `inventory/hosts.yml`:

```yaml
all:
  children:
    cisco_switches:
      hosts:
        SW-NEW-01:
          ansible_host: 192.168.100.20
          device_model: Cisco Catalyst 9200
```

Then verify:

```bash
ansible-playbook playbooks/ping.yml --ask-vault-pass
```

### Change the switch login username

Edit:

`inventory/group_vars/cisco_switches/main.yml`

Example:

```yaml
ansible_user: your-username
```

### Change the switch password

Use:

```bash
ansible-vault edit inventory/group_vars/cisco_switches/vault.yml
```

## 9. Safe usage rules

- Always run `ping.yml` before changes.
- Always run `backup-config.yml` before changes.
- Save after successful changes with `save-config.yml`.
- Use `facts.yml` or `compliance.yml` after changes.
- Be careful with `rollback-restore.yml`.
- Be careful with `stack-renumber.yml`.
- Test on one device or a lab switch first when possible.

## 10. Useful commands

Show inventory graph:

```bash
ansible-inventory --graph
```

Show resolved host variables:

```bash
ansible-inventory --host SW-CORE-01
```

Check Ansible version:

```bash
ansible --version
```

Syntax-check a playbook:

```bash
ansible-playbook playbooks/vlans.yml --syntax-check
```

## 11. Troubleshooting

### Authentication failure

Check:

- `ansible_user`
- vault password contents
- switch SSH access
- privilege level

### Timeout or unreachable

Check:

- switch IP in `inventory/hosts.yml`
- network reachability
- firewall rules
- SSH enabled on the device

### A playbook changes nothing

Check:

- whether required variables were passed with `-e`
- whether your desired config already exists
- whether the interface or VLAN names are correct

### Restore or stack operation feels risky

Do not run it blind. Back up first, validate the target device, and use a maintenance window.
