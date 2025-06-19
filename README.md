# Ansible HA Firewall Role

This role configures a High Availability (HA) firewall cluster using keepalived for VIP management, conntrackd for connection state synchronization, and iptables/nftables for packet filtering.

## Features

- **Dual VIP Setup**: External and Internal Virtual IPs with automatic failover
- **State Synchronization**: conntrackd keeps connection states in sync between nodes
- **Flexible Backend**: Supports both iptables and nftables

## Quick Start

### 1. Inventory Configuration
The role is hardcoded to use the names fw1 and fw2. If you rename these, it will break.
```ini
[firewalls]
fw1 ansible_host=192.168.0.117 external_ip=192.168.0.117 external_peer=192.168.0.243 external_vip=192.168.0.203 internal_ip=10.10.0.2 internal_peer=10.10.0.3 internal_vip=10.10.0.1
fw2 ansible_host=192.168.0.243 external_ip=192.168.0.243 external_peer=192.168.0.117 external_vip=192.168.0.203 internal_ip=10.10.0.3 internal_peer=10.10.0.2 internal_vip=10.10.0.1

[firewalls:vars]
ext_interface=ens18
int_interface=ens19
firewall_backend=iptables
```

### 2. Playbook

```yaml
- hosts: firewalls
  become: yes
  roles:
    - role: role-iptables-ha
```

### 3. Deploy

```bash
ansible-playbook -i inventory.ini your_playbook.yml --ask-become-pass
```

## Monitoring HA Status

Use this command to monitor the HA cluster status in real-time:

```bash
watch -n1 "ip -4 -brief addr show ens18 ; \
           journalctl -u keepalived -n 1 --no-pager ; \
           conntrackd -s network | head -4"
```

This shows:
- Current IP addresses on external interface (including VIP when active)
- Latest keepalived log entry (state changes, failovers)
- conntrackd network synchronization status

## Architecture

### Network Layout
- **External Network**: 192.168.0.0/24 (ens18)
  - fw1: 192.168.0.117
  - fw2: 192.168.0.243
  - VIP: 192.168.0.203
- **Internal Network**: 10.10.0.0/24 (ens19)
  - fw1: 10.10.0.2
  - fw2: 10.10.0.3
  - VIP: 10.10.0.1

           ┌────────────────── 192.168.0.0/24 (external LAN / vmbr0) ───────────────────┐
           │                                                                            │
           │  Admin PC / clients                     ↑  VRRP VIP 192.168.0.203          │
           │  192.168.0.100 …                       (what users SSH to)                 │
           └────────────────────────────┬───────────────────────────────────────────────┘
                                        │             DNAT :22 → 10.10.0.86
                                        ▼
                          +-----------------------------+
                          |  FW-1   (primary when up)   |
                          |                             |
          ext uplink ───► |  ens18  192.168.0.117       |
                          |  ens19   10.10.0.2          |
                          |  keepalived  (priority 101) |
                          |  conntrackd   (send state)  |
                          +--------------┬--------------+
                                         │  UDP 3780 state-sync
                                         │
                          +--------------┴--------------+
                          |  FW-2   (secondary)         |
                          |                             |
          ext uplink ───► |  ens18  192.168.0.243       |
                          |  ens19   10.10.0.3          |
                          |  keepalived  (priority 100) |
                          |  conntrackd  (recv state)   |
                          +--------------┬--------------+
                                         │
           ┌─────────────────────────────┴─────────────────────────────────┐
           │          10.10.0.0/24  internal / vmbr1                       │
           │                                                               │
           │     Server VM (SSH target) 10.10.0.86  ← default-gw 10.10.0.1 │
           └───────────────────────────────────────────────────────────────┘


### Services
- **keepalived**: Manages VIP failover using VRRP (unicast mode)
- **conntrackd**: Synchronizes connection states between nodes
- **iptables/nftables**: Packet filtering and NAT

## Variable Reference

### Required Variables (per host)
- `external_ip`: Node's external IP address
- `external_peer`: Peer node's external IP address
- `external_vip`: External Virtual IP address
- `internal_ip`: Node's internal IP address
- `internal_peer`: Peer node's internal IP address
- `internal_vip`: Internal Virtual IP address

### Global Variables
- `ext_interface`: External network interface name
- `int_interface`: Internal network interface name

### Backends Variable
- `firewall_backend`: Accepts either iptables or nftables

## Firewall Rules

Rules are defined in `vars/iptables_rules/`:
- `input.yml`: INPUT chain rules
- `output.yml`: OUTPUT chain rules
- `forward.yml`: FORWARD chain rules
- `nat_prerouting.yml`: NAT PREROUTING rules
- `nat_postrouting.yml`: NAT POSTROUTING rules

## Troubleshooting

### Check Service Status
```bash
systemctl status keepalived conntrackd iptables
```

### View keepalived State
```bash
journalctl -u keepalived -f
```

### Check conntrackd Sync
```bash
conntrackd -s network
conntrackd -s cache
```

### Verify VIP Assignment
```bash
ip addr show | grep -E "(ens18|ens19)"
```

### Test Failover
```bash
# On MASTER node
systemctl stop keepalived
# Watch VIP move to BACKUP node
```