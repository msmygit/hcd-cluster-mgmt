# HCD Cluster Management with Ansible

Comprehensive Ansible automation for deploying and managing [Hyper-Converged Database (HCD)](https://docs.datastax.com/en/hyper-converged-database/1.2/get-started/get-started-hcd.html) clusters with monitoring, repair, and backup capabilities.

## 🎯 Features

- **Multi-datacenter support** with flexible node configuration
- **OS-agnostic** implementation (Debian/Ubuntu and RedHat/CentOS)
- **Regex-based configuration** updates (preserves comments, no template replacement)
- **Hardware-aware auto-configuration** (CPU cores, memory)
- **Rolling operations** with health checks
- **Comprehensive monitoring** with MCAC (Metric Collector for Apache Cassandra)
- **Automated repairs** with Cassandra Reaper
- **Backup/restore** with Medusa

## 📋 Prerequisites

### Control Machine
- Ansible 2.20.2 or higher
- Python 3.8+
- SSH access to target nodes

### Target Nodes
- Ubuntu 20.04/22.04 or RHEL/CentOS 7/8
- Minimum 8GB RAM (16GB+ recommended)
- Minimum 4 CPU cores (8+ recommended)
- Root or sudo access
- Network connectivity between all nodes

### HCD Tarball
- HCD 1.2.4 tarball should be placed in `./hcd-1.2.4/` directory
- Download from DataStax or use your licensed version

## 🚀 Quick Start

### 1. Clone and Setup

```bash
git clone <repository-url>
cd hcd-cluster-mgmt

# Ensure HCD tarball is in place
ls hcd-1.2.4/
```

### 2. Configure Inventory

Copy an example inventory and customize it:

```bash
cp inventory/examples/single-dc-3-nodes.ini inventory/hosts
vim inventory/hosts
```

Update with your actual IPs:

```ini
[hcd_nodes]
54.123.45.67 private_ip=10.0.1.10 rack=rack1 dc=dc1 seed=true
54.123.45.68 private_ip=10.0.1.11 rack=rack1 dc=dc1 seed=false
54.123.45.69 private_ip=10.0.1.12 rack=rack2 dc=dc1 seed=false

[dc1]
54.123.45.67
54.123.45.68
54.123.45.69
```

### 3. Configure Variables

Edit group variables as needed:

```bash
# Global settings
vim inventory/group_vars/all.yml

# HCD cluster settings
vim inventory/group_vars/hcd_nodes.yml

# Datacenter-specific settings
vim inventory/group_vars/dc1.yml
```

### 4. Test Connectivity

```bash
ansible -i inventory/hosts hcd_nodes -m ping --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

> [!TIP]
> This assumes the user as `ubuntu`. For others, just suffix --extra-vars "ansible_user=YOUR_USERNAME" to the ansible command.

### 5. Install HCD Cluster

```bash
ansible-playbook -i inventory/hosts playbooks/install_cluster.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

## 📚 Available Playbooks

### Core Operations

#### Install Complete Cluster
```bash
ansible-playbook -i inventory/hosts playbooks/install_cluster.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```
Performs full installation: system setup, Java, HCD installation, configuration, and rolling start.

#### Configure Cluster
```bash
ansible-playbook -i inventory/hosts playbooks/configure_cluster.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```
Updates configuration files and performs rolling restart.

#### Start Cluster
```bash
ansible-playbook -i inventory/hosts playbooks/start_cluster.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```
Starts HCD services with rolling approach and health checks.

#### Stop Cluster
```bash
ansible-playbook -i inventory/hosts playbooks/stop_cluster.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```
Gracefully stops HCD services (drains nodes first).

#### Rolling Restart
```bash
ansible-playbook -i inventory/hosts playbooks/rolling_restart.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```
Performs rolling restart with health checks between nodes.

### Monitoring and Management

#### Install MCAC (Metrics Collection)
```bash
ansible-playbook -i inventory/hosts playbooks/install_monitoring.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```
Installs MCAC agent on all nodes with Prometheus endpoint.

#### Install Reaper (Automated Repairs)
```bash
ansible-playbook -i inventory/hosts playbooks/install_reaper.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```
Installs Cassandra Reaper on the first node for repair management.

#### Install Medusa (Backup/Restore)
```bash
ansible-playbook -i inventory/hosts playbooks/install_medusa.yml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```
Installs Medusa on all nodes for backup and restore operations.

## 🏗️ Architecture

### Directory Structure

```
hcd-cluster-mgmt/
├── ansible.cfg                 # Ansible configuration
├── inventory/
│   ├── hosts                   # Main inventory file
│   ├── examples/              # Example inventory files
│   └── group_vars/            # Variable files
│       ├── all.yml            # Global variables
│       ├── hcd_nodes.yml      # HCD cluster variables
│       ├── dc1.yml            # DC1-specific variables
│       ├── dc2.yml            # DC2-specific variables
│       └── monitoring.yml     # Monitoring variables
├── roles/
│   ├── system_setup/          # System prerequisites
│   ├── java/                  # Java installation
│   ├── hcd_install/           # HCD installation
│   ├── hcd_config/            # HCD configuration
│   ├── mcac/                  # MCAC installation
│   ├── reaper/                # Reaper installation
│   └── medusa/                # Medusa installation
├── playbooks/                 # Ansible playbooks
└── hcd-1.2.4/                # HCD tarball (not in git)
```

### Network Configuration

The automation uses a dual-IP approach:

- **Public IP** (inventory_hostname): Used for SSH access and `broadcast_address`
- **Private IP**: Used for internal cluster communication (`listen_address`, `rpc_address`)

This allows clusters to work in cloud environments with separate public/private networks.

## ⚙️ Configuration

### Key Variables

#### Cluster Configuration (`inventory/group_vars/hcd_nodes.yml`)

```yaml
cluster_name: "HCD_Cluster"
num_tokens: 16
cassandra_authenticator: "PasswordAuthenticator"
cassandra_authorizer: "CassandraAuthorizer"

# Hardware-based auto-configuration
concurrent_compactors: "{{ ansible_facts['processor_cores'] }}"
```

#### JVM Configuration

```yaml
heap_size: "16G"              # -Xms and -Xmx
heap_new_size: "4G"           # -Xmn
```

#### Monitoring Configuration (`inventory/group_vars/monitoring.yml`)

```yaml
# MCAC
mcac_prometheus_enabled: true
mcac_prometheus_port: 9103

# Reaper
reaper_auto_scheduling_enabled: true
reaper_repair_intensity: 0.9

# Medusa
medusa_storage_provider: "s3"
medusa_bucket_name: "cassandra-backups"
```

### Datacenter-Specific Overrides

Create `inventory/group_vars/dc1.yml` for datacenter-specific settings:

```yaml
# DC1 specific settings
cassandra_local_dc: "dc1"
```

## 🔧 Advanced Usage

### Running Specific Roles

```bash
# Only install Java
ansible-playbook -i inventory/hosts playbooks/install_cluster.yml --tags java --key-file PRIVATE_KEY_FILE_PATH --user USERNAME

# Only configure cassandra.yaml
ansible-playbook -i inventory/hosts playbooks/configure_cluster.yml --tags cassandra_yaml --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

### Limiting to Specific Hosts

```bash
# Install on specific datacenter
ansible-playbook -i inventory/hosts playbooks/install_cluster.yml --limit dc1 --key-file PRIVATE_KEY_FILE_PATH --user USERNAME

# Install on specific node
ansible-playbook -i inventory/hosts playbooks/install_cluster.yml --limit 54.123.45.67 --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

### Dry Run

```bash
ansible-playbook -i inventory/hosts playbooks/install_cluster.yml --check --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

## 📊 Monitoring

### MCAC Metrics

After installing MCAC, Prometheus metrics are available at:
```
http://<node-ip>:9103/metrics
```

### Reaper Web UI

Access Reaper at:
```
http://<first-node-ip>:8080/webui
```

### Checking Cluster Status

```bash
# From any node
/opt/hcd/bin/nodetool status

# Via Ansible
ansible -i inventory/hosts hcd_nodes -a "/opt/hcd/bin/nodetool status" --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

## 💾 Backup and Restore

### Creating a Backup

```bash
# On any node
sudo -u cassandra medusa backup --backup-name=backup-$(date +%Y%m%d-%H%M%S)
```

### Listing Backups

```bash
sudo -u cassandra medusa list-backups
```

### Restoring a Backup

```bash
sudo -u cassandra medusa restore --backup-name=<backup-name>
```

## 🔍 Troubleshooting

### Check Ansible Connectivity

```bash
ansible -i inventory/hosts hcd_nodes -m ping --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

### View HCD Logs

```bash
ansible -i inventory/hosts hcd_nodes -a "tail -n 50 /var/log/cassandra/system.log" --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

### Check Service Status

```bash
ansible -i inventory/hosts hcd_nodes -a "systemctl status hcd" --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

### Verify Configuration

```bash
ansible -i inventory/hosts hcd_nodes -a "grep cluster_name /opt/hcd/resources/cassandra/conf/cassandra.yaml" --key-file PRIVATE_KEY_FILE_PATH --user USERNAME
```

### Check ansible inventory graph
```bash
ansible-inventory --graph -i inventory
```

## 📖 Documentation

- [HCD Installation Guide](https://docs.datastax.com/en/hyper-converged-database/1.2/install/install-tarball.html)
- [HCD Configuration](https://docs.datastax.com/en/hyper-converged-database/1.2/manage/configure/recommended-settings.html)
- [MCAC Documentation](https://github.com/datastax/metric-collector-for-apache-cassandra)
- [Reaper Documentation](https://cassandra-reaper.io)
- [Medusa Documentation](https://github.com/thelastpickle/cassandra-medusa)

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## 📝 License

This project is licensed under the Apache License 2.0 - see the LICENSE file for details.

## ⚠️ Important Notes

- **Always test in a non-production environment first**
- **Backup your data before making changes**
- **Review and customize variables for your environment**
- **Ensure proper network security (firewalls, security groups)**
- **Monitor cluster health during and after operations**

## 🆘 Support

For issues and questions:

1. Check the troubleshooting section
2. Review the documentation links
3. Check Ansible logs: `~/.ansible/ansible.log`
4. Open an issue in the repository

## 🎓 Best Practices

1. **Start Small**: Test with a single-node or 3-node cluster first
2. **Use Version Control**: Keep your inventory and variables in git
3. **Document Changes**: Maintain a changelog of configuration changes
4. **Regular Backups**: Schedule regular backups with Medusa
5. **Monitor Metrics**: Set up Prometheus/Grafana for MCAC metrics
6. **Schedule Repairs**: Use Reaper's auto-scheduling feature
7. **Test Restores**: Regularly test backup restoration procedures
8. **Keep Updated**: Stay current with HCD and tool versions

## 🔄 Upgrade Path

To upgrade HCD version:

1. Download new HCD tarball to `./hcd-1.2.x/`
2. Update `hcd_version` in `inventory/group_vars/hcd_nodes.yml`
3. Run rolling restart playbook
4. Verify cluster health

---

**Version**: 1.0.0  
**HCD Version**: 1.2.4  
**Ansible Version**: 2.20.2+
