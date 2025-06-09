# Kubernetes + Keeper Secrets Manager Suite

A comprehensive DevOps automation suite for Kubernetes development with integrated Keeper Secrets Manager support. This suite provides production-ready tools for cluster management, secrets automation, deployment orchestration, and security compliance.

## 🚀 Quick Start & Usage Instructions

### Installation
```bash
# Basic installation (recommended for most users)
python scripts/install-k8s-keeper.py --allow-root

# Production-only installation (no development tools)
python scripts/install-k8s-keeper.py --allow-root --production-only

# Development-only installation (no External Secrets/Keeper)
python scripts/install-k8s-keeper.py --allow-root --dev-only

# Custom port configuration
python scripts/install-k8s-keeper.py --allow-root --interactive-ports
```

### Post-Installation Verification

#### 1. Access the Kubernetes Dashboard
```bash
# Dashboard URL: https://localhost:30001 (or your selected port)
# Use the token provided at the end of installation, or generate a new one:
kubectl create token admin-user -n kubernetes-dashboard --duration=8760h
```

#### 2. Verify All Components
```bash
# Check all pods across namespaces
kubectl get pods --all-namespaces

# Verify Keeper integration components
kubectl get secretstores,externalsecrets -n keeper-demo

# Expected output:
# secretstore.external-secrets.io/keeper-secretstore   STATUS: Valid, READY: True
# externalsecret.external-secrets.io/keeper-example-secret   STATUS: SecretSyncedError (expected with placeholder)
```

#### 3. Configure Real Keeper Secrets

**Step 1: Find Your Keeper Record UIDs**
```bash
ksm secret list
```

**Step 2: Edit the Example ExternalSecret**
```bash
kubectl edit externalsecret keeper-example-secret -n keeper-demo
```

**Step 3: Replace the Placeholder**
Change this line in the YAML:
```yaml
key: "PLACEHOLDER_RECORD_UID"
```
To your actual record UID:
```yaml
key: "your_actual_record_uid_here"  # e.g., "KSM12345abcdef"
```

**Step 4: Monitor Secret Synchronization**
```bash
# Watch for successful sync (should change from SecretSyncedError to Ready)
kubectl get externalsecrets -n keeper-demo -w

# Verify the secret was created
kubectl get secrets -n keeper-demo

# View the synchronized secret data
kubectl describe secret keeper-synced-secret -n keeper-demo
```

### Expected Status After Installation

#### ✅ Normal Healthy State
- **Dashboard**: Accessible at https://localhost:30001 (or custom port)
- **All Pods**: Running in respective namespaces
- **SecretStore**: Status "Valid" and Ready "True"
- **ExternalSecret**: Status "SecretSyncedError" (expected with placeholder UID)

#### ⚠️ Normal "Errors" (Not Actually Problems)
- **SecretSyncedError**: Expected until you replace `PLACEHOLDER_RECORD_UID` with real UIDs
- **Certificate warnings**: Normal for self-signed dashboard certificates

#### ❌ Actual Problems to Investigate
- Pods stuck in "Pending" or "CrashLoopBackOff" states
- SecretStore showing "Invalid" status
- External Secrets CRDs not found

---

## 📁 Project Structure

```
kubernetes-keeper-suite/
├── scripts/                    # Core automation scripts
│   ├── install-k8s-keeper.py   # Foundation installer (7-stage process)
│   ├── manage-secrets.py       # Secrets management with Enhanced UX
│   ├── k8s-devops-automation.py # DevOps operations & monitoring
│   └── advanced-secrets-manager.py # Advanced secrets workflows
├── config/                     # Configuration templates
│   ├── environments.yaml       # Environment-specific configs
│   └── keeper-config.template.json # Keeper configuration template
├── examples/                   # Usage examples & templates
│   ├── basic-secret.yaml       # Basic ExternalSecret example
│   ├── docker-registry-example.yaml # Docker registry auth
│   └── push-secret-example.yaml # PushSecret example
├── docs/                       # Comprehensive documentation
│   └── DEVELOPMENT.md          # Development workflow guide
├── k8s-templates/              # Kubernetes YAML templates
│   ├── deployment.yaml         # Deployment template with variables
│   ├── service.yaml            # Service template
│   ├── ingress.yaml            # Ingress template
│   ├── configmap.yaml          # ConfigMap template
│   └── hpa.yaml                # HorizontalPodAutoscaler template
└── Makefile                    # Automation shortcuts
```

---

## 🏗️ Architecture & Design Philosophy

### Core Design Principles
- **Stage-based Architecture**: Modular installation stages that can be resumed independently
- **State Persistence**: Installation state is saved and can be resumed from any point
- **Intelligent Error Recovery**: Advanced diagnostics and automatic recovery mechanisms
- **Port Conflict Resolution**: Automatic port scanning and intelligent port selection
- **User Experience Focus**: Clean logging, interactive prompts, and helpful guidance

### Key Features
- **7-Stage Installation Process**: Prerequisites → Tools → Cluster → Dashboard → External Secrets → Keeper → Validation
- **Smart Port Management**: Automatic port scanning, conflict detection, and user selection
- **Comprehensive Diagnostics**: Advanced troubleshooting and validation mechanisms
- **Multiple Installation Modes**: Production, development, interactive, and automated modes
- **Complete Uninstall Capability**: Clean removal of all components with granular control

---

## 📋 Complete Script Documentation & API Reference

### 1. Foundation Installer (`install-k8s-keeper.py`)

**Purpose**: Complete Kubernetes cluster setup with Keeper Secrets Manager integration using a 7-stage installation process with intelligent port management and state persistence.

#### Command Syntax
```bash
python scripts/install-k8s-keeper.py [OPTIONS]
```

#### Core Configuration Classes

##### `BuildConfig` (Dataclass)
```python
@dataclass
class BuildConfig:
    # Security options
    allow_root: bool = False
    allow_destructive: bool = False
    
    # Installation modes
    production_only: bool = False
    dev_only: bool = False
    skip_dashboard: bool = False
    
    # Keeper integration
    ksm_token: Optional[str] = None
    ksm_config_file: Optional[str] = None
    
    # Port configuration
    auto_port_selection: Optional[bool] = None
    custom_ports: Optional[Dict[str, int]] = None
    
    # State management
    resume_stage: Optional[str] = None
    skip_stages: List[str] = None
    work_dir: str = "/tmp/k8s-keeper-build"
    
    # Debug options
    debug_mode: bool = False
    debug_output_file: Optional[str] = None
    reconfigure_mode: bool = False
```

##### `BuildState` (Dataclass)
```python
@dataclass
class BuildState:
    config: BuildConfig
    stages_completed: List[str] = None
    current_stage: Optional[str] = None
    keeper_folders: List[Dict[str, str]] = None
    cluster_ready: bool = False
    dashboard_token: Optional[str] = None
    selected_ports: Dict[str, int] = None
```

#### Build Options
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--allow-root` | flag | false | **Security**: Permit execution as root user |
| `--production-only` | flag | false | **Mode**: Install only production components (no dev tools) |
| `--dev-only` | flag | false | **Mode**: Install only development components (no External Secrets) |
| `--skip-dashboard` | flag | false | **Component**: Skip Kubernetes Dashboard installation |

#### Keeper Integration Options
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--ksm-token` | string | None | **Auth**: Keeper Secrets Manager one-time token |
| `--ksm-config` | path | None | **Auth**: Path to existing KSM configuration file |

#### Advanced Port Configuration System
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--auto-ports` | flag | false | **Network**: Automatically select available ports using intelligent scanning |
| `--interactive-ports` | flag | false | **Network**: Interactive port selection with comprehensive analysis |
| `--http-port` | int | 8080 | **Network**: Custom HTTP port for cluster access |
| `--https-port` | int | 8443 | **Network**: Custom HTTPS port for cluster access |
| `--dashboard-port` | int | 30001 | **Network**: Custom dashboard port (NodePort) |
| `--nodeport` | int | 30000 | **Network**: Custom NodePort base for services |

#### State Management & Resume System
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--resume` | string | None | **State**: Resume from specific stage (01-07) |
| `--skip-stages` | string | None | **State**: Skip specified stages (comma-separated) |
| `--work-dir` | path | `/tmp/k8s-keeper-build` | **State**: Custom work directory for logs/state persistence |
| `--debug` | flag | false | **Debug**: Enable debug mode with verbose logging |
| `--debug-output` | path | None | **Debug**: Save debug output to file |
| `--reconfigure` | flag | false | **State**: Interactive reconfigure mode |
| `--allow-destructive` | flag | false | **Security**: Allow destructive operations (requires confirmation) |

#### Comprehensive Uninstall System
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--uninstall` | flag | false | **Removal**: Interactive uninstall menu with granular options |
| `--uninstall-all` | flag | false | **Removal**: Complete removal (cluster + tools + dashboard + Docker cleanup) |
| `--uninstall-cluster` | flag | false | **Removal**: Remove only Kubernetes cluster while preserving tools |
| `--uninstall-tools` | flag | false | **Removal**: Remove only installed tools (kubectl, kind, helm, go, ksm) |
| `--uninstall-dashboard` | flag | false | **Removal**: Remove only Kubernetes Dashboard |
| `--clean-external-secrets` | flag | false | **Removal**: Specialized cleanup for corrupted External Secrets installations |

#### 7-Stage Installation Process

1. **Stage 01 - Prerequisites Check**: System validation, Docker status, Python version
2. **Stage 02 - Tools Installation**: kubectl, Kind, Helm, Go, KSM CLI installation
3. **Stage 03 - Cluster Setup**: Kind cluster creation with intelligent port mapping
4. **Stage 04 - Dashboard**: Kubernetes Dashboard with admin access and NodePort
5. **Stage 05 - External Secrets**: External Secrets Operator with advanced CRD validation
6. **Stage 06 - Keeper Integration**: KSM configuration, SecretStore creation, example setup
7. **Stage 07 - Validation**: Comprehensive system validation and usage information

#### Port Management System Features

##### Intelligent Port Selection Methods
1. **Automatic Selection** (`--auto-ports`): Scans system, detects conflicts, suggests optimal ports
2. **Interactive Selection** (`--interactive-ports`): Comprehensive analysis with user control
3. **Custom Assignment**: Specific port assignment with conflict detection

##### Port Analysis Categories
- **Web Services** (8000-8100): HTTP alternatives and development servers
- **Alt HTTP** (3000-3100): Framework defaults and development tools
- **Development** (4000-4100): Local development server ranges
- **Kubernetes NodePort** (30000-32767): Kubernetes service exposure range

##### Conflict Resolution
- **Automatic Detection**: Real-time port availability scanning
- **Alternative Suggestions**: Intelligent fallback port recommendations
- **User Warnings**: Clear notifications of port conflicts with override options

#### Installation Examples
```bash
# Basic installation with automatic port selection
python scripts/install-k8s-keeper.py --allow-root --auto-ports

# Production installation with custom ports
python scripts/install-k8s-keeper.py --allow-root --production-only --http-port 8090 --dashboard-port 30005

# Development installation with interactive port selection
python scripts/install-k8s-keeper.py --allow-root --dev-only --interactive-ports

# Resume from External Secrets stage after failure
python scripts/install-k8s-keeper.py --resume 05 --allow-root

# Debug installation with custom work directory
python scripts/install-k8s-keeper.py --allow-root --debug --work-dir /custom/build --debug-output install-debug.log

# Skip dashboard and use Keeper token
python scripts/install-k8s-keeper.py --allow-root --skip-dashboard --ksm-token "YOUR_KEEPER_TOKEN"

# Complete uninstall
python scripts/install-k8s-keeper.py --uninstall-all --allow-root
```

#### Common Operations & Recovery
```bash
# Resume failed installation from specific stage
python scripts/install-k8s-keeper.py --resume 05 --allow-root

# Clean up corrupted External Secrets installation
python scripts/install-k8s-keeper.py --clean-external-secrets --allow-root

# Interactive uninstall menu
python scripts/install-k8s-keeper.py --uninstall --allow-root

# Check installation logs
cat /tmp/k8s-keeper-build/install.log

# Verify cluster status
kubectl cluster-info
kubectl get pods --all-namespaces
```

---

### 2. Enhanced Secrets Management (`manage-secrets.py`)

**Purpose**: Comprehensive Keeper Secrets Manager integration with Enhanced UX, validation, and guided assistance

#### Command Syntax
```bash
python scripts/manage-secrets.py [OPTIONS]
```

#### Core Configuration Classes

##### `SecretConfig` (Dataclass)
```python
@dataclass
class SecretConfig:
    record_uid: str
    secret_name: str
    namespace: str = "default"
    properties: List[str] = None  # Defaults to ["login", "password"]
    template: Optional[str] = None
    refresh_interval: str = "30s"
```

##### `KeeperRecord` (Dataclass)
```python
@dataclass
class KeeperRecord:
    uid: str
    title: str
    record_type: str
    folder: Optional[str] = None
    properties: List[str] = None
```

#### Discovery & Search Commands
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--list-records` | flag | false | **Discovery**: List all Keeper records with UID, type, and title |
| `--search` | string | None | **Discovery**: Search records by keyword with filtering |

#### Enhanced Secret Management
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--create` | flag | false | **Interactive**: Enhanced wizard with validation and examples |
| `--deploy` | flag | false | **Quick**: Quick deploy (requires --record and --name) |
| `--record` | string | None | **Config**: Keeper record UID with validation |
| `--name` | string | None | **Config**: Kubernetes secret name (validated against K8s naming rules) |
| `--namespace` | string | `default` | **Config**: Kubernetes namespace (validated) |
| `--properties` | string | `login,password` | **Config**: Comma-separated properties with validation |

#### Monitoring & Status Commands
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--status` | flag | false | **Monitor**: Show comprehensive External Secrets status |
| `--watch` | flag | false | **Monitor**: Real-time monitoring (use with --status) |
| `--debug` | flag | false | **Debug**: Enhanced debug mode with detailed logging |

#### Enhanced UX Features

##### Interactive Secret Creation Wizard
1. **Record Discovery**: Automatic Keeper vault enumeration with search
2. **Smart Selection**: Record selection with search, pagination, and manual entry
3. **Validation**: Real-time Kubernetes naming validation
4. **Property Analysis**: Automatic property discovery with custom selection
5. **Progress Feedback**: Step-by-step progress with helpful tips
6. **Error Recovery**: Comprehensive error handling with troubleshooting guidance

##### Validation System
- **Kubernetes Names**: RFC 1123 compliance validation for secrets and namespaces
- **Record UIDs**: Format validation and existence checking
- **Properties**: Available property validation against record schema
- **Namespace**: Existence checking with auto-creation

##### Enhanced Error Handling
- **User-Friendly Messages**: Clear, actionable error descriptions
- **Troubleshooting Guidance**: Step-by-step recovery instructions
- **Examples**: Inline examples for common operations
- **Help Tips**: Contextual guidance throughout the workflow

#### Secret Management Examples
```bash
# Enhanced interactive creation with full guidance
python scripts/manage-secrets.py --create

# Quick discovery and listing
python scripts/manage-secrets.py --list-records
python scripts/manage-secrets.py --search "production"

# Validated quick deployment
python scripts/manage-secrets.py --deploy --record "ABC123def456" --name "db-credentials" --namespace "production"

# Custom properties with validation
python scripts/manage-secrets.py --deploy --record "API789xyz" --name "api-keys" --properties "api_key,secret_key,endpoint_url"

# Comprehensive monitoring
python scripts/manage-secrets.py --status
python scripts/manage-secrets.py --status --watch

# Debug mode with enhanced logging
python scripts/manage-secrets.py --create --debug
```

#### Secret Workflow Examples
```bash
# Complete secret setup workflow
python scripts/manage-secrets.py --list-records                    # Discover available records
python scripts/manage-secrets.py --search "database"              # Find specific records
python scripts/manage-secrets.py --create                         # Interactive creation
python scripts/manage-secrets.py --status --watch                 # Monitor sync status

# Production secret deployment
python scripts/manage-secrets.py --deploy \
  --record "PROD_DB_123456" \
  --name "production-database" \
  --namespace "production" \
  --properties "username,password,host,port,database"

# Development workflow
python scripts/manage-secrets.py --deploy \
  --record "DEV_API_789" \
  --name "dev-api-keys" \
  --namespace "development" \
  --properties "api_key,webhook_secret"
```

---

### 3. DevOps Automation (`k8s-devops-automation.py`)

**Purpose**: Comprehensive deployment, monitoring, backup, CI/CD, and security management

#### Command Syntax
```bash
python scripts/k8s-devops-automation.py [OPTIONS]
```

#### Core Configuration Classes

##### `DeploymentConfig` (Dataclass)
```python
@dataclass
class DeploymentConfig:
    name: str
    namespace: str
    image: str
    tag: str = "latest"
    replicas: int = 1
    strategy: DeploymentStrategy = DeploymentStrategy.ROLLING
    environment: EnvironmentType = EnvironmentType.DEVELOPMENT
    secrets: List[str] = field(default_factory=list)
    config_maps: List[str] = field(default_factory=list)
    ports: List[int] = field(default_factory=list)
    health_check_path: str = "/health"
    resources: Dict[str, str] = field(default_factory=dict)
    env_vars: Dict[str, str] = field(default_factory=dict)
```

##### `MonitoringConfig` (Dataclass)
```python
@dataclass
class MonitoringConfig:
    enable_metrics: bool = True
    enable_logging: bool = True
    enable_tracing: bool = False
    prometheus_enabled: bool = True
    grafana_enabled: bool = True
    alertmanager_enabled: bool = True
    log_level: str = "info"
    retention_days: int = 30
```

##### `BackupConfig` (Dataclass)
```python
@dataclass
class BackupConfig:
    enabled: bool = True
    schedule: str = "0 2 * * *"  # Daily at 2 AM
    retention_days: int = 7
    include_secrets: bool = True
    include_configs: bool = True
    storage_path: str = "/tmp/k8s-backups"
```

#### Deployment Management
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--deploy` | flag | false | **Interactive**: Comprehensive deployment wizard with templates |
| `--quick-deploy` | flag | false | **Quick**: Quick deployment (requires --name and --image) |
| `--list-deployments` | flag | false | **List**: List all deployments with status and metrics |
| `--scale` | string | None | **Scale**: Scale deployment (format: name:namespace:replicas) |
| `--update` | string | None | **Update**: Update deployment image (format: name:namespace:image:tag) |
| `--rollback` | string | None | **Rollback**: Rollback deployment (format: name:namespace[:revision]) |
| `--delete-deployment` | string | None | **Delete**: Delete deployment with confirmation (format: name:namespace) |

#### Monitoring & Observability Stack
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--setup-monitoring` | flag | false | **Setup**: Complete monitoring stack (Prometheus, Grafana, Alertmanager) |
| `--cluster-status` | flag | false | **Status**: Comprehensive cluster health and resource usage |
| `--metrics` | flag | false | **Metrics**: Detailed cluster metrics and performance data |

#### Backup & Recovery System
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--setup-backup` | flag | false | **Setup**: Complete backup system with Velero and scheduling |
| `--create-backup` | string | None | **Backup**: Create immediate backup (optional custom name) |
| `--list-backups` | flag | false | **List**: List all available backups with status and metadata |
| `--restore-backup` | string | None | **Restore**: Restore from backup with confirmation |

#### Helm Chart Management
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--list-helm` | flag | false | **List**: List all Helm releases with status and versions |
| `--install-chart` | string | None | **Install**: Install Helm chart (format: release:chart:namespace[:repo-url]) |

#### CI/CD Pipeline Integration
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--setup-cicd` | string | None | **Setup**: Setup CI/CD pipeline (jenkins or argocd) |

#### Security & Compliance Management
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--setup-security` | flag | false | **Setup**: Complete security stack (Falco, OPA Gatekeeper, Network Policies) |
| `--security-scan` | flag | false | **Scan**: Comprehensive vulnerability scanning |
| `--compliance-report` | flag | false | **Report**: Generate detailed compliance report |

#### Common Configuration Parameters
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--name` | string | None | **Config**: Application/resource name |
| `--image` | string | None | **Config**: Container image |
| `--tag` | string | `latest` | **Config**: Image tag |
| `--namespace` | string | `default` | **Config**: Kubernetes namespace |
| `--replicas` | int | 1 | **Config**: Number of replicas |
| `--debug` | flag | false | **Debug**: Enhanced debug logging |

#### Comprehensive DevOps Examples
```bash
# Complete DevOps setup workflow
python scripts/k8s-devops-automation.py --setup-monitoring      # Setup Prometheus/Grafana stack
python scripts/k8s-devops-automation.py --setup-backup         # Setup Velero backup system
python scripts/k8s-devops-automation.py --setup-security       # Setup security scanning tools
python scripts/k8s-devops-automation.py --setup-cicd jenkins   # Setup Jenkins CI/CD

# Interactive application deployment
python scripts/k8s-devops-automation.py --deploy

# Quick deployment with full configuration
python scripts/k8s-devops-automation.py --quick-deploy \
  --name "web-app" \
  --image "nginx" \
  --tag "1.21" \
  --namespace "production" \
  --replicas 3

# Deployment lifecycle management
python scripts/k8s-devops-automation.py --scale web-app:production:5
python scripts/k8s-devops-automation.py --update web-app:production:nginx:1.22
python scripts/k8s-devops-automation.py --rollback web-app:production:2

# Monitoring and observability
python scripts/k8s-devops-automation.py --cluster-status
python scripts/k8s-devops-automation.py --metrics

# Backup operations
python scripts/k8s-devops-automation.py --create-backup "weekly-backup-$(date +%Y%m%d)"
python scripts/k8s-devops-automation.py --list-backups
python scripts/k8s-devops-automation.py --restore-backup "weekly-backup-20241201"

# Security operations
python scripts/k8s-devops-automation.py --security-scan --namespace production
python scripts/k8s-devops-automation.py --compliance-report

# Helm operations
python scripts/k8s-devops-automation.py --list-helm
python scripts/k8s-devops-automation.py --install-chart prometheus:prometheus-community/kube-prometheus-stack:monitoring
```

#### Advanced Monitoring Features

##### Prometheus Stack Setup
- **Prometheus**: Metrics collection and alerting
- **Grafana**: Visualization and dashboards (NodePort: 30091)
- **Alertmanager**: Alert routing and management (NodePort: 30092)
- **Node Exporter**: System metrics collection
- **Custom Dashboards**: Application-specific monitoring

##### Backup System Features
- **Velero Integration**: Complete cluster backup and restore
- **Scheduled Backups**: Automated backup scheduling with retention
- **Selective Backup**: Namespace and resource filtering
- **Cross-cluster Restore**: Disaster recovery capabilities

##### Security Stack Components
- **Falco**: Runtime security monitoring and alerting
- **OPA Gatekeeper**: Policy enforcement and compliance
- **Network Policies**: Micro-segmentation and traffic control
- **Vulnerability Scanning**: Container image security assessment

---

### 4. Advanced Secrets Manager (`advanced-secrets-manager.py`)

**Purpose**: Advanced secrets workflows, templates, bulk operations, and automation

#### Command Syntax
```bash
python scripts/advanced-secrets-manager.py [OPTIONS]
```

#### Template Management System
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--list-templates` | flag | false | **Templates**: List available secret templates with descriptions |
| `--template` | string | None | **Templates**: Create secret from template (postgres, mysql, docker-registry, etc.) |
| `--create-template` | string | None | **Templates**: Create new custom template |

#### Bulk Operations & Automation
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--import-folder` | string | None | **Bulk**: Import all secrets from Keeper folder with batch processing |
| `--export-secrets` | string | None | **Bulk**: Export secrets configuration to file |
| `--sync-all` | flag | false | **Bulk**: Synchronize all ExternalSecrets with status reporting |

#### Advanced Security Configuration
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--rotation-policy` | string | None | **Security**: Set automatic rotation policy for secrets |
| `--access-policy` | string | None | **Security**: Configure RBAC access policies |
| `--audit-log` | flag | false | **Security**: Enable comprehensive audit logging |

#### Workflow Management & Automation
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--workflow` | string | None | **Workflow**: Execute predefined workflow (dev-deploy, prod-migrate, etc.) |
| `--schedule` | string | None | **Workflow**: Schedule workflow execution with cron syntax |

#### Built-in Template System

##### Available Templates
- **postgres**: PostgreSQL database credentials with connection parameters
- **mysql**: MySQL database credentials with host/port configuration
- **docker-registry**: Docker registry authentication for private repositories
- **api-key**: API key authentication with endpoint configuration
- **tls-cert**: TLS certificate and private key for HTTPS
- **generic**: Generic key-value secret for custom use cases

##### Template Features
- **Auto-configuration**: Intelligent property mapping based on record type
- **Validation**: Template-specific validation rules
- **Best Practices**: Security and naming conventions enforcement
- **Documentation**: Inline comments and usage examples

#### Advanced Examples
```bash
# Template operations
python scripts/advanced-secrets-manager.py --list-templates
python scripts/advanced-secrets-manager.py --template postgres --record "DB_PROD_123" --name "postgres-creds" --namespace "database"

# Bulk operations
python scripts/advanced-secrets-manager.py --import-folder "production-secrets"
python scripts/advanced-secrets-manager.py --export-secrets secrets-backup.yaml
python scripts/advanced-secrets-manager.py --sync-all

# Security policies
python scripts/advanced-secrets-manager.py --rotation-policy "30d" --name "database-credentials"
python scripts/advanced-secrets-manager.py --access-policy "team:database-admins" --namespace "production"

# Workflow automation
python scripts/advanced-secrets-manager.py --workflow "prod-deployment" --namespace "production"
python scripts/advanced-secrets-manager.py --schedule "0 2 * * 0" --workflow "weekly-rotation"
```

---

## 🔧 Configuration System & Environment Management

### Environment Configuration (`config/environments.yaml`)

```yaml
environments:
  development:
    namespace: "dev"
    replicas: 1
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
    monitoring:
      enabled: true
      retention: "7d"
    security:
      network_policies: false
      pod_security: "baseline"
    
  staging:
    namespace: "staging"
    replicas: 2
    resources:
      requests:
        memory: "256Mi"
        cpu: "200m"
      limits:
        memory: "512Mi"
        cpu: "400m"
    monitoring:
      enabled: true
      retention: "30d"
    security:
      network_policies: true
      pod_security: "restricted"
    
  production:
    namespace: "prod"
    replicas: 3
    resources:
      requests:
        memory: "512Mi"
        cpu: "400m"
      limits:
        memory: "1Gi"
        cpu: "800m"
    monitoring:
      enabled: true
      retention: "90d"
    security:
      network_policies: true
      pod_security: "restricted"
      audit_logging: true
```

### Keeper Configuration Template (`config/keeper-config.template.json`)

```json
{
  "version": "2",
  "server": "https://keepersecurity.com/api/v2/",
  "user": "your-keeper-email@example.com",
  "deviceToken": "device-token-from-keeper",
  "privateKey": "base64-encoded-private-key",
  "appData": {
    "appKey": "application-specific-key",
    "serverPublicKeyId": "server-public-key-id"
  },
  "clientVersion": "16.10.2"
}
```

---

## 🔄 Complete Usage Patterns & Workflows

### Foundation Setup Workflow

```bash
# 1. Complete foundation setup with intelligent port management
python scripts/install-k8s-keeper.py --allow-root --auto-ports

# 2. Verify installation components
kubectl get pods --all-namespaces
kubectl get secretstores,externalsecrets -n keeper-demo

# 3. Access dashboard and get token
echo "Dashboard: https://localhost:$(kubectl get service kubernetes-dashboard-nodeport -n kubernetes-dashboard -o jsonpath='{.spec.ports[0].nodePort}')"
kubectl create token admin-user -n kubernetes-dashboard --duration=8760h

# 4. Configure real Keeper secrets
python scripts/manage-secrets.py --list-records
python scripts/manage-secrets.py --create

# 5. Setup monitoring and security
python scripts/k8s-devops-automation.py --setup-monitoring
python scripts/k8s-devops-automation.py --setup-security

# 6. Setup backup system
python scripts/k8s-devops-automation.py --setup-backup
```

### Development Workflow

```bash
# 1. Setup development environment
python scripts/install-k8s-keeper.py --allow-root --dev-only --interactive-ports

# 2. Create development secrets
python scripts/manage-secrets.py --create --namespace dev

# 3. Deploy development application
python scripts/k8s-devops-automation.py --deploy --namespace dev

# 4. Monitor development environment
python scripts/manage-secrets.py --status --watch
python scripts/k8s-devops-automation.py --cluster-status
```

### Production Workflow

```bash
# 1. Setup production cluster with security focus
python scripts/install-k8s-keeper.py --allow-root --production-only --auto-ports

# 2. Import production secrets using templates
python scripts/advanced-secrets-manager.py --template postgres --record "PROD_DB_123" --name "postgres-prod" --namespace "production"
python scripts/advanced-secrets-manager.py --template docker-registry --record "DOCKER_REG_456" --name "registry-auth" --namespace "production"

# 3. Setup comprehensive monitoring and security
python scripts/k8s-devops-automation.py --setup-monitoring
python scripts/k8s-devops-automation.py --setup-security
python scripts/k8s-devops-automation.py --setup-backup

# 4. Deploy production application with full configuration
python scripts/k8s-devops-automation.py --deploy --namespace "production"

# 5. Setup CI/CD pipeline
python scripts/k8s-devops-automation.py --setup-cicd jenkins

# 6. Configure security policies and compliance
python scripts/advanced-secrets-manager.py --rotation-policy "30d" --namespace "production"
python scripts/k8s-devops-automation.py --security-scan --namespace "production"
python scripts/k8s-devops-automation.py --compliance-report

# 7. Setup automated backup schedule
python scripts/k8s-devops-automation.py --create-backup "production-initial"
```

### Multi-Environment Workflow

```bash
# Setup environments
for env in dev staging prod; do
  python scripts/install-k8s-keeper.py --allow-root --resume 06 --namespace $env
  python scripts/manage-secrets.py --deploy --record "APP_${env^^}_123" --name "app-secrets" --namespace $env
  python scripts/k8s-devops-automation.py --deploy --namespace $env
done

# Environment-specific monitoring
python scripts/k8s-devops-automation.py --cluster-status
python scripts/manage-secrets.py --status
```

---

## 🔍 Comprehensive Troubleshooting & Diagnostics

### Installation Issues

#### CRD Establishment Problems
**Symptoms**: External Secrets CRDs exist but show "Not Established"
**Diagnosis**:
```bash
# Check CRD status
kubectl get crd | grep external-secrets

# Check API resource recognition
kubectl api-resources | grep external-secrets

# Test functional capability
kubectl get secretstores --all-namespaces
```

**Recovery**:
```bash
# Method 1: Restart External Secrets controllers
kubectl rollout restart deployment/external-secrets -n external-secrets-system
kubectl rollout restart deployment/external-secrets-cert-controller -n external-secrets-system

# Method 2: Complete cleanup and reinstall
python scripts/install-k8s-keeper.py --clean-external-secrets --allow-root
python scripts/install-k8s-keeper.py --resume 05 --allow-root

# Method 3: Manual CRD validation
kubectl delete crd secretstores.external-secrets.io externalsecrets.external-secrets.io clustersecretstores.external-secrets.io
python scripts/install-k8s-keeper.py --resume 05 --allow-root
```

#### Port Conflicts & Network Issues
**Symptoms**: Services fail to start due to port conflicts
**Diagnosis**:
```bash
# Check port usage
netstat -tulpn | grep :8080
ss -tulpn | grep :8080

# Check Kind port mappings
docker ps | grep kind

# Verify cluster accessibility
kubectl cluster-info
```

**Recovery**:
```bash
# Method 1: Use automatic port selection
python scripts/install-k8s-keeper.py --allow-root --auto-ports

# Method 2: Specify custom ports
python scripts/install-k8s-keeper.py --allow-root --http-port 8090 --dashboard-port 30005

# Method 3: Interactive port analysis
python scripts/install-k8s-keeper.py --allow-root --interactive-ports

# Method 4: Complete cluster recreation
python scripts/install-k8s-keeper.py --uninstall-cluster --allow-root
python scripts/install-k8s-keeper.py --resume 03 --allow-root
```

#### Permission & Authentication Issues
**Symptoms**: Docker or kubectl permission denied
**Diagnosis**:
```bash
# Check user groups
groups $USER

# Check Docker daemon
systemctl status docker
docker info

# Check kubectl configuration
kubectl config current-context
kubectl auth can-i '*' '*'
```

**Recovery**:
```bash
# Method 1: Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Method 2: Use root installation
python scripts/install-k8s-keeper.py --allow-root

# Method 3: Fix kubectl configuration
kubectl config use-context kind-keeper-cluster
```

### Secrets Management Issues

#### Secret Sync Errors
**Symptoms**: ExternalSecret shows "SecretSyncedError" status
**Diagnosis**:
```bash
# Check SecretStore status
kubectl describe secretstore keeper-secretstore -n keeper-demo

# Verify KSM configuration
kubectl get secret keeper-config-secret -n keeper-demo -o yaml

# Test KSM CLI connectivity
ksm secret list

# Check External Secrets logs
kubectl logs -n external-secrets-system deployment/external-secrets --tail=50
```

**Recovery**:
```bash
# Method 1: Verify record UID and properties
python scripts/manage-secrets.py --list-records
ksm secret get "YOUR_RECORD_UID"

# Method 2: Recreate ExternalSecret
kubectl delete externalsecret keeper-example-secret -n keeper-demo
python scripts/manage-secrets.py --deploy --record "VALID_UID" --name "test-secret"

# Method 3: Reconfigure KSM integration
python scripts/install-k8s-keeper.py --resume 06 --allow-root --ksm-token "NEW_TOKEN"
```

#### Record Not Found Errors
**Symptoms**: "Record not found in Keeper vault"
**Diagnosis**:
```bash
# List available records
python scripts/manage-secrets.py --list-records

# Search for specific records
python scripts/manage-secrets.py --search "database"

# Test direct KSM access
ksm secret list
ksm secret get "SUSPECTED_UID"
```

**Recovery**:
```bash
# Method 1: Use correct record UID
python scripts/manage-secrets.py --list-records
python scripts/manage-secrets.py --deploy --record "CORRECT_UID" --name "secret-name"

# Method 2: Verify KSM configuration and permissions
kubectl get secret keeper-config-secret -n keeper-demo -o jsonpath='{.data.config}' | base64 -d

# Method 3: Reconfigure with correct Keeper credentials
python scripts/install-k8s-keeper.py --resume 06 --allow-root --ksm-config "/path/to/correct/config.json"
```

### Application Deployment Issues

#### Deployment Failures
**Symptoms**: Pods stuck in Pending, CrashLoopBackOff, or ImagePullBackOff
**Diagnosis**:
```bash
# Check pod status and events
kubectl get pods -o wide
kubectl describe pod POD_NAME

# Check deployment status
kubectl get deployments
kubectl describe deployment DEPLOYMENT_NAME

# Check resource availability
kubectl describe nodes
kubectl top nodes
```

**Recovery**:
```bash
# Method 1: Fix resource constraints
python scripts/k8s-devops-automation.py --scale DEPLOYMENT:NAMESPACE:1

# Method 2: Update deployment configuration
python scripts/k8s-devops-automation.py --update DEPLOYMENT:NAMESPACE:IMAGE:TAG

# Method 3: Rollback to working version
python scripts/k8s-devops-automation.py --rollback DEPLOYMENT:NAMESPACE

# Method 4: Complete redeployment
python scripts/k8s-devops-automation.py --delete-deployment DEPLOYMENT:NAMESPACE
python scripts/k8s-devops-automation.py --deploy
```

#### Service Connectivity Issues
**Symptoms**: Cannot access services or dashboard
**Diagnosis**:
```bash
# Check service status
kubectl get services -o wide
kubectl describe service SERVICE_NAME

# Check endpoints
kubectl get endpoints

# Check NodePort accessibility
kubectl get service kubernetes-dashboard-nodeport -n kubernetes-dashboard
```

**Recovery**:
```bash
# Method 1: Port forward for temporary access
kubectl port-forward service/kubernetes-dashboard 8443:443 -n kubernetes-dashboard

# Method 2: Recreate NodePort services
python scripts/install-k8s-keeper.py --resume 04 --allow-root

# Method 3: Check firewall and network configuration
sudo ufw status
iptables -L -n
```

### Monitoring & Backup Issues

#### Monitoring Stack Problems
**Symptoms**: Prometheus, Grafana, or Alertmanager not accessible
**Diagnosis**:
```bash
# Check monitoring pods
kubectl get pods -n monitoring

# Check services and endpoints
kubectl get services -n monitoring
kubectl get endpoints -n monitoring

# Check persistent volumes
kubectl get pv,pvc -n monitoring
```

**Recovery**:
```bash
# Method 1: Restart monitoring stack
python scripts/k8s-devops-automation.py --setup-monitoring

# Method 2: Check specific component logs
kubectl logs -n monitoring deployment/prometheus-server
kubectl logs -n monitoring deployment/grafana

# Method 3: Access via port forwarding
kubectl port-forward -n monitoring service/prometheus-server 9090:80
kubectl port-forward -n monitoring service/grafana 3000:80
```

#### Backup System Issues
**Symptoms**: Velero backups failing or not completing
**Diagnosis**:
```bash
# Check Velero status
kubectl get backups -n velero
kubectl get restores -n velero

# Check Velero logs
kubectl logs -n velero deployment/velero

# Check backup storage
python scripts/k8s-devops-automation.py --list-backups
```

**Recovery**:
```bash
# Method 1: Recreate backup system
python scripts/k8s-devops-automation.py --setup-backup

# Method 2: Manual backup creation
python scripts/k8s-devops-automation.py --create-backup "manual-$(date +%Y%m%d)"

# Method 3: Check storage configuration
kubectl describe backupstoragelocation default -n velero
```

---

## 📚 Advanced Features & API Extensions

### Intelligent Port Management System

The suite includes a sophisticated port management system that provides conflict-free service exposure:

#### Port Analysis & Selection
- **Real-time Scanning**: Automated port availability detection across multiple ranges
- **Conflict Resolution**: Intelligent alternative port suggestions with user confirmation
- **Service-aware Assignment**: Context-aware port assignment based on service type
- **Range Optimization**: Optimized scanning of common service port ranges

#### Port Management Methods

##### 1. Automatic Selection (`--auto-ports`)
```bash
python scripts/install-k8s-keeper.py --allow-root --auto-ports
```
- Scans system for optimal port availability
- Uses intelligent fallback algorithms
- Provides conflict-free port assignments
- Minimizes user intervention

##### 2. Interactive Selection (`--interactive-ports`)
```bash
python scripts/install-k8s-keeper.py --allow-root --interactive-ports
```
- Comprehensive port analysis dashboard
- User-guided port selection with recommendations
- Real-time conflict detection and warnings
- Educational port usage information

##### 3. Custom Port Assignment
```bash
python scripts/install-k8s-keeper.py --allow-root --http-port 8090 --dashboard-port 30005
```
- Precise port control for specific requirements
- Validation and conflict detection
- Override capabilities with user confirmation

#### Port Analysis Categories
- **Web Services (8000-8100)**: HTTP alternatives and reverse proxies
- **Development (3000-4100)**: Framework defaults and development servers
- **Kubernetes NodePort (30000-32767)**: Service exposure range
- **System Services**: Database, cache, and infrastructure service ports

### Advanced State Management & Resume System

#### State Persistence Architecture
- **JSON-based State Storage**: Comprehensive installation state tracking
- **Stage-based Checkpoints**: Granular resume capability from any installation stage
- **Configuration Preservation**: User choices and port selections preserved across resume
- **Error Context Preservation**: Detailed error information for troubleshooting

#### Resume Capabilities
```bash
# Resume from specific stage after failure
python scripts/install-k8s-keeper.py --resume 05 --allow-root

# Skip problematic stages
python scripts/install-k8s-keeper.py --allow-root --skip-stages "04,06"

# Resume with different configuration
python scripts/install-k8s-keeper.py --resume 03 --allow-root --interactive-ports

# Debug resume with state inspection
python scripts/install-k8s-keeper.py --resume 05 --allow-root --debug --work-dir /custom/path
```

#### State Management Features
- **Atomic Operations**: Each stage completion atomically updates state
- **Rollback Safety**: Safe rollback to previous successful state
- **Configuration Versioning**: Track configuration changes across resume attempts
- **Progress Tracking**: Visual progress indication with stage completion status

### Comprehensive Error Recovery & Diagnostics

#### Multi-level Error Recovery
1. **Automatic Recovery**: Self-healing mechanisms for common issues
2. **Guided Recovery**: Step-by-step user guidance for complex issues
3. **Manual Recovery**: Advanced recovery procedures for edge cases

#### Advanced Diagnostic Systems
- **Component Health Checking**: Real-time status of all system components
- **Resource Validation**: Kubernetes resource existence and configuration validation
- **Network Connectivity Testing**: Port accessibility and service reachability verification
- **Integration Testing**: End-to-end integration validation

#### Recovery Command Examples
```bash
# Comprehensive diagnostic run
python scripts/install-k8s-keeper.py --allow-root --debug --debug-output full-diagnostic.log

# Specific component cleanup and recovery
python scripts/install-k8s-keeper.py --clean-external-secrets --allow-root
python scripts/install-k8s-keeper.py --resume 05 --allow-root

# Complete system recovery
python scripts/install-k8s-keeper.py --uninstall-all --allow-root
python scripts/install-k8s-keeper.py --allow-root --auto-ports

# Network troubleshooting
python scripts/install-k8s-keeper.py --allow-root --interactive-ports --debug
```

---

## 🛡️ Security Architecture & Compliance

### Security-First Design Philosophy
- **Principle of Least Privilege**: Minimal required permissions for all operations
- **Defense in Depth**: Multiple security layers across all components
- **Zero Trust Network**: Assume breach security model implementation
- **Compliance by Design**: Built-in compliance with security frameworks

### Security Stack Components

#### Runtime Security (Falco)
- **Behavioral Monitoring**: Real-time detection of suspicious container behavior
- **Rule Engine**: Customizable security rules for application-specific threats
- **Alert Integration**: Integration with monitoring and notification systems
- **Forensic Capabilities**: Detailed security event logging and analysis

#### Policy Enforcement (OPA Gatekeeper)
- **Admission Control**: Kubernetes admission controller for policy enforcement
- **Custom Policies**: Organization-specific security and compliance policies
- **Constraint Templates**: Reusable policy templates for common scenarios
- **Violation Reporting**: Comprehensive policy violation reporting

#### Network Security
- **Micro-segmentation**: Network policies for application isolation
- **Traffic Control**: Ingress and egress traffic filtering
- **Service Mesh Ready**: Istio integration capabilities
- **Zero Trust Networking**: Identity-based network access control

#### Vulnerability Management
- **Image Scanning**: Container image vulnerability assessment
- **Dependency Analysis**: Security analysis of application dependencies
- **Compliance Scanning**: CIS Kubernetes Benchmark compliance checking
- **Continuous Monitoring**: Ongoing security posture assessment

### Compliance Framework Support
- **SOC 2**: Service Organization Control 2 compliance
- **PCI DSS**: Payment Card Industry Data Security Standard
- **HIPAA**: Health Insurance Portability and Accountability Act
- **CIS Benchmarks**: Center for Internet Security Kubernetes Benchmarks

---

## 🚀 API Reference for Unified Integration

### Core API Classes & Methods

#### Installation API (`KubernetesKeeperInstaller`)
```python
class KubernetesKeeperInstaller:
    def __init__(self, config: BuildConfig)
    def run(self) -> bool
    def _load_state(self) -> BuildState
    def _save_state(self) -> None
    def _show_usage_info(self) -> None
    
    # Stage management
    def execute_stage(self, stage_name: str) -> StageResult
    def resume_from_stage(self, stage_id: str) -> bool
    def skip_stages(self, stage_list: List[str]) -> None
```

#### Secrets Management API (`SecretManager`)
```python
class SecretManager:
    def create_external_secret(self, config: SecretConfig) -> OperationResult
    def list_secrets(self, namespace: Optional[str] = None) -> List[Dict[str, Any]]
    def delete_secret(self, secret_name: str, namespace: str) -> OperationResult
    def sync_secret(self, secret_name: str, namespace: str) -> OperationResult
    def get_secret_status(self, secret_name: str, namespace: str) -> Dict[str, Any]
    def _wait_for_secret_sync(self, secret_name: str, namespace: str) -> bool
```

#### Deployment Management API (`DeploymentManager`)
```python
class DeploymentManager:
    def create_deployment(self, config: DeploymentConfig) -> OperationResult
    def list_deployments(self, namespace: Optional[str] = None) -> List[Dict[str, Any]]
    def scale_deployment(self, name: str, namespace: str, replicas: int) -> OperationResult
    def update_deployment(self, name: str, namespace: str, image: str, tag: str) -> OperationResult
    def rollback_deployment(self, name: str, namespace: str, revision: Optional[int] = None) -> OperationResult
    def delete_deployment(self, name: str, namespace: str) -> OperationResult
    def get_deployment_status(self, name: str, namespace: str) -> Dict[str, Any]
```

#### Monitoring Management API (`MonitoringManager`)
```python
class MonitoringManager:
    def setup_monitoring_stack(self, config: MonitoringConfig) -> OperationResult
    def get_cluster_metrics(self) -> Dict[str, Any]
    def get_application_health(self, namespace: Optional[str] = None) -> Dict[str, Any]
    def create_custom_dashboard(self, name: str, queries: List[str]) -> OperationResult
    def setup_alerts(self, rules: List[Dict[str, Any]]) -> OperationResult
    def get_monitoring_status(self) -> Dict[str, Any]
```

#### Backup Management API (`BackupManager`)
```python
class BackupManager:
    def setup_backup_system(self, config: BackupConfig) -> OperationResult
    def create_backup(self, name: Optional[str] = None, namespaces: Optional[List[str]] = None) -> OperationResult
    def list_backups(self) -> List[Dict[str, Any]]
    def restore_backup(self, backup_name: str, namespaces: Optional[List[str]] = None) -> OperationResult
    def delete_backup(self, backup_name: str) -> OperationResult
    def get_backup_status(self, backup_name: str) -> Dict[str, Any]
```

#### Security Management API (`SecurityManager`)
```python
class SecurityManager:
    def setup_security_tools(self) -> OperationResult
    def scan_vulnerabilities(self, namespace: Optional[str] = None) -> Dict[str, Any]
    def generate_compliance_report(self) -> Dict[str, Any]
    def setup_network_policies(self, namespace: str) -> OperationResult
    def configure_rbac(self, policies: List[Dict[str, Any]]) -> OperationResult
```

### Data Models & Return Types

#### OperationResult
```python
@dataclass
class OperationResult:
    status: OperationStatus  # SUCCESS, FAILED, WARNING, PENDING, etc.
    message: str            # Human-readable status message
    data: Optional[Dict[str, Any]] = None    # Additional response data
    error: Optional[str] = None              # Detailed error information
    timestamp: datetime = field(default_factory=datetime.now)
```

#### OperationStatus Enum
```python
class OperationStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"
    SYNCED = "synced"
    ERROR = "error"
    WARNING = "warning"
    DEGRADED = "degraded"
```

### REST API Mapping for Unified Interface

#### Installation Endpoints
```
POST /api/v1/install              # Start installation
GET  /api/v1/install/status       # Get installation status
POST /api/v1/install/resume       # Resume from stage
DELETE /api/v1/install            # Uninstall components
```

#### Secrets Management Endpoints
```
GET    /api/v1/secrets                    # List secrets
POST   /api/v1/secrets                    # Create secret
GET    /api/v1/secrets/{name}             # Get secret details
PUT    /api/v1/secrets/{name}             # Update secret
DELETE /api/v1/secrets/{name}             # Delete secret
GET    /api/v1/secrets/{name}/status      # Get sync status
POST   /api/v1/secrets/{name}/sync        # Force sync
```

#### Deployment Management Endpoints
```
GET    /api/v1/deployments                # List deployments
POST   /api/v1/deployments                # Create deployment
GET    /api/v1/deployments/{name}         # Get deployment
PUT    /api/v1/deployments/{name}         # Update deployment
DELETE /api/v1/deployments/{name}         # Delete deployment
POST   /api/v1/deployments/{name}/scale   # Scale deployment
POST   /api/v1/deployments/{name}/rollback # Rollback deployment
```

#### Monitoring Endpoints
```
GET  /api/v1/monitoring/status     # Monitoring stack status
POST /api/v1/monitoring/setup      # Setup monitoring
GET  /api/v1/monitoring/metrics    # Get cluster metrics
GET  /api/v1/monitoring/health     # Application health
POST /api/v1/monitoring/dashboards # Create dashboard
POST /api/v1/monitoring/alerts     # Configure alerts
```

#### Backup Endpoints
```
GET    /api/v1/backups           # List backups
POST   /api/v1/backups           # Create backup
GET    /api/v1/backups/{name}    # Get backup details
DELETE /api/v1/backups/{name}    # Delete backup
POST   /api/v1/backups/{name}/restore # Restore backup
```

#### Security Endpoints
```
GET  /api/v1/security/status          # Security stack status
POST /api/v1/security/setup           # Setup security tools
POST /api/v1/security/scan            # Run vulnerability scan
GET  /api/v1/security/compliance      # Get compliance report
POST /api/v1/security/policies        # Configure policies
```

---

## 📖 Getting Started Examples

### Beginner: Automated Setup
```bash
# Complete automated setup
./quick-start.sh

# Verify everything is working
python scripts/manage-secrets.py --status
python scripts/k8s-devops-automation.py --cluster-status

# Create your first secret
python scripts/manage-secrets.py --create

# Access the dashboard
echo "Dashboard: https://localhost:30001"
kubectl create token admin-user -n kubernetes-dashboard --duration=8760h
```

### Intermediate: Custom Environment
```bash
# Custom installation with monitoring and security
python scripts/install-k8s-keeper.py --allow-root --auto-ports
python scripts/k8s-devops-automation.py --setup-monitoring
python scripts/k8s-devops-automation.py --setup-security

# Deploy application with secrets
python scripts/manage-secrets.py --template postgres --record "DB_123" --name "app-db"
python scripts/k8s-devops-automation.py --deploy

# Setup backup and CI/CD
python scripts/k8s-devops-automation.py --setup-backup
python scripts/k8s-devops-automation.py --setup-cicd jenkins
```

### Advanced: Production-Ready Environment
```bash
# Production installation with full security
python scripts/install-k8s-keeper.py --allow-root --production-only --interactive-ports

# Import production secrets with templates
python scripts/advanced-secrets-manager.py --import-folder "production-secrets"
python scripts/advanced-secrets-manager.py --template docker-registry --record "REG_456" --name "registry-auth"

# Setup comprehensive monitoring, security, and backup
python scripts/k8s-devops-automation.py --setup-monitoring
python scripts/k8s-devops-automation.py --setup-security
python scripts/k8s-devops-automation.py --setup-backup

# Deploy with full production configuration
python scripts/k8s-devops-automation.py --deploy --namespace "production"

# Setup GitOps and compliance
python scripts/k8s-devops-automation.py --setup-cicd argocd
python scripts/advanced-secrets-manager.py --rotation-policy "30d" --namespace "production"
python scripts/k8s-devops-automation.py --compliance-report

# Automated backup schedule
python scripts/k8s-devops-automation.py --create-backup "production-$(date +%Y%m%d)"
```

---

## 📚 Documentation & Support

### Documentation Structure
- **Installation Guide**: Complete setup instructions with troubleshooting
- **API Reference**: Comprehensive API documentation for integration
- **User Guide**: Step-by-step workflows for common operations
- **Architecture Guide**: Technical architecture and design decisions
- **Security Guide**: Security best practices and compliance information
- **Troubleshooting Guide**: Common issues and resolution procedures

### Support Resources
- **Examples Directory**: Real-world usage examples and templates
- **Configuration Templates**: Pre-configured templates for common scenarios
- **Diagnostic Tools**: Built-in diagnostic and troubleshooting utilities
- **Recovery Procedures**: Comprehensive recovery and disaster recovery guides

### Community & Contributions
This suite follows a modular architecture designed for extensibility and community contributions. Each script maintains clear API boundaries and follows consistent patterns for easy customization and extension.

---

## 📄 License & Legal

[Your License Information Here]

---

*This comprehensive Kubernetes + Keeper Secrets Manager suite provides enterprise-grade DevOps automation with integrated secrets management, from initial setup through production deployment and ongoing maintenance. The modular architecture and comprehensive API make it suitable for both standalone use and integration into larger DevOps platforms.*
