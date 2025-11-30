# HCD Cluster Management - Ansible Architecture

## Overview

This Ansible project automates the deployment and management of HyperConverged Database (HCD) clusters with support for:
- Multi-datacenter, multi-node deployments
- OS-agnostic operations (Ubuntu/Debian and RHEL/CentOS)
- Flexible configuration via variables
- Monitoring (MCAC), Repairs (Reaper), and Backups (Medusa)

## Project Structure

```
hcd-cluster-mgmt/
├── ansible.cfg                          # Ansible configuration
├── inventory/
│   ├── hosts                           # Main inventory file
│   └── group_vars/
│       ├── all.yml                     # Global variables
│       ├── hcd_nodes.yml               # HCD-specific variables
│       ├── dc1.yml                     # DC1-specific variables
│       ├── dc2.yml                     # DC2-specific variables
│       └── monitoring.yml              # Monitoring tools variables
├── roles/
│   ├── system_setup/                   # System prerequisites & recommended settings
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── debian.yml
│   │   │   └── redhat.yml
│   │   ├── defaults/main.yml
│   │   └── templates/
│   │       └── limits.conf.j2
│   ├── java/                           # Java installation
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── debian.yml
│   │   │   └── redhat.yml
│   │   └── defaults/main.yml
│   ├── hcd_install/                    # HCD installation
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install.yml
│   │   │   ├── user.yml
│   │   │   └── directories.yml
│   │   ├── defaults/main.yml
│   │   └── handlers/main.yml
│   ├── hcd_config/                     # HCD configuration
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── cassandra_yaml.yml
│   │   │   ├── jvm_options.yml
│   │   │   ├── logback.yml
│   │   │   └── rackdc.yml
│   │   ├── defaults/main.yml
│   │   └── handlers/main.yml
│   ├── mcac/                           # Metric Collector for Apache Cassandra
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install.yml
│   │   │   └── configure.yml
│   │   └── defaults/main.yml
│   ├── reaper/                         # Cassandra Reaper
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install.yml
│   │   │   └── configure.yml
│   │   └── defaults/main.yml
│   └── medusa/                         # Medusa backup/restore
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── install.yml
│       │   └── configure.yml
│       └── defaults/main.yml
├── playbooks/
│   ├── install_cluster.yml             # Full cluster installation
│   ├── configure_cluster.yml           # Configuration only
│   ├── start_cluster.yml               # Start HCD services
│   ├── stop_cluster.yml                # Stop HCD services
│   ├── rolling_restart.yml             # Rolling restart
│   ├── install_monitoring.yml          # Install MCAC
│   ├── install_reaper.yml              # Install Reaper
│   └── install_medusa.yml              # Install Medusa
├── hcd-1.2.4/                          # HCD tarball (gitignored)
└── README.md                           # Project documentation
```

## Design Principles

### 1. Variable-Driven Configuration
All configurations are driven by variables defined in inventory files, allowing users to customize:
- Cluster topology (DCs, racks, nodes)
- Resource allocation (heap size, concurrent operations)
- Network settings (IPs, ports, broadcast addresses)
- Security settings (authentication, encryption)
- Storage paths (data, commitlog, hints, etc.)

### 2. OS-Agnostic Implementation
Each role uses conditional tasks based on `ansible_facts['os_family']`:
- Debian family: Ubuntu, Debian
- RedHat family: RHEL, CentOS, Rocky Linux

### 3. Regex-Based Configuration
Instead of Jinja2 templates, we use Ansible's `lineinfile` and `replace` modules with regex patterns to:
- Preserve comments and formatting in config files
- Make targeted, surgical changes
- Avoid full file replacements
- Maintain upgrade compatibility

### 4. Parallel Execution with Serial Restarts
- Preparation, installation, and configuration run in **parallel** across all nodes for speed
- Service starts and restarts execute **serially** (one node at a time) with health checks
- Each node must be fully operational (UN state) before proceeding to the next

### 5. Idempotent Operations
All tasks are designed to be idempotent - running playbooks multiple times produces the same result without errors.

### 6. Modular Roles
Each component (system setup, Java, HCD, monitoring tools) is a separate role that can be:
- Used independently
- Combined in different playbooks
- Extended or customized

## Inventory Structure

### Multi-Datacenter Support

```ini
[hcd_nodes:children]
dc1
dc2

[dc1]
54.123.45.67 private_ip=10.0.1.10 rack=rack1 dc=dc1 seed=true
54.123.45.68 private_ip=10.0.1.11 rack=rack1 dc=dc1 seed=false
54.123.45.69 private_ip=10.0.1.12 rack=rack2 dc=dc1 seed=false

[dc2]
54.234.56.78 private_ip=10.0.2.10 rack=rack1 dc=dc2 seed=true
54.234.56.79 private_ip=10.0.2.11 rack=rack1 dc=dc2 seed=false
54.234.56.80 private_ip=10.0.2.12 rack=rack2 dc=dc2 seed=false

[reaper_nodes]
10.0.3.10

[monitoring_nodes]
10.0.4.10
```

**Host Variables:**
- First parameter (inventory_hostname): Public IP for SSH connection and broadcast_address
- `private_ip`: Private IP for internal communication (listen_address, rpc_address)
- `rack`: Rack assignment for topology awareness
- `dc`: Datacenter assignment
- `seed`: Boolean flag (true/false) to identify seed nodes

**Note:** Seed nodes are auto-inferred from hosts where `seed=true`, eliminating the need for a separate `[seed_nodes]` group.

## Key Configuration Variables

### Global Variables (group_vars/all.yml)
```yaml
# Ansible settings
ansible_user: ubuntu
ansible_become: yes
ansible_python_interpreter: /usr/bin/python3

# HCD version
hcd_version: "1.2.4"
hcd_tarball_path: "./hcd-{{ hcd_version }}"

# Installation paths
hcd_install_dir: "/opt/hcd"
cassandra_data_dir: "/var/lib/cassandra"
cassandra_log_dir: "/var/log/cassandra"

# Java settings
java_version: "11"

# Auto-discovered runtime values
private_ip: "{{ ansible_default_ipv4['address'] }}"
cpu_cores: "{{ ansible_facts['processor_cores'] }}"
total_memory_mb: "{{ ansible_facts['memtotal_mb'] }}"
```

### HCD-Specific Variables (group_vars/hcd_nodes.yml)
```yaml
# Cluster settings
cluster_name: "Production HCD Cluster"
num_tokens: 16
allocate_tokens_for_local_replication_factor: 3

# Network settings (private/public IP configuration)
listen_address: "{{ ansible_default_ipv4['address'] }}"  # Private IP
rpc_address: "{{ ansible_default_ipv4['address'] }}"     # Private IP
broadcast_address: "{{ public_ip }}"                      # Public IP from inventory
broadcast_rpc_address: "{{ public_ip }}"                  # Public IP from inventory

# Performance tuning (auto-calculated based on hardware)
concurrent_reads: "{{ ansible_facts['processor_cores'] * 2 }}"
concurrent_writes: "{{ ansible_facts['processor_cores'] * 2 }}"
concurrent_compactors: "{{ ansible_facts['processor_cores'] }}"

# JVM settings
heap_size: "31G"
heap_newsize: "8G"
parallel_gc_threads: "{{ ansible_facts['processor_cores'] }}"
conc_gc_threads: "{{ ansible_facts['processor_cores'] }}"
gc_log_path: "{{ cassandra_log_dir }}/gc.log"

# Security (optional)
authenticator: "AllowAllAuthenticator"
authorizer: "AllowAllAuthorizer"
enable_client_encryption: false
enable_internode_encryption: false
```

### Datacenter Variables (group_vars/dc1.yml)
```yaml
datacenter: "dc1"
# Seed nodes are auto-discovered from inventory where seed=true
```

## Configuration Strategy

### Regex-Based Updates

Instead of replacing entire configuration files, we use targeted regex replacements:

```yaml
# Example: Update num_tokens in cassandra.yaml
- name: Set num_tokens
  lineinfile:
    path: "{{ cassandra_yaml_path }}"
    regexp: '^num_tokens:\s*\d+'
    line: "num_tokens: {{ num_tokens }}"
    backrefs: no

# Example: Update heap size in jvm-server.options
- name: Set heap size
  replace:
    path: "{{ jvm_options_path }}"
    regexp: '^-Xmx\d+[GMK]'
    replace: "-Xmx{{ heap_size }}"
```

### Benefits:
1. Preserves file structure and comments
2. Only modifies specific settings
3. Easier to track changes
4. Compatible with HCD upgrades
5. More maintainable than full templates

## System Prerequisites

Based on [HCD Recommended Settings](https://docs.datastax.com/en/hyper-converged-database/1.2/manage/configure/recommended-settings.html):

### Linux Kernel Settings
- Disable swap
- Set vm.max_map_count
- Configure TCP settings
- Set file descriptor limits

### Storage Settings
- Configure I/O scheduler (deadline or noop for SSDs)
- Set readahead values
- Configure filesystem options

### Network Settings
- Increase network buffer sizes
- Configure connection tracking

## Deployment Workflows

### 1. Full Cluster Installation
```bash
ansible-playbook -i inventory/hosts playbooks/install_cluster.yml
```

Executes:
1. System setup (prerequisites)
2. Java installation
3. HCD installation
4. HCD configuration
5. Service startup

### 2. Configuration Update
```bash
ansible-playbook -i inventory/hosts playbooks/configure_cluster.yml
```

Updates configuration without reinstalling.

### 3. Rolling Restart
```bash
ansible-playbook -i inventory/hosts playbooks/rolling_restart.yml
```

Restarts nodes one at a time with health checks.

### 4. Install Monitoring Stack
```bash
ansible-playbook -i inventory/hosts playbooks/install_monitoring.yml
```

Installs MCAC on HCD nodes and configures metrics collection.

## Monitoring, Repairs, and Backups

### MCAC (Metric Collector for Apache Cassandra)
- Installed as agent on each HCD node
- Configurable metrics backend (Prometheus, Graphite, etc.)
- Collects JVM, OS, and Cassandra metrics

### Reaper
- Can be installed on dedicated nodes or co-located
- Configured to connect to HCD cluster
- Manages automated repair schedules

### Medusa
- Installed on HCD nodes
- Supports multiple storage backends (S3, GCS, local)
- Provides backup and restore capabilities

## Security Considerations

### Authentication & Authorization
- Configurable via variables
- Supports internal authentication or LDAP
- Role-based access control

### Encryption
- Client-to-node SSL/TLS
- Node-to-node encryption
- Certificate management

## Testing Strategy

### Pre-deployment Validation
- Syntax check: `ansible-playbook --syntax-check`
- Dry run: `ansible-playbook --check`
- Inventory validation: `ansible-inventory --list`

### Post-deployment Validation
- Service status checks
- Nodetool status verification
- Connectivity tests
- Performance baseline

## Maintenance Operations

### Common Tasks
- Add/remove nodes
- Change configuration
- Upgrade HCD version
- Backup/restore operations
- Repair operations

### Playbook Examples
```bash
# Add new node
ansible-playbook -i inventory/hosts playbooks/add_node.yml -e "new_node=node4-dc1"

# Change heap size
ansible-playbook -i inventory/hosts playbooks/configure_cluster.yml -e "heap_size=32G"

# Run repairs
ansible-playbook -i inventory/hosts playbooks/run_repair.yml
```

## Best Practices

1. **Version Control**: Keep inventory and variables in version control
2. **Secrets Management**: Use Ansible Vault for sensitive data
3. **Testing**: Test in non-production environment first
4. **Backups**: Always backup before major changes
5. **Documentation**: Document custom configurations
6. **Monitoring**: Monitor cluster health during and after changes

## Next Steps

1. Implement each role with OS-specific tasks
2. Create comprehensive variable defaults
3. Build playbooks for different operations
4. Add validation and error handling
5. Create documentation and examples
6. Test on both Debian and RedHat systems