# Inventory Examples

This directory contains example inventory files for different HCD cluster configurations.

## Available Examples

### 1. Single Datacenter (3 Nodes)
**File:** `single-dc-3-nodes.ini`

A simple setup with one datacenter and three nodes. Ideal for:
- Development environments
- Small production clusters
- Testing and learning

**Configuration:**
- 1 datacenter (dc1)
- 3 nodes total
- 2 racks for rack awareness
- 1 seed node

### 2. Multi-Datacenter (6 Nodes)
**File:** `multi-dc-6-nodes.ini`

A multi-datacenter setup with 3 nodes per datacenter. Ideal for:
- Production environments requiring high availability
- Geographic distribution
- Disaster recovery scenarios

**Configuration:**
- 2 datacenters (dc1, dc2)
- 6 nodes total (3 per DC)
- 2 racks per datacenter
- 2 seed nodes (1 per DC)

## Inventory Format

Each inventory file follows this format:

```ini
[hcd_nodes]
public_ip private_ip=<internal_ip> rack=<rack_name> dc=<datacenter_name> seed=<true|false>

[dc_name]
public_ip1
public_ip2
```

### Required Variables

- **public_ip**: The public IP address or hostname (used as inventory_hostname for SSH and broadcast_address)
- **private_ip**: The private/internal IP address (used for listen_address and rpc_address)
- **rack**: Rack identifier for rack-aware replication
- **dc**: Datacenter identifier
- **seed**: Whether this node is a seed node (true/false)

### Datacenter Groups

Each datacenter should have its own group (e.g., `[dc1]`, `[dc2]`) containing the public IPs of nodes in that datacenter.

## Using These Examples

1. **Copy an example to your inventory directory:**
   ```bash
   cp inventory/examples/single-dc-3-nodes.ini inventory/hosts
   ```

2. **Edit the inventory file with your actual IPs:**
   ```bash
   vim inventory/hosts
   ```

3. **Update group_vars as needed:**
   - `inventory/group_vars/all.yml` - Global settings
   - `inventory/group_vars/hcd_nodes.yml` - HCD cluster settings
   - `inventory/group_vars/dc1.yml` - DC1-specific settings
   - `inventory/group_vars/dc2.yml` - DC2-specific settings

4. **Run the installation playbook:**
   ```bash
   ansible-playbook -i inventory/hosts playbooks/install_cluster.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
   ```

## Best Practices

### Seed Nodes
- Use at least 1 seed node per datacenter
- Typically 1-3 seed nodes per datacenter is sufficient
- Seed nodes should be stable and not frequently restarted

### Rack Awareness
- Distribute nodes across multiple racks for better fault tolerance
- Use rack names that reflect your actual infrastructure (e.g., rack1, rack2, us-east-1a, us-east-1b)

### Network Configuration
- **public_ip**: Should be accessible from your Ansible control machine
- **private_ip**: Should be accessible from all cluster nodes
- Ensure proper security group/firewall rules are in place

### Datacenter Naming
- Use meaningful datacenter names (e.g., us-east, us-west, eu-central)
- Keep names consistent across inventory and group_vars

## Testing Your Inventory

Before running the full installation, test your inventory:

```bash
# Test connectivity
ansible -i inventory/hosts hcd_nodes -m ping --key-file PRIVATE_KEY_FILE_PATH --user USERNAME

# Check gathered facts
ansible -i inventory/hosts hcd_nodes -m setup --key-file PRIVATE_KEY_FILE_PATH --user USERNAME

# Verify seed node configuration
ansible -i inventory/hosts hcd_nodes -m debug -a "var=seed" --key-file PRIVATE_KEY_FILE_PATH --user USERNAME