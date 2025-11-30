# HCD Cluster Management - Detailed Implementation Plan

## Phase 1: Project Foundation

### 1.1 Update .gitignore
**File**: `.gitignore`
**Action**: Add pattern to exclude HCD tarball directory
```
hcd-*.*.*
```

### 1.2 Create Ansible Configuration
**File**: `ansible.cfg`
**Purpose**: Configure Ansible behavior
```ini
[defaults]
inventory = inventory/hosts
roles_path = roles
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
stdout_callback = yaml
callbacks_enabled = profile_tasks, timer

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

## Phase 2: Inventory Structure

### 2.1 Main Inventory File
**File**: `inventory/hosts`
**Structure**:
```ini
# HCD Cluster Inventory
# Multi-datacenter, multi-node support
#
# Format: public_ip private_ip=<private_ip> rack=<rack> dc=<dc> seed=<true|false>
#
# Host Variables:
#   inventory_hostname (first parameter): Public IP for SSH and broadcast_address
#   private_ip: Private IP for internal communication (listen_address, rpc_address)
#   rack: Rack assignment for topology awareness
#   dc: Datacenter assignment
#   seed: Boolean (true/false) to identify seed nodes
#
# Note: Seed nodes are auto-inferred from hosts where seed=true

[hcd_nodes:children]
dc1
dc2

[dc1]
# Example nodes for datacenter 1
# 54.123.45.67 private_ip=10.0.1.10 rack=rack1 dc=dc1 seed=true
# 54.123.45.68 private_ip=10.0.1.11 rack=rack1 dc=dc1 seed=false
# 54.123.45.69 private_ip=10.0.1.12 rack=rack2 dc=dc1 seed=false

[dc2]
# Example nodes for datacenter 2
# 54.234.56.78 private_ip=10.0.2.10 rack=rack1 dc=dc2 seed=true
# 54.234.56.79 private_ip=10.0.2.11 rack=rack1 dc=dc2 seed=false
# 54.234.56.80 private_ip=10.0.2.12 rack=rack2 dc=dc2 seed=false

[reaper_nodes]
# Dedicated Reaper nodes (optional - can be co-located)
# 10.0.3.10

[monitoring_nodes]
# Monitoring infrastructure nodes
# 10.0.4.10
```

### 2.2 Global Variables
**File**: `inventory/group_vars/all.yml`
```yaml
---
# Global variables for all hosts

# Ansible connection settings
ansible_user: ubuntu
ansible_become: yes
ansible_python_interpreter: /usr/bin/python3

# HCD version and paths
hcd_version: "1.2.4"
hcd_tarball_name: "hcd-{{ hcd_version }}"
hcd_tarball_path: "./{{ hcd_tarball_name }}"
hcd_install_dir: "/opt/hcd"

# Cassandra directories
cassandra_home: "{{ hcd_install_dir }}"
cassandra_conf_dir: "{{ hcd_install_dir }}/resources/cassandra/conf"
cassandra_data_dir: "/var/lib/cassandra"
cassandra_log_dir: "/var/log/cassandra"
cassandra_user: "cassandra"
cassandra_group: "cassandra"

# Java settings
java_version: "11"
java_home_debian: "/usr/lib/jvm/java-11-openjdk-amd64"
java_home_redhat: "/usr/lib/jvm/java-11-openjdk"

# System settings
disable_swap: true
configure_limits: true
configure_sysctl: true

# Auto-discovered runtime values (set during playbook execution)
private_ip: "{{ ansible_default_ipv4['address'] }}"
cpu_cores: "{{ ansible_facts['processor_cores'] }}"
total_memory_mb: "{{ ansible_facts['memtotal_mb'] }}"
```

### 2.3 HCD Nodes Variables
**File**: `inventory/group_vars/hcd_nodes.yml`
```yaml
---
# HCD cluster configuration

# Cluster identity
cluster_name: "HCD Production Cluster"

# Token allocation
num_tokens: 16
allocate_tokens_for_local_replication_factor: 3

# Network configuration (using private and public IPs)
listen_address: "{{ private_ip }}"                        # Private IP for internal communication
rpc_address: "{{ private_ip }}"                           # Private IP for CQL connections
broadcast_address: "{{ inventory_hostname }}"             # Public IP (inventory_hostname)
broadcast_rpc_address: "{{ inventory_hostname }}"         # Public IP for client connections

# Ports
storage_port: 7000
ssl_storage_port: 7001
native_transport_port: 9042
rpc_port: 9160

# Snitch configuration
endpoint_snitch: "GossipingPropertyFileSnitch"

# Performance tuning (auto-calculated based on hardware)
concurrent_reads: "{{ ansible_facts['processor_cores'] * 2 }}"
concurrent_writes: "{{ ansible_facts['processor_cores'] * 2 }}"
concurrent_counter_writes: 32
concurrent_compactors: "{{ ansible_facts['processor_cores'] }}"  # Set to number of CPU cores
compaction_throughput_mb_per_sec: 64
stream_throughput_outbound_megabits_per_sec: 400

# Memory settings
file_cache_size_in_mb: 512
memtable_heap_space_in_mb: 2048
memtable_offheap_space_in_mb: 2048

# Commitlog settings
commitlog_sync: "periodic"
commitlog_sync_period_in_ms: 10000
commitlog_segment_size_in_mb: 32

# Data directories (can be customized per node)
data_file_directories:
  - "{{ cassandra_data_dir }}/data"
commitlog_directory: "{{ cassandra_data_dir }}/commitlog"
saved_caches_directory: "{{ cassandra_data_dir }}/saved_caches"
hints_directory: "{{ cassandra_data_dir }}/hints"
cdc_raw_directory: "{{ cassandra_data_dir }}/cdc_raw"
metadata_directory: "{{ cassandra_data_dir }}/metadata"

# JVM options
heap_size: "31G"
heap_newsize: "8G"
parallel_gc_threads: "{{ ansible_facts['processor_cores'] }}"  # Set to number of CPU cores
conc_gc_threads: "{{ ansible_facts['processor_cores'] }}"      # Set to number of CPU cores
gc_log_path: "{{ cassandra_log_dir }}/gc.log"
jvm_extra_opts: []

# Security settings
authenticator: "com.datastax.cassandra.auth.AdvancedAuthenticator"
authorizer: "com.datastax.cassandra.auth.AdvancedAuthorizer"
role_manager: "com.datastax.cassandra.auth.AdvancedRoleManager"
enable_client_encryption: false
enable_internode_encryption: false

# Logging
log_level: "INFO"
```

### 2.4 Datacenter-Specific Variables
**File**: `inventory/group_vars/dc1.yml`
```yaml
---
datacenter: "dc1"
# Seed nodes are auto-discovered from inventory where seed=true
```

**File**: `inventory/group_vars/dc2.yml`
```yaml
---
datacenter: "dc2"
# Seed nodes are auto-discovered from inventory where seed=true
```

### 2.5 Monitoring Variables
**File**: `inventory/group_vars/monitoring.yml`
```yaml
---
# MCAC configuration
# MCAC repository URL https://github.com/datastax/metric-collector-for-apache-cassandra
mcac_enabled: true
mcac_version: "0.3.6"
mcac_metrics_backend: "prometheus"
mcac_prometheus_port: 9103

# Reaper configuration
# Reaper code repository URL https://github.com/thelastpickle/cassandra-reaper
# Reaper docs is located at https://cassandra-reaper.io 
reaper_enabled: true
reaper_version: "4.0.1"
reaper_install_type: "dedicated"  # or "colocated"
reaper_ui_port: 8080
reaper_storage_type: "cassandra"

# Medusa configuration
# Medusa code repository URL https://github.com/thelastpickle/cassandra-medusa/tree/master
medusa_enabled: true
medusa_version: "0.26.0"
medusa_storage_provider: "local"  # s3, gcs, azure, local
medusa_bucket_name: "hcd-backups"
medusa_backup_retention_days: 30
```

## Phase 3: Role Implementation

### 3.1 System Setup Role
**Role**: `roles/system_setup`

**Files to create**:
- `roles/system_setup/tasks/main.yml` - Main task orchestrator
- `roles/system_setup/tasks/debian.yml` - Debian-specific tasks
- `roles/system_setup/tasks/redhat.yml` - RedHat-specific tasks
- `roles/system_setup/defaults/main.yml` - Default variables
- `roles/system_setup/templates/limits.conf.j2` - File descriptor limits
- `roles/system_setup/templates/sysctl.conf.j2` - Kernel parameters

**Key tasks**:
1. Disable swap
2. Configure file descriptor limits
3. Set kernel parameters (vm.max_map_count, network buffers)
4. Configure I/O scheduler
5. Install required packages (python3, etc.)

### 3.2 Java Installation Role
**Role**: `roles/java`

**Files to create**:
- `roles/java/tasks/main.yml`
- `roles/java/tasks/debian.yml`
- `roles/java/tasks/redhat.yml`
- `roles/java/defaults/main.yml`

**Key tasks**:
1. Install OpenJDK 11
2. Set JAVA_HOME
3. Configure alternatives
4. Verify installation

### 3.3 HCD Installation Role
**Role**: `roles/hcd_install`

**Files to create**:
- `roles/hcd_install/tasks/main.yml`
- `roles/hcd_install/tasks/user.yml` - Create cassandra user
- `roles/hcd_install/tasks/directories.yml` - Create directory structure
- `roles/hcd_install/tasks/install.yml` - Extract and install HCD
- `roles/hcd_install/defaults/main.yml`
- `roles/hcd_install/handlers/main.yml`

**Key tasks**:
1. Create cassandra user and group
2. Create directory structure
3. Copy tarball to remote host
4. Extract HCD
5. Set ownership and permissions
6. Create systemd service file

### 3.4 HCD Configuration Role
**Role**: `roles/hcd_config`

**Files to create**:
- `roles/hcd_config/tasks/main.yml`
- `roles/hcd_config/tasks/cassandra_yaml.yml` - Configure cassandra.yaml
- `roles/hcd_config/tasks/jvm_options.yml` - Configure JVM options
- `roles/hcd_config/tasks/logback.yml` - Configure logging
- `roles/hcd_config/tasks/rackdc.yml` - Configure rack/DC properties
- `roles/hcd_config/defaults/main.yml`
- `roles/hcd_config/handlers/main.yml`

**Key configuration areas**:

#### cassandra.yaml updates (using regex):
- cluster_name
- num_tokens
- seed_provider
- listen_address / rpc_address
- endpoint_snitch
- data directories
- concurrent operations
- compaction settings
- authenticator / authorizer

#### jvm-server.options updates:
- Heap size (-Xms, -Xmx)
- New generation size (-Xmn)
- GC settings
- JMX settings

#### logback.xml updates:
- Log levels
- Log file paths
- Rotation policies

#### cassandra-rackdc.properties:
- dc
- rack

### 3.5 MCAC Role
**Role**: `roles/mcac`

**Files to create**:
- `roles/mcac/tasks/main.yml`
- `roles/mcac/tasks/install.yml`
- `roles/mcac/tasks/configure.yml`
- `roles/mcac/defaults/main.yml`

**Key tasks**:
1. Download MCAC agent
2. Install to HCD lib directory
3. Configure metrics collection
4. Configure backend (Prometheus/Graphite)
5. Update cassandra-env.sh

### 3.6 Reaper Role
**Role**: `roles/reaper`

**Files to create**:
- `roles/reaper/tasks/main.yml`
- `roles/reaper/tasks/install.yml`
- `roles/reaper/tasks/configure.yml`
- `roles/reaper/defaults/main.yml`

**Key tasks**:
1. Install Reaper (package or Docker)
2. Configure connection to HCD cluster
3. Set up repair schedules
4. Configure authentication

### 3.7 Medusa Role
**Role**: `roles/medusa`

**Files to create**:
- `roles/medusa/tasks/main.yml`
- `roles/medusa/tasks/install.yml`
- `roles/medusa/tasks/configure.yml`
- `roles/medusa/defaults/main.yml`

**Key tasks**:
1. Install Medusa via pip
2. Configure storage backend
3. Set up credentials
4. Configure backup schedules

## Phase 4: Playbook Implementation

### 4.1 Main Installation Playbook
**File**: `playbooks/install_cluster.yml`
```yaml
---
- name: Install HCD Cluster - Parallel Preparation
  hosts: hcd_nodes
  serial: "100%"  # Run in parallel for speed
  gather_facts: yes
  roles:
    - system_setup
    - java
    - hcd_install
    - hcd_config
    - mcac

- name: Start HCD Services - Rolling Start
  hosts: hcd_nodes
  serial: 1  # Start one node at a time
  gather_facts: yes
  tasks:
    - name: Start HCD service
      systemd:
        name: hcd
        state: started
        enabled: yes

    - name: Wait for native transport port (9042) to be available
      wait_for:
        port: 9042
        timeout: 1200
        host: "{{ ansible_default_ipv4['address'] }}"
        state: started

    - name: Wait for node to be in UN (Up/Normal) state
      shell: "{{ hcd_install_dir }}/bin/nodetool status | grep UN | grep {{ ansible_default_ipv4['address'] }}"
      register: node_status
      until: node_status.rc == 0
      retries: 60
      delay: 10
      changed_when: false

    - name: Display node status
      debug:
        msg: "Node {{ inventory_hostname }} is UP and NORMAL"

    - name: Pause before starting next node
      pause:
        seconds: 30
```

### 4.2 Configuration Update Playbook
**File**: `playbooks/configure_cluster.yml`
```yaml
---
- name: Configure HCD Cluster
  hosts: hcd_nodes
  serial: "100%"  # Run in parallel for configuration updates
  gather_facts: yes
  roles:
    - hcd_config
```

### 4.3 Rolling Restart Playbook
**File**: `playbooks/rolling_restart.yml`
```yaml
---
- name: Rolling Restart of HCD Cluster
  hosts: hcd_nodes
  serial: 1  # One node at a time
  gather_facts: yes
  tasks:
    - name: Check current node status
      shell: "{{ hcd_install_dir }}/bin/nodetool status"
      register: pre_restart_status
      ignore_errors: yes
      changed_when: false

    - name: Drain node before restart
      shell: "{{ hcd_install_dir }}/bin/nodetool drain"
      ignore_errors: yes

    - name: Stop HCD service
      systemd:
        name: hcd
        state: stopped

    - name: Wait for service to stop
      wait_for:
        port: 9042
        host: "{{ ansible_default_ipv4['address'] }}"
        state: stopped
        timeout: 300

    - name: Start HCD service
      systemd:
        name: hcd
        state: started

    - name: Wait for native transport port (9042) to be available
      wait_for:
        port: 9042
        timeout: 1200
        host: "{{ ansible_default_ipv4['address'] }}"
        state: started

    - name: Wait for node to join cluster and be in UN state
      shell: "{{ hcd_install_dir }}/bin/nodetool status | grep UN | grep {{ ansible_default_ipv4['address'] }}"
      register: node_status
      until: node_status.rc == 0
      retries: 60
      delay: 10
      changed_when: false

    - name: Verify node is fully operational
      shell: "{{ hcd_install_dir }}/bin/nodetool info"
      register: node_info
      changed_when: false

    - name: Display node status after restart
      debug:
        msg:
          - "Node {{ inventory_hostname }} successfully restarted"
          - "Status: {{ node_status.stdout }}"

    - name: Pause before next node
      pause:
        seconds: 30
        prompt: "Waiting 30 seconds before restarting next node..."
```

### 4.4 Start Cluster Playbook
**File**: `playbooks/start_cluster.yml`
```yaml
---
- name: Start HCD Cluster
  hosts: hcd_nodes
  serial: 1  # Start one node at a time
  gather_facts: yes
  tasks:
    - name: Start HCD service
      systemd:
        name: hcd
        state: started
        enabled: yes

    - name: Wait for native transport port (9042)
      wait_for:
        port: 9042
        timeout: 1200
        host: "{{ ansible_default_ipv4['address'] }}"
        state: started

    - name: Wait for node to be in UN state
      shell: "{{ hcd_install_dir }}/bin/nodetool status | grep UN | grep {{ ansible_default_ipv4['address'] }}"
      register: node_status
      until: node_status.rc == 0
      retries: 60
      delay: 10
      changed_when: false

    - name: Pause before starting next node
      pause:
        seconds: 30
```

### 4.5 Stop Cluster Playbook
**File**: `playbooks/stop_cluster.yml`
```yaml
---
- name: Stop HCD Cluster
  hosts: hcd_nodes
  serial: 1  # Stop one node at a time
  gather_facts: yes
  tasks:
    - name: Drain node before stopping
      shell: "{{ hcd_install_dir }}/bin/nodetool drain"
      ignore_errors: yes

    - name: Stop HCD service
      systemd:
        name: hcd
        state: stopped

    - name: Wait for service to stop
      wait_for:
        port: 9042
        host: "{{ ansible_default_ipv4['address'] }}"
        state: stopped
        timeout: 300

    - name: Pause before stopping next node
      pause:
        seconds: 10
```

### 4.6 Monitoring Playbooks
**Files**:
- `playbooks/install_monitoring.yml`
- `playbooks/install_reaper.yml`
- `playbooks/install_medusa.yml`

## Phase 5: Documentation

### 5.1 Main README
**File**: `README.md`
**Content**:
- Project overview
- Prerequisites
- Quick start guide
- Inventory setup
- Variable configuration
- Common operations
- Troubleshooting

### 5.2 Examples
**Directory**: `examples/`
**Files**:
- `inventory-single-dc.ini` - Single DC example
- `inventory-multi-dc.ini` - Multi DC example
- `group_vars-examples/` - Variable examples

## Implementation Order

1. ✅ Architecture documentation (ARCHITECTURE.md)
2. ✅ Implementation plan (this document)
3. Update .gitignore
4. Create ansible.cfg
5. Create inventory structure
6. Create group_vars files
7. Implement system_setup role
8. Implement java role
9. Implement hcd_install role
10. Implement hcd_config role
11. Implement mcac role
12. Implement reaper role
13. Implement medusa role
14. Create playbooks
15. Create documentation
16. Create examples
17. Testing and validation

## Key Implementation Details

### Regex Patterns for Configuration

#### cassandra.yaml patterns:
```yaml
# Cluster name
regexp: '^cluster_name:\s*.*'
line: "cluster_name: '{{ cluster_name }}'"

# Num tokens
regexp: '^num_tokens:\s*\d+'
line: "num_tokens: {{ num_tokens }}"

# Seeds (dynamically built from hosts where seed=true)
regexp: '^\s+- seeds:\s*".*"'
line: '          - seeds: "{{ seed_nodes_string }}"'
# Where seed_nodes_string is built as:
# seed_nodes_string: "{{ groups['hcd_nodes'] | map('extract', hostvars) | selectattr('seed', 'equalto', true) | map(attribute='private_ip') | list | join(',') }}"

# Listen address (private IP from inventory)
regexp: '^#?listen_address:\s*.*'
line: "listen_address: {{ private_ip }}"

# Broadcast address (public IP - inventory_hostname)
regexp: '^#?\s*broadcast_address:\s*.*'
line: "broadcast_address: {{ inventory_hostname }}"

# RPC address (private IP from inventory)
regexp: '^#?rpc_address:\s*.*'
line: "rpc_address: {{ private_ip }}"

# Broadcast RPC address (public IP - inventory_hostname)
regexp: '^#?\s*broadcast_rpc_address:\s*.*'
line: "broadcast_rpc_address: {{ inventory_hostname }}"

# Concurrent compactors (based on CPU cores)
regexp: '^#?\s*concurrent_compactors:\s*.*'
line: "concurrent_compactors: {{ ansible_facts['processor_cores'] }}"

# Endpoint snitch
regexp: '^endpoint_snitch:\s*.*'
line: "endpoint_snitch: {{ endpoint_snitch }}"

# Data directories
regexp: '^#?\s*data_file_directories:'
line: "data_file_directories:"

# Commitlog directory
regexp: '^#?\s*commitlog_directory:\s*.*'
line: "commitlog_directory: {{ commitlog_directory }}"

# Similarly do for other directories too:
# metadata_directory
# hints_directory
# cdc_raw_directory
# saved_caches_directory

# HCD unified authentication should look like below. Uncomment lines & set values accordingly. Maintain spaces and indentation properly
authenticator:
  class_name: com.datastax.cassandra.auth.AdvancedAuthenticator
  parameters:
    enabled: true
    default_scheme: internal
    plain_text_without_ssl: warn
# HCD Authroizator setting should be like below. Uncomment lines and set accordingly. Maintain spaces and indentation properly
authorizer:
  class_name: com.datastax.cassandra.auth.AdvancedAuthorizer
  parameters:
     enabled: true
```

#### jvm-server.options patterns (for both jvm11 and jvm17):
```yaml
# Heap size
regexp: '^-Xms\d+[GMK]'
replace: "-Xms{{ heap_size }}"

regexp: '^-Xmx\d+[GMK]'
replace: "-Xmx{{ heap_size }}"

# New generation
regexp: '^-Xmn\d+[GMK]'
replace: "-Xmn{{ heap_newsize }}"

# ParallelGCThreads (based on CPU cores)
regexp: '^#?\s*-XX:ParallelGCThreads=.*'
replace: "-XX:ParallelGCThreads={{ ansible_facts['processor_cores'] }}"

# ConcGCThreads (based on CPU cores)
regexp: '^#?\s*-XX:ConcGCThreads=.*'
replace: "-XX:ConcGCThreads={{ ansible_facts['processor_cores'] }}"

# GC logging (uncomment and configure)
regexp: '^#?\s*-Xlog:gc=.*'
replace: "-Xlog:gc=info,heap*=debug,age*=debug,safepoint=info,promotion*=debug:file={{ cassandra_log_dir }}/gc.log:time,uptime,pid,tid,level:filecount=10,filesize=10485760"
```

### Systemd Service File
**File**: `/etc/systemd/system/hcd.service`
```ini
[Unit]
Description=HyperConverged Database (HCD)
After=network.target

[Service]
Type=forking
User=cassandra
Group=cassandra
ExecStart={{ hcd_install_dir }}/bin/hcd cassandra
ExecStop={{ hcd_install_dir }}/bin/nodetool stopdaemon
Restart=on-failure
LimitNOFILE=100000
LimitMEMLOCK=infinity
LimitNPROC=32768
LimitAS=infinity

[Install]
WantedBy=multi-user.target
```

## Testing Strategy

### Unit Testing
- Test each role independently
- Use molecule for role testing
- Verify idempotency

### Integration Testing
- Test full playbook execution
- Verify cluster formation
- Test rolling restarts
- Validate monitoring integration

### Validation Checks
- `nodetool status` - Cluster status
- `nodetool info` - Node information
- `cqlsh` connectivity
- Log file verification
- Service status checks

## Success Criteria

1. ✅ Complete project structure created
2. ✅ All roles implemented and tested
3. ✅ Playbooks execute successfully
4. ✅ Cluster forms correctly
5. ✅ Monitoring tools operational
6. ✅ Documentation complete
7. ✅ Examples provided
8. ✅ Works on both Debian and RedHat systems