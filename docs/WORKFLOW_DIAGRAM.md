# HCD Cluster Management - Workflow Diagrams

## Overall Architecture

```mermaid
graph TB
    subgraph "Control Node"
        A[Ansible Controller]
        B[Inventory Files]
        C[Group Variables]
        D[Playbooks]
        E[Roles]
    end
    
    subgraph "Datacenter 1"
        F1[HCD Node 1]
        F2[HCD Node 2]
        F3[HCD Node 3]
    end
    
    subgraph "Datacenter 2"
        G1[HCD Node 4]
        G2[HCD Node 5]
        G3[HCD Node 6]
    end
    
    subgraph "Monitoring"
        H[Reaper Node]
        I[Metrics Backend]
    end
    
    A --> B
    A --> C
    A --> D
    D --> E
    
    A -.SSH.-> F1
    A -.SSH.-> F2
    A -.SSH.-> F3
    A -.SSH.-> G1
    A -.SSH.-> G2
    A -.SSH.-> G3
    A -.SSH.-> H
    
    F1 -.Gossip.-> F2
    F2 -.Gossip.-> F3
    F1 -.Gossip.-> G1
    G1 -.Gossip.-> G2
    G2 -.Gossip.-> G3
    
    F1 -.Metrics.-> I
    F2 -.Metrics.-> I
    F3 -.Metrics.-> I
    G1 -.Metrics.-> I
    G2 -.Metrics.-> I
    G3 -.Metrics.-> I
    
    H -.Repairs.-> F1
    H -.Repairs.-> G1
```

## Deployment Workflow

```mermaid
flowchart TD
    Start[Start Deployment] --> CheckInv[Check Inventory]
    CheckInv --> LoadVars[Load Variables]
    LoadVars --> ValidateHosts[Validate Host Connectivity]
    ValidateHosts --> SysSetup[System Setup Role]
    
    SysSetup --> DisableSwap[Disable Swap]
    SysSetup --> ConfigLimits[Configure File Limits]
    SysSetup --> ConfigSysctl[Configure Kernel Parameters]
    SysSetup --> InstallPkgs[Install Required Packages]
    
    InstallPkgs --> JavaRole[Java Installation Role]
    JavaRole --> InstallJava{OS Family?}
    InstallJava -->|Debian| JavaDeb[Install OpenJDK 11 - apt]
    InstallJava -->|RedHat| JavaRH[Install OpenJDK 11 - yum]
    JavaDeb --> VerifyJava[Verify Java Installation]
    JavaRH --> VerifyJava
    
    VerifyJava --> HCDInstall[HCD Installation Role]
    HCDInstall --> CreateUser[Create Cassandra User]
    CreateUser --> CreateDirs[Create Directory Structure]
    CreateDirs --> CopyTarball[Copy HCD Tarball]
    CopyTarball --> ExtractHCD[Extract HCD]
    ExtractHCD --> SetPerms[Set Permissions]
    SetPerms --> CreateService[Create Systemd Service]
    
    CreateService --> HCDConfig[HCD Configuration Role]
    HCDConfig --> ConfigYAML[Configure cassandra.yaml]
    HCDConfig --> ConfigJVM[Configure JVM Options]
    HCDConfig --> ConfigLog[Configure Logback]
    HCDConfig --> ConfigRackDC[Configure Rack/DC Properties]
    
    ConfigRackDC --> MCACRole[MCAC Installation Role]
    MCACRole --> InstallMCAC[Install MCAC Agent]
    InstallMCAC --> ConfigMCAC[Configure Metrics Collection]
    
    ConfigMCAC --> StartService[Start HCD Service]
    StartService --> VerifyCluster[Verify Cluster Formation]
    VerifyCluster --> Complete[Deployment Complete]
```

## Configuration Update Workflow

```mermaid
flowchart TD
    Start[Configuration Update Request] --> LoadVars[Load Updated Variables]
    LoadVars --> BackupConfig[Backup Current Configuration]
    BackupConfig --> UpdateYAML{Update cassandra.yaml?}
    
    UpdateYAML -->|Yes| RegexYAML[Apply Regex Updates to YAML]
    UpdateYAML -->|No| UpdateJVM{Update JVM Options?}
    
    RegexYAML --> UpdateJVM
    UpdateJVM -->|Yes| RegexJVM[Apply Regex Updates to JVM]
    UpdateJVM -->|No| UpdateLog{Update Logging?}
    
    RegexJVM --> UpdateLog
    UpdateLog -->|Yes| RegexLog[Apply Regex Updates to Logback]
    UpdateLog -->|No| ValidateConfig[Validate Configuration]
    
    RegexLog --> ValidateConfig
    ValidateConfig --> RestartRequired{Restart Required?}
    
    RestartRequired -->|Yes| RollingRestart[Perform Rolling Restart]
    RestartRequired -->|No| ReloadConfig[Reload Configuration]
    
    RollingRestart --> VerifyNode[Verify Node Health]
    ReloadConfig --> VerifyNode
    VerifyNode --> NextNode{More Nodes?}
    
    NextNode -->|Yes| RollingRestart
    NextNode -->|No| Complete[Update Complete]
```

## Role Execution Order

```mermaid
graph LR
    A[system_setup] --> B[java]
    B --> C[hcd_install]
    C --> D[hcd_config]
    D --> E[mcac]
    
    style A fill:#e1f5ff
    style B fill:#e1f5ff
    style C fill:#fff4e1
    style D fill:#fff4e1
    style E fill:#e8f5e9
```

## Multi-Datacenter Deployment Strategy

```mermaid
flowchart TD
    Start[Start Multi-DC Deployment] --> DC1Start[Deploy Datacenter 1]
    
    DC1Start --> DC1Node1[Deploy Node 1 - Seed]
    DC1Node1 --> DC1Wait1[Wait for Node 1 Ready]
    DC1Wait1 --> DC1Node2[Deploy Node 2]
    DC1Node2 --> DC1Wait2[Wait for Node 2 Ready]
    DC1Wait2 --> DC1Node3[Deploy Node 3]
    DC1Node3 --> DC1Complete[DC1 Complete]
    
    DC1Complete --> DC2Start[Deploy Datacenter 2]
    DC2Start --> DC2Node1[Deploy Node 4 - Seed]
    DC2Node1 --> DC2Wait1[Wait for Node 4 Ready]
    DC2Wait1 --> DC2Node2[Deploy Node 5]
    DC2Node2 --> DC2Wait2[Wait for Node 5 Ready]
    DC2Wait2 --> DC2Node3[Deploy Node 6]
    DC2Node3 --> DC2Complete[DC2 Complete]
    
    DC2Complete --> VerifyCluster[Verify Multi-DC Cluster]
    VerifyCluster --> CreateKeyspace[Create Test Keyspace with NTS]
    CreateKeyspace --> VerifyReplication[Verify Data Replication]
    VerifyReplication --> Complete[Deployment Complete]
```

## Monitoring Stack Integration

```mermaid
graph TB
    subgraph "HCD Nodes"
        A1[Node 1 + MCAC Agent]
        A2[Node 2 + MCAC Agent]
        A3[Node 3 + MCAC Agent]
    end
    
    subgraph "Metrics Collection"
        B[Prometheus Server]
        C[Grafana Dashboard]
    end
    
    subgraph "Repair Management"
        D[Reaper UI]
        E[Reaper Backend]
    end
    
    subgraph "Backup Management"
        F[Medusa CLI]
        G[S3/GCS Storage]
    end
    
    A1 -->|Expose Metrics :9103| B
    A2 -->|Expose Metrics :9103| B
    A3 -->|Expose Metrics :9103| B
    
    B -->|Query Metrics| C
    
    E -->|Schedule Repairs| A1
    E -->|Schedule Repairs| A2
    E -->|Schedule Repairs| A3
    D -->|Manage| E
    
    F -->|Backup| A1
    F -->|Backup| A2
    F -->|Backup| A3
    A1 -->|Store Snapshots| G
    A2 -->|Store Snapshots| G
    A3 -->|Store Snapshots| G
```

## Rolling Restart Process

```mermaid
sequenceDiagram
    participant A as Ansible
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3
    participant C as Cluster
    
    A->>N1: Check Node Status
    N1-->>A: Status: UP
    A->>N1: Drain Node
    N1-->>A: Drained
    A->>N1: Stop Service
    N1-->>A: Stopped
    A->>N1: Start Service
    N1-->>A: Started
    A->>N1: Wait for UN Status
    N1-->>A: Status: UN
    
    A->>N2: Check Node Status
    N2-->>A: Status: UP
    A->>N2: Drain Node
    N2-->>A: Drained
    A->>N2: Stop Service
    N2-->>A: Stopped
    A->>N2: Start Service
    N2-->>A: Started
    A->>N2: Wait for UN Status
    N2-->>A: Status: UN
    
    A->>N3: Check Node Status
    N3-->>A: Status: UP
    A->>N3: Drain Node
    N3-->>A: Drained
    A->>N3: Stop Service
    N3-->>A: Stopped
    A->>N3: Start Service
    N3-->>A: Started
    A->>N3: Wait for UN Status
    N3-->>A: Status: UN
    
    A->>C: Verify Cluster Health
    C-->>A: All Nodes UP
```

## Variable Inheritance Flow

```mermaid
graph TD
    A[inventory/group_vars/all.yml] --> B[Global Variables]
    C[inventory/group_vars/hcd_nodes.yml] --> D[HCD Cluster Variables]
    E[inventory/group_vars/dc1.yml] --> F[DC1 Variables]
    G[inventory/group_vars/dc2.yml] --> H[DC2 Variables]
    I[inventory/hosts - Host Vars] --> J[Per-Host Variables]
    
    B --> K[Final Variable Set]
    D --> K
    F --> K
    H --> K
    J --> K
    
    K --> L[Applied to Roles]
    
    style K fill:#ffeb3b
    style L fill:#4caf50
```

## File Structure Overview

```mermaid
graph TD
    A[hcd-cluster-mgmt/] --> B[inventory/]
    A --> C[roles/]
    A --> D[playbooks/]
    A --> E[hcd-1.2.4/]
    A --> F[Documentation]
    
    B --> B1[hosts]
    B --> B2[group_vars/]
    
    C --> C1[system_setup/]
    C --> C2[java/]
    C --> C3[hcd_install/]
    C --> C4[hcd_config/]
    C --> C5[mcac/]
    C --> C6[reaper/]
    C --> C7[medusa/]
    
    D --> D1[install_cluster.yml]
    D --> D2[configure_cluster.yml]
    D --> D3[rolling_restart.yml]
    D --> D4[start_cluster.yml]
    D --> D5[stop_cluster.yml]
    
    F --> F1[README.md]
    F --> F2[ARCHITECTURE.md]
    F --> F3[IMPLEMENTATION_PLAN.md]
    
    style A fill:#2196f3,color:#fff
    style B fill:#4caf50,color:#fff
    style C fill:#ff9800,color:#fff
    style D fill:#9c27b0,color:#fff
```

## Decision Tree for Configuration Method

```mermaid
graph TD
    Start[Configuration Change Needed] --> Q1{Full File Replacement?}
    
    Q1 -->|Yes| UseWrite[Use write_to_file]
    Q1 -->|No| Q2{Single Line Change?}
    
    Q2 -->|Yes| UseLineinfile[Use lineinfile with regex]
    Q2 -->|No| Q3{Multiple Similar Changes?}
    
    Q3 -->|Yes| UseReplace[Use replace with regex]
    Q3 -->|No| Q4{Complex Multi-line?}
    
    Q4 -->|Yes| UseBlockinfile[Use blockinfile]
    Q4 -->|No| UseLineinfile
    
    UseWrite --> Validate[Validate Configuration]
    UseLineinfile --> Validate
    UseReplace --> Validate
    UseBlockinfile --> Validate
    
    Validate --> Complete[Configuration Applied]
```

## Backup and Restore Workflow

```mermaid
flowchart TD
    Start[Backup Request] --> CheckMedusa{Medusa Installed?}
    
    CheckMedusa -->|No| InstallMedusa[Install Medusa]
    CheckMedusa -->|Yes| CreateSnapshot[Create Snapshot]
    
    InstallMedusa --> CreateSnapshot
    CreateSnapshot --> UploadS3[Upload to S3/GCS]
    UploadS3 --> VerifyBackup[Verify Backup]
    VerifyBackup --> Complete[Backup Complete]
    
    RestoreStart[Restore Request] --> StopNode[Stop HCD Node]
    StopNode --> ClearData[Clear Data Directories]
    ClearData --> DownloadBackup[Download from S3/GCS]
    DownloadBackup --> RestoreData[Restore Data]
    RestoreData --> StartNode[Start HCD Node]
    StartNode --> VerifyRestore[Verify Data]
    VerifyRestore --> RestoreComplete[Restore Complete]
```

## Notes

- All diagrams use Mermaid syntax for easy rendering in Markdown viewers
- Workflows show the logical flow of operations
- Architecture diagrams illustrate component relationships
- Decision trees help choose the right approach for different scenarios