# Ansible Role: iptables-ha

This Ansible role sets up iptables management with High Availability using Keepalived on Debian 12 systems. It includes simple, effective auditing of iptables command changes.

## Requirements

- Debian 12
- Ansible 2.9 or higher
- Two or more nodes for HA setup

## Role Variables

All variables can be overridden in your inventory or playbook:

```yaml
# Keepalived configuration
keepalived_state: MASTER  # or BACKUP
keepalived_priority: 100  # Higher for MASTER, lower for BACKUP
keepalived_vrrp_interface: ens18
keepalived_virtual_router_id: 51
keepalived_vip: 192.168.0.203
keepalived_auth_pass: "your_secure_password"
```

### Firewall Rule Structure

Firewall rules are now split by chain for clarity and maintainability. Each rule can include a `comment` field for documentation. You can also specify `source`, `destination`, `in_interface`, and `out_interface` for granular control.

#### Example: vars/iptables_rules/input.yml
```yaml
---
input_rules:
  # Allow loopback traffic
  - in_interface: lo
    jump: ACCEPT
    comment: "Allow all loopback (lo) traffic"

  # Allow SSH from a specific source
  - protocol: tcp
    destination_port: 22
    source: 192.168.0.254/24
    jump: ACCEPT
    comment: "Allow SSH from 192.168.0.254/24"

  # Allow HTTP from a specific source to a specific destination
  - protocol: tcp
    destination_port: 80
    source: 10.0.0.0/8
    destination: 192.168.0.10
    jump: ACCEPT
    comment: "Allow HTTP from 10.0.0.0/8 to 192.168.0.10"

  # Default deny
  - jump: DROP
    comment: "Default deny all other traffic"
```

#### Example: vars/iptables_rules/output.yml
```yaml
---
output_rules:
  # Allow outbound DNS only from eth0
  - protocol: udp
    destination_port: 53
    out_interface: eth0
    jump: ACCEPT
    comment: "Allow outbound DNS (UDP) from eth0"
```

#### Example: vars/iptables_rules/forward.yml
```yaml
---
forward_rules:
  # Allow forwarding for a specific subnet
  - source: 10.0.0.0/8
    destination: 192.168.0.0/24
    jump: ACCEPT
    comment: "Allow forwarding from 10.0.0.0/8 to 192.168.0.0/24"
  # Default deny
  - jump: DROP
    comment: "Default deny all other forwarded traffic"
```

### Adding Rules
- Add rules to the appropriate file (`input.yml`, `output.yml`, `forward.yml`).
- Each rule can have a `comment` for documentation.
- You can use `source`, `destination`, `in_interface`, and `out_interface` for granular control.
- The last rule in `input.yml` and `forward.yml` should always be a DROP rule for default deny.

## Example Inventory

```ini
[firewall_nodes]
node1 ansible_host=192.168.0.201 keepalived_state=MASTER keepalived_priority=100
node2 ansible_host=192.168.0.202 keepalived_state=BACKUP keepalived_priority=90

[firewall_nodes:vars]
ansible_user=debian
ansible_become=yes
keepalived_vip=192.168.0.203
keepalived_vrrp_interface=ens18
keepalived_virtual_router_id=51
```

## Example Playbook

```yaml
- hosts: firewall_nodes
  become: yes
  roles:
    - role: role-iptables-ha
```

## Features
- **Atomic rule application:** All rules are applied at once using `iptables-restore` for safety and consistency.
- **Idempotent:** Rules are always applied in a consistent order; default deny is enforced by a final DROP rule.
- **Per-rule comments:** Each rule can be documented for clarity and auditing.
- **Supports `source`, `destination`, `in_interface`, and `out_interface` fields for granular control.**
- **Auditing:** All iptables changes are logged using `auditd`.
- **Supports INPUT, OUTPUT, and FORWARD chains.**

## Auditing

The role includes simple auditing of iptables command changes using `auditd`.

- **To see who ran which iptables command:**
  ```bash
  ausearch -k iptables_cmd | grep proctitle
  ```
  To decode the command line from the `proctitle` field (if needed):
  ```bash
  echo <hex_from_proctitle> | xxd -r -p
  ```
- **To see direct edits to iptables rules files:**
  ```bash
  ausearch -k iptables_changes
  ```