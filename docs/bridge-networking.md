# Bridge Networking Mode

This document describes how to configure OpenShift clusters to use bridge networking instead of the default NAT mode.

## Overview

By default, this project creates a libvirt virtual network with NAT for VM connectivity. Bridge mode allows VMs to connect directly to an existing Linux bridge on the host, which is useful for:

- **Direct network access**: VMs appear as regular hosts on your physical network
- **External DHCP integration**: Use existing network infrastructure
- **Multiple network segments**: Connect VMs to specific VLANs or network segments
- **Advanced networking scenarios**: Custom routing, firewall rules, or network isolation

### Architecture Comparison

| Aspect | NAT Mode (default) | Bridge Mode |
|--------|-------------------|-------------|
| Network creation | Libvirt creates virtual network | Uses existing Linux bridge |
| DHCP/DNS | Libvirt's internal dnsmasq | System dnsmasq service |
| VM visibility | Hidden behind NAT | Directly on bridge network |
| External access | Via host port forwarding | Direct (if bridge is routed) |
| Multi-cluster | Automatic isolation | Requires unique subnets |

## Prerequisites

Before using bridge mode, ensure the following:

### 1. Linux Bridge Exists

The bridge interface must already exist on the host. Create it using NetworkManager or systemd-networkd.

**Using NetworkManager (recommended for RHEL/Rocky):**

```bash
# Create bridge interface
nmcli connection add type bridge con-name br0 ifname br0

# Configure IP address on the bridge (this will be the gateway for VMs)
nmcli connection modify br0 ipv4.addresses "192.168.50.1/24"
nmcli connection modify br0 ipv4.method manual

# Optional: Add a physical interface to the bridge
nmcli connection add type bridge-slave con-name br0-slave ifname eth1 master br0

# Bring up the bridge
nmcli connection up br0
```

**Verify the bridge:**

```bash
ip addr show br0
bridge link show
```

### 2. IP Forwarding Enabled

Ensure IP forwarding is enabled if VMs need external network access:

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl -p /etc/sysctl.d/99-ip-forward.conf
```

### 3. Firewall Configuration

The playbook automatically configures firewalld, but ensure the bridge interface is in a trusted zone or has appropriate rules.

## Configuration

### Basic Bridge Mode Setup

Add these settings to your `cluster.yml`:

```yaml
---
cluster_name: ocp4
public_domain: example.com

# Enable bridge mode
network_mode: bridge
bridge_interface: br0

# Subnet for this cluster (must be unique per cluster on the same bridge)
vn_subnet: 192.168.50.0

# Rest of your cluster configuration...
dns_provider: cloudflare
# ...
```

### Multi-Cluster Setup

When running multiple clusters on the same bridge, each cluster **must use a unique subnet** to avoid DHCP conflicts:

```yaml
# Cluster 1 (cluster1.yml)
cluster_name: cluster1
network_mode: bridge
bridge_interface: br0
vn_subnet: 192.168.50.0

# Cluster 2 (cluster2.yml)
cluster_name: cluster2
network_mode: bridge
bridge_interface: br0
vn_subnet: 192.168.51.0

# Cluster 3 (cluster3.yml)
cluster_name: cluster3
network_mode: bridge
bridge_interface: br0
vn_subnet: 192.168.52.0
```

The playbook includes automatic conflict detection and will fail with a clear error message if you attempt to use a subnet that's already in use by another cluster.

### Configuration Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `network_mode` | `nat` | Network mode: `nat` or `bridge` |
| `bridge_interface` | `br0` | Name of the existing Linux bridge |
| `vn_subnet` | `192.168.50.0` | Subnet for DHCP range (unique per cluster) |

## How It Works

When `network_mode: bridge` is set:

1. **VM Creation**: VMs are created with `<interface type='bridge'>` instead of `<interface type='network'>`, connecting directly to the specified bridge interface.

2. **DHCP/DNS**: A dnsmasq configuration is deployed to `/etc/dnsmasq.d/<cluster_name>.conf` providing:
   - DHCP with static reservations for all cluster nodes
   - DNS resolution for cluster endpoints (api, api-int, *.apps)
   - IPv6 support (DHCPv6 and RA) when enabled

3. **MAC Addresses**: Unique MAC addresses are generated using a hash of the cluster name combined with the subnet, ensuring uniqueness even when multiple clusters share the same subnet on different bridges.

4. **Cleanup**: When destroying a cluster, only that cluster's dnsmasq configuration is removed and dnsmasq is reloaded (not restarted), preserving DHCP leases for other clusters.

## Troubleshooting

### VMs Not Getting IP Addresses

1. **Check dnsmasq status:**
   ```bash
   systemctl status dnsmasq
   journalctl -u dnsmasq -f
   ```

2. **Verify configuration exists:**
   ```bash
   cat /etc/dnsmasq.d/<cluster_name>.conf
   ```

3. **Check bridge interface has IP:**
   ```bash
   ip addr show br0
   ```

4. **Test DHCP manually:**
   ```bash
   sudo tcpdump -i br0 port 67 or port 68
   ```

### DNS Not Resolving

1. **Verify dnsmasq is listening:**
   ```bash
   ss -ulnp | grep :53
   ```

2. **Test DNS resolution:**
   ```bash
   dig @192.168.50.1 api.ocp4.example.com
   ```

### DHCP Conflict Error

If you see a "DHCP range conflict detected" error:

1. **List existing cluster configs:**
   ```bash
   ls -la /etc/dnsmasq.d/*.conf
   grep dhcp-range /etc/dnsmasq.d/*.conf
   ```

2. **Choose a unique subnet** that doesn't overlap with existing clusters.

### Firewall Issues

1. **Check firewalld zones:**
   ```bash
   firewall-cmd --get-active-zones
   firewall-cmd --zone=trusted --list-all
   ```

2. **Manually open ports if needed:**
   ```bash
   firewall-cmd --zone=trusted --add-interface=br0 --permanent
   firewall-cmd --zone=trusted --add-port=53/udp --permanent
   firewall-cmd --zone=trusted --add-port=67/udp --permanent
   firewall-cmd --reload
   ```

## Reverting to NAT Mode

To switch back to NAT mode, simply remove or comment out the bridge settings:

```yaml
# network_mode: bridge
# bridge_interface: br0
```

Or explicitly set:

```yaml
network_mode: nat
```

The default NAT mode will use libvirt's built-in virtual networking with automatic DHCP and DNS.
