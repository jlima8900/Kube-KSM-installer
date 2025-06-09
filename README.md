# Kubernetes + Keeper Secrets Manager Installer - Technical Documentation

## Overview

The Kubernetes + Keeper Installer (`install-k8s-keeper.py`) is a comprehensive, multi-stage Python script that automates the installation and configuration of a complete Kubernetes development environment with Keeper Secrets Manager integration. The script provides intelligent port management, robust error handling, state persistence, and comprehensive diagnostics.

---

## Quick Start & Usage Instructions

### Installation
```bash
# Basic installation (recommended for most users)
python install-k8s-keeper.py --allow-root

# Production-only installation (no development tools)
python install-k8s-keeper.py --allow-root --production-only

# Development-only installation (no External Secrets/Keeper)
python install-k8s-keeper.py --allow-root --dev-only

# Custom port configuration
python install-k8s-keeper.py --allow-root --interactive-ports
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

### Common Operations

#### Resume Failed Installation
```bash
# Resume from a specific stage
python install-k8s-keeper.py --resume 05 --allow-root
```

#### Uninstall Components
```bash
# Interactive uninstall menu
python install-k8s-keeper.py --uninstall --allow-root

# Complete removal
python install-k8s-keeper.py --uninstall-all --allow-root

# Clean up corrupted External Secrets
python install-k8s-keeper.py --clean-external-secrets --allow-root
```

#### Troubleshooting
```bash
# Check installation logs
cat /tmp/k8s-keeper-build/install.log

# Verify cluster status
kubectl cluster-info

# Check External Secrets CRDs
kubectl get crd | grep external-secrets
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

## Architecture & Design Philosophy

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

## Section-by-Section Technical Documentation

## 1. Configuration & Constants

### 1.1 Core Configuration Classes

#### `StageStatus` (Enum)
```python
class StageStatus(Enum):
    PENDING = "pending"
    RUNNING = "running" 
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"
```
**Purpose**: Defines the possible states of each installation stage for state management and flow control.

#### `StageResult` (Dataclass)
```python
@dataclass
class StageResult:
    status: StageStatus
    message: str
    data: Optional[Dict[str, Any]] = None
    error: Optional[str] = None
```
**Purpose**: Standardized return object for all stage operations containing status, descriptive message, optional data payload, and error details.

#### `BuildConfig` (Dataclass)
```python
@dataclass
class BuildConfig:
    allow_root: bool = False
    production_only: bool = False
    dev_only: bool = False
    skip_dashboard: bool = False
    ksm_token: Optional[str] = None
    ksm_config_file: Optional[str] = None
    resume_stage: Optional[str] = None
    skip_stages: List[str] = None
    work_dir: str = "/tmp/k8s-keeper-build"
    debug_mode: bool = False
    debug_output_file: Optional[str] = None
    reconfigure_mode: bool = False
    allow_destructive: bool = False
    auto_port_selection: Optional[bool] = None
    custom_ports: Optional[Dict[str, int]] = None
```
**Purpose**: Central configuration object that controls all aspects of the installation behavior, command-line argument processing, and user preferences.

**Key Fields**:
- **Security**: `allow_root`, `allow_destructive` - Control dangerous operations
- **Installation Modes**: `production_only`, `dev_only` - Control component selection
- **Port Management**: `auto_port_selection`, `custom_ports` - Control port configuration
- **State Management**: `resume_stage`, `skip_stages` - Control installation flow
- **Keeper Integration**: `ksm_token`, `ksm_config_file` - Keeper configuration options

#### `BuildState` (Dataclass)
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
**Purpose**: Persistent state object that tracks installation progress, stores configuration decisions, and enables resume functionality.

**Key Fields**:
- **Progress Tracking**: `stages_completed`, `current_stage` - Resume capability
- **Runtime State**: `cluster_ready`, `dashboard_token` - Component status
- **User Decisions**: `selected_ports`, `keeper_folders` - Preserve user choices

---

## 2. Logging System

### 2.1 ColoredFormatter Class
```python
class ColoredFormatter(logging.Formatter):
    COLORS = {
        'DEBUG': '\033[0;36m',    # Cyan
        'INFO': '\033[0;34m',     # Blue  
        'WARNING': '\033[1;33m',  # Yellow
        'ERROR': '\033[0;31m',    # Red
        'SUCCESS': '\033[0;32m',  # Green
        'STAGE': '\033[0;35m',    # Purple
    }
```
**Purpose**: Provides color-coded console output for improved readability and user experience.

**Features**:
- **Visual Hierarchy**: Different colors for different log levels
- **Stage Identification**: Special purple color for stage transitions
- **Error Emphasis**: Red for errors, yellow for warnings
- **Success Feedback**: Green for successful operations

### 2.2 Dual Logging System
```python
def setup_logging(work_dir: str) -> logging.Logger:
    # Console handler with colors
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    console_formatter = ColoredFormatter('%(levelname)s %(message)s')
    
    # File handler for debugging
    file_handler = logging.FileHandler(f"{work_dir}/install.log")
    file_handler.setLevel(logging.DEBUG)
    file_formatter = logging.Formatter('%(asctime)s %(levelname)s %(message)s')
```
**Purpose**: Creates both console and file logging for user feedback and debugging.

**Features**:
- **Console Logging**: Colored, INFO level, user-friendly
- **File Logging**: Detailed, DEBUG level, includes timestamps
- **Persistent Records**: Installation logs saved for troubleshooting

---

## 3. Port Management System

### 3.1 Port Detection Functions

#### `is_port_in_use(port: int, host: str = 'localhost') -> bool`
**Purpose**: Tests if a specific port is currently occupied using socket connection.
**Implementation**: Attempts TCP connection with 1-second timeout.
**Return**: `True` if port is in use, `False` if available.

#### `get_used_ports(port_range: Tuple[int, int]) -> List[int]`
**Purpose**: Scans a range of ports to identify which are currently in use.
**Implementation**: Iterates through range calling `is_port_in_use()`.
**Optimization**: Limits scans to 50 ports per range to prevent excessive scanning time.

#### `get_available_ports(port_range: Tuple[int, int], count: int = 5) -> List[int]`
**Purpose**: Finds available ports within a specified range.
**Implementation**: Returns first `count` available ports found.
**Use Case**: Provides alternatives when preferred ports are unavailable.

### 3.2 Intelligent Port Suggestion

#### `suggest_ports() -> Dict[str, int]`
**Purpose**: Automatically suggests optimal ports for all services based on availability.

**Default Port Preferences**:
- `http`: 8080 (standard alternate HTTP)
- `https`: 8443 (standard alternate HTTPS)
- `nodeport`: 30000 (Kubernetes NodePort range start)
- `dashboard`: 30001 (Sequential NodePort assignment)

**Fallback Logic**:
1. **Try Preferred**: Use default ports if available
2. **Find Alternatives**: Search appropriate ranges for alternatives
3. **Fallback Range**: Use any available port in 8000-9000 range
4. **Force Assignment**: Use preferred port with conflict warning

### 3.3 Interactive Port Management

#### `interactive_port_selection() -> Dict[str, int]`
**Purpose**: Provides comprehensive interactive port selection interface.

**Features**:
1. **Port Analysis Dashboard**: Shows used/available ports by category
2. **Smart Recommendations**: Displays suggested ports with availability status
3. **User Choice Menu**: 
   - Use suggested ports (recommended)
   - Customize port selection
   - Show detailed port analysis

#### `custom_port_selection(suggested: Dict[str, int]) -> Dict[str, int]`
**Purpose**: Allows manual override of any suggested port.

**Features**:
- **Per-Service Configuration**: Individual port selection for each service
- **Conflict Warnings**: Alerts when selecting in-use ports
- **Validation**: Ensures ports are within valid range (1-65535)
- **Default Preservation**: Press Enter to keep suggested ports

#### `show_detailed_port_analysis()`
**Purpose**: Comprehensive port analysis showing system-wide port usage.

**Analysis Categories**:
- **Web Services** (8000-8100): HTTP alternatives
- **Alt HTTP** (3000-3100): Development servers
- **Development** (4000-4100): Framework defaults
- **Kubernetes** (30000-32767): NodePort range
- **Common Services**: MySQL, PostgreSQL, MongoDB, Redis, etc.

**Output Format**:
- **Used Ports**: Shows occupied ports in each range
- **Available Ports**: Lists available alternatives
- **Service Status**: Shows common service port availability

---

## 4. Utility Functions

### 4.1 Command Execution

#### `run_command()` - Enhanced Command Runner
```python
def run_command(cmd: str, check: bool = True, capture_output: bool = True, 
                shell: bool = True, cwd: Optional[str] = None, 
                suppress_errors: bool = False) -> subprocess.CompletedProcess
```
**Purpose**: Centralized command execution with comprehensive error handling.

**Parameters**:
- `cmd`: Shell command to execute
- `check`: Whether to raise exception on non-zero exit
- `capture_output`: Whether to capture stdout/stderr
- `shell`: Whether to use shell interpretation
- `cwd`: Working directory for command execution
- `suppress_errors`: Whether to suppress error logging

**Features**:
- **Debug Logging**: All commands logged at DEBUG level
- **Error Context**: Detailed error reporting with command and output
- **Selective Suppression**: Can suppress expected errors (like existence checks)
- **Flexible Configuration**: Supports various execution contexts

#### `check_command_exists(command: str) -> bool`
**Purpose**: Tests if a command exists in system PATH.
**Implementation**: Uses `command -v` with error suppression.
**Use Case**: Determines which tools need installation.

#### `check_service_exists(check_cmd: str, service_name: str) -> bool`
**Purpose**: Generic service/resource existence checker.
**Implementation**: Runs check command with error suppression.
**Use Case**: Determines if Kubernetes resources or Helm releases exist.

### 4.2 File Operations

#### `download_file(url: str, dest: str) -> None`
**Purpose**: Downloads files from URLs with proper error handling.
**Features**:
- **Error Context**: Detailed error messages with URL
- **Permissions**: Sets executable permissions automatically
- **Exception Handling**: Converts download errors to RuntimeError

#### `prompt_choice(question: str, choices: List[str]) -> str`
**Purpose**: Interactive menu system for user choices.
**Features**:
- **Numbered Options**: Clear numbering for all choices
- **Input Validation**: Ensures valid selection
- **Error Recovery**: Handles invalid input gracefully
- **Interrupt Handling**: Manages KeyboardInterrupt exceptions

---

## 5. Stage Base Class Architecture

### 5.1 Stage Class Design
```python
class Stage:
    def __init__(self, name: str, description: str, config: BuildConfig, state: BuildState)
    def should_run(self) -> bool
    def execute(self) -> StageResult  # Abstract method
    def run(self) -> StageResult      # Template method
```

**Design Pattern**: Template Method Pattern
**Purpose**: Provides consistent stage execution framework with customizable implementation.

#### Core Methods:

##### `should_run() -> bool`
**Purpose**: Determines if stage should execute based on configuration and state.
**Logic**:
1. **Skip Check**: Stage in skip_stages list
2. **Completion Check**: Stage already completed
3. **Default**: Run if neither condition applies

##### `run() -> StageResult` (Template Method)
**Purpose**: Orchestrates stage execution with consistent logging and error handling.
**Flow**:
1. **Pre-execution Check**: Call `should_run()`
2. **Logging**: Log stage start with description
3. **Execution**: Call abstract `execute()` method
4. **State Management**: Update completion list on success
5. **Result Logging**: Log success/failure with details
6. **Exception Handling**: Convert exceptions to StageResult

##### `execute() -> StageResult` (Abstract)
**Purpose**: Stage-specific implementation logic.
**Contract**: Must return StageResult with appropriate status and message.

---

## 6. Stage Implementations

### 6.1 PrerequisitesStage

**Purpose**: Validates system prerequisites before installation begins.

**Checks Performed**:
1. **Root User Validation**: Prevents root execution unless explicitly allowed
2. **Docker Installation**: Verifies Docker is installed
3. **Docker Status**: Confirms Docker daemon is running
4. **Docker Permissions**: Tests user can execute Docker commands
5. **Python Version**: Ensures Python 3.7+ compatibility

**Error Conditions**:
- Running as root without `--allow-root` flag
- Docker not installed or not running
- User lacks Docker permissions (warning only)
- Python version incompatibility

**Success Criteria**: All checks pass with detailed status report.

### 6.2 ToolsInstallationStage

**Purpose**: Installs required Kubernetes and development tools.

**Tools Installed**:

#### kubectl (Kubernetes CLI)
- **Detection**: Uses `command -v kubectl`
- **Installation**: Downloads latest stable version from official repository
- **Location**: `/usr/local/bin/kubectl`
- **Permissions**: Sets executable permissions
- **Verification**: Tool added to tools_installed list

#### Kind (Kubernetes in Docker)
- **Detection**: Uses `command -v kind`
- **Version**: Fixed version v0.20.0 for stability
- **Source**: Official Kind GitHub releases
- **Installation**: Direct binary download and installation

#### Helm (Kubernetes Package Manager)
- **Detection**: Uses `command -v helm`
- **Installation**: Official get-helm-3 script
- **Method**: Automated script execution with bash

#### Go Programming Language (Development Mode Only)
- **Condition**: Skipped if `production_only` mode
- **Detection**: Uses `command -v go`
- **Version**: 1.21.5 (specific version for compatibility)
- **Installation**: Downloads, extracts, and installs to `/usr/local/go`
- **PATH Update**: Adds Go binary directory to current session PATH

#### KSM CLI (Keeper Secrets Manager CLI)
- **Condition**: Skipped if `dev_only` mode
- **Installation**: Uses pip3 package manager
- **Package**: `keeper-secrets-manager-cli`
- **Error Handling**: Non-fatal - warns on failure

**Installation Flow**:
1. **Detection Phase**: Check each tool individually
2. **Status Reporting**: Log "already installed" or "installing"
3. **Installation Phase**: Download and install missing tools
4. **Verification Phase**: Confirm successful installation
5. **Summary**: Report which tools were installed

**Error Handling**:
- **Fatal Errors**: kubectl, kind, helm installation failures
- **Warnings**: Go, KSM CLI installation failures (non-critical)
- **Permission Handling**: Uses sudo for non-root installations

### 6.3 ClusterSetupStage

**Purpose**: Creates Kubernetes cluster with intelligent port configuration.

#### Port Configuration System

##### `_get_port_configuration() -> Dict[str, int]`
**Decision Matrix**:
1. **Previous Selection**: Use saved ports from previous run
2. **Command Line**: Use custom ports from CLI arguments
3. **Automatic Mode**: Use `suggest_ports()` for optimal selection
4. **Interactive Mode**: Use full interactive selection interface
5. **Default Prompt**: Ask user to choose method

##### `_prompt_port_selection_method() -> Dict[str, int]`
**User Options**:
1. **Automatic Selection**: Recommended for most users
2. **Interactive Selection**: Full control with recommendations
3. **Show Analysis First**: Educational port analysis before selection

#### Cluster Creation Process

##### Kind Cluster Configuration
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: keeper-cluster
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: {selected_http_port}
  - containerPort: 443
    hostPort: {selected_https_port}
  - containerPort: 30000
    hostPort: {selected_nodeport}
  - containerPort: 30001
    hostPort: {selected_dashboard_port}
```

**Configuration Features**:
- **Single Node**: Control-plane only for development
- **Ingress Ready**: Labels node for ingress controller
- **Port Mapping**: Maps container ports to selected host ports
- **Dynamic Configuration**: Uses user-selected or auto-detected ports

##### Cluster Creation Flow
1. **Existence Check**: Verify cluster doesn't already exist
2. **Port Selection**: Execute port configuration logic
3. **Config Generation**: Create Kind configuration file
4. **Cluster Creation**: Execute `kind create cluster`
5. **Verification**: Confirm cluster accessibility
6. **State Update**: Mark cluster as ready for subsequent stages

**Error Conditions**:
- Port conflicts during creation
- Kind installation failures
- Cluster startup timeouts
- Network connectivity issues

### 6.4 DashboardStage

**Purpose**: Installs Kubernetes Dashboard with admin access and custom port configuration.

#### Installation Components

##### Dashboard Deployment
- **Source**: Official Kubernetes Dashboard v2.7.0
- **Method**: Direct YAML application from GitHub
- **URL**: `https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml`
- **Namespace**: `kubernetes-dashboard`

##### Admin Service Account Creation
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

**Purpose**: Creates service account with full cluster administration privileges.

##### NodePort Service Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard-nodeport
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
  - port: 443
    targetPort: 8443
    nodePort: {selected_dashboard_port}
    protocol: TCP
  selector:
    k8s-app: kubernetes-dashboard
```

**Features**:
- **External Access**: Exposes dashboard on host port
- **HTTPS Only**: Maintains security with TLS
- **Custom Port**: Uses user-selected or auto-detected port

#### Token Management

##### `_get_admin_token() -> str`
**Purpose**: Generates long-lived authentication token for dashboard access.

**Primary Method** (Kubernetes 1.24+):
```bash
kubectl create token admin-user -n kubernetes-dashboard --duration=8760h
```

**Fallback Method** (Older Kubernetes):
```bash
kubectl get secret -n kubernetes-dashboard -o jsonpath='{.items[?(@.metadata.annotations.kubernetes\.io/service-account\.name=="admin-user")].data.token}' | base64 -d
```

**Token Duration**: 8760 hours (1 year) for development convenience.

#### Installation Flow
1. **Skip Check**: Honor `--skip-dashboard` flag
2. **Existence Check**: Verify dashboard not already installed
3. **Dashboard Installation**: Apply official YAML manifests
4. **Readiness Wait**: Wait for deployment to become available
5. **Service Account**: Create admin user with cluster privileges
6. **NodePort Service**: Create external access service
7. **Token Generation**: Generate and store access token
8. **State Persistence**: Save token for post-installation display

### 6.5 ExternalSecretsStage

**Purpose**: Installs External Secrets Operator with advanced CRD validation and recovery mechanisms.

#### Advanced CRD Management

##### `_check_crds_ready() -> bool`
**Purpose**: Validates CRDs are properly installed and established.
**CRDs Checked**:
- `secretstores.external-secrets.io`
- `externalsecrets.external-secrets.io`
- `clustersecretstores.external-secrets.io`

**Validation Criteria**: CRD must show "Established" status in kubectl output.

##### `_wait_for_crds(timeout_seconds: int = 150) -> bool`
**Purpose**: Advanced CRD waiting with recovery mechanisms.

**Features**:
- **Progress Tracking**: Counts ready vs missing CRDs
- **Status Reporting**: Logs progress every 30 seconds
- **Recovery Mechanism**: Attempts controller restart at 90 seconds
- **Manual Validation**: Falls back to functional testing if status fails

##### `_force_crd_refresh()`
**Purpose**: Attempts to fix CRD establishment issues.
**Method**: Restarts External Secrets controller deployments:
- `deployment/external-secrets`
- `deployment/external-secrets-cert-controller`

##### `_manual_crd_validation() -> bool`
**Purpose**: Functional validation when status checks fail.

**Validation Tests**:
1. **API Resource Check**: Verify Kubernetes API recognizes External Secrets
2. **SecretStore Dry-Run**: Test SecretStore creation capability
3. **ExternalSecret Dry-Run**: Test ExternalSecret creation capability
4. **Resource Listing**: Verify ability to list all External Secrets resources

**Fallback Logic**: Proceeds with installation if CRDs are functionally ready despite status.

#### Installation Process

##### Helm Chart Installation
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
    -n external-secrets-system \
    --create-namespace \
    --wait \
    --timeout=5m
```

**Features**:
- **Automatic Namespace**: Creates namespace if not exists
- **Wait for Ready**: Blocks until all components are ready
- **Timeout Protection**: Fails after 5 minutes if not ready

##### Corruption Recovery System

##### `_complete_external_secrets_cleanup()`
**Purpose**: Complete removal of corrupted External Secrets installation.

**Cleanup Process**:
1. **Helm Removal**: Uninstall Helm release
2. **Namespace Deletion**: Remove entire namespace and all resources
3. **Namespace Wait**: Wait for complete namespace deletion
4. **CRD Cleanup**: Manually remove all External Secrets CRDs
5. **Stabilization**: Wait for cleanup to complete

**CRDs Removed**:
- `secretstores.external-secrets.io`
- `externalsecrets.external-secrets.io`
- `clustersecretstores.external-secrets.io`
- `clusterexternalsecrets.external-secrets.io`
- `pushsecrets.external-secrets.io`

#### Diagnostic System

##### `_show_diagnostics()`
**Purpose**: Comprehensive troubleshooting information display.

**Diagnostic Checks**:
1. **Namespace Status**: Verify namespace exists
2. **Helm Release**: Confirm Helm installation
3. **Pod Status**: Show controller pod states
4. **CRD Status**: Individual CRD establishment status

**Output Format**:
- ✅ **Success Indicators**: Green checkmarks for working components
- ❌ **Failure Indicators**: Red X marks for missing components
- 📋 **Information Blocks**: Detailed status information

#### Installation Flow
1. **Skip Check**: Honor `dev_only` mode
2. **Readiness Check**: Verify CRDs not already established
3. **Corruption Detection**: Check for incomplete installations
4. **Cleanup Decision**: Offer or perform automatic cleanup
5. **Fresh Installation**: Install via Helm with full validation
6. **CRD Validation**: Advanced CRD establishment verification
7. **Recovery Attempts**: Try multiple recovery mechanisms
8. **Functional Validation**: Fall back to functional testing
9. **Status Reporting**: Comprehensive success/failure reporting

### 6.6 KeeperIntegrationStage

**Purpose**: Configures Keeper Secrets Manager integration with External Secrets Operator.

#### CRD Validation

##### `_verify_external_secrets_ready() -> bool`
**Purpose**: Functional validation of External Secrets readiness.

**Validation Strategy**:
1. **Functional Tests (Primary)**:
   - API resource recognition test
   - Resource listing capability test
   - Actual functionality verification

2. **Status Check (Fallback)**:
   - Traditional "Established" status check
   - Non-blocking if functional tests pass

**Design Philosophy**: Prioritizes functional capability over status indicators.

#### Configuration Management

##### KSM Configuration Sources

###### `_generate_config_from_token(token: str) -> str`
**Purpose**: Generates KSM configuration from Keeper one-time token.
**Process**:
1. Execute `ksm init k8s '{token}'`
2. Parse kubectl secret YAML output
3. Extract base64-encoded configuration
4. Return configuration for Kubernetes secret creation

###### `_load_config_from_file(config_file: str) -> str`
**Purpose**: Loads existing KSM configuration from file.
**Process**:
1. Read JSON configuration file
2. Base64 encode configuration content
3. Return encoded configuration

###### `_interactive_config_setup() -> Optional[str]`
**Purpose**: Interactive configuration wizard.

**User Options**:
1. **One-time Token**: Generate config from Keeper token
2. **Existing Config**: Load from existing file
3. **Skip Configuration**: Create template only

#### Kubernetes Secret Creation

##### `_create_config_secret(config_data: str) -> None`
**Purpose**: Creates Kubernetes secret containing KSM configuration.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: keeper-config-secret
  namespace: keeper-demo
type: Opaque
data:
  config: {base64_encoded_config}
```

#### Folder Discovery System

##### `_discover_folders() -> List[Dict[str, str]]`
**Purpose**: Discovers available Keeper folders for SecretStore configuration.

**Process**:
1. **Config Extraction**: Get KSM config from Kubernetes secret
2. **Temporary File**: Create temporary config file
3. **Environment Setup**: Set KSM_CONFIG_FILE environment variable
4. **Folder Listing**: Execute `ksm folder list`
5. **Output Parsing**: Parse folder list output
6. **Cleanup**: Remove temporary files

**Output Format**:
```python
[
    {"id": "folder_uid", "title": "Folder Name"},
    ...
]
```

##### `_select_folder(folders: List[Dict[str, str]]) -> Optional[str]`
**Purpose**: Interactive folder selection for SecretStore scope.

**Selection Options**:
1. **Available Folders**: Show discovered folders with descriptions
2. **Manual Entry**: Allow manual folder ID entry
3. **Root Level**: No folder restriction (vault-wide access)

#### SecretStore Creation

##### `_create_secretstore(folder_id: Optional[str]) -> None`
**Purpose**: Creates Keeper SecretStore resource.

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: keeper-secretstore
  namespace: keeper-demo
spec:
  provider:
    keepersecurity:
      authRef:
        name: keeper-config-secret
        key: config
      folderID: "{folder_id}"  # Optional
```

**Features**:
- **Authentication**: References Kubernetes secret containing KSM config
- **Folder Scoping**: Optional folder restriction for security
- **Namespace Isolation**: Created in dedicated keeper-demo namespace

#### Example ExternalSecret

##### `_create_example_externalsecret() -> None`
**Purpose**: Creates template ExternalSecret for user customization.

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: keeper-example-secret
  namespace: keeper-demo
spec:
  refreshInterval: 30s
  secretStoreRef:
    kind: SecretStore
    name: keeper-secretstore
  target:
    name: keeper-synced-secret
    creationPolicy: Owner
  data:
  - secretKey: example-key
    remoteRef:
      key: "PLACEHOLDER_RECORD_UID"
      property: "login"
```

**Features**:
- **Placeholder UID**: Users must replace with actual record UIDs
- **Documentation**: Inline comments with configuration instructions
- **Refresh Interval**: 30-second sync frequency
- **Target Secret**: Creates `keeper-synced-secret` in same namespace

#### Integration Flow
1. **CRD Verification**: Ensure External Secrets is functional
2. **Namespace Creation**: Create `keeper-demo` namespace
3. **Configuration Acquisition**: Get KSM config via interactive/automated means
4. **Secret Creation**: Store KSM config in Kubernetes secret
5. **Folder Discovery**: Enumerate available Keeper folders
6. **Folder Selection**: Interactive or automated folder selection
7. **SecretStore Creation**: Create External Secrets SecretStore
8. **Example Creation**: Create template ExternalSecret for user customization

### 6.7 ValidationStage

**Purpose**: Comprehensive validation of entire installation.

#### Validation Checks

##### Cluster Accessibility
- **Test**: `kubectl cluster-info`
- **Purpose**: Verify Kubernetes cluster is responsive
- **Failure Impact**: Installation marked as failed

##### Dashboard Validation (Conditional)
- **Condition**: Not skipped via `--skip-dashboard`
- **Test**: Check `kubernetes-dashboard` deployment exists
- **Purpose**: Verify dashboard installation success
- **Failure Impact**: Warning only

##### External Secrets Validation (Conditional)
- **Condition**: Not in `dev_only` mode
- **Test**: Check `external-secrets` deployment exists
- **Purpose**: Verify External Secrets Operator installation
- **Failure Impact**: Warning only

##### Keeper Integration Validation (Optional)
- **Test**: Check `keeper-secretstore` resource exists
- **Purpose**: Verify Keeper integration configuration
- **Failure Impact**: Informational only

#### Validation Flow
1. **Critical Checks**: Cluster accessibility (failure blocks completion)
2. **Component Checks**: Individual component validation (warnings only)
3. **Integration Checks**: End-to-end integration validation (informational)
4. **Summary Report**: Comprehensive status report

---

## 7. Uninstall System

### 7.1 Uninstall Architecture

#### Granular Uninstall Options
- **Complete Removal**: Everything including tools and cluster
- **Cluster Only**: Kubernetes cluster and associated resources
- **Dashboard Only**: Just the Kubernetes dashboard
- **Tools Only**: Installed command-line tools
- **External Secrets Only**: Corrupted External Secrets cleanup

#### Interactive Uninstall Menu
```python
def interactive_uninstall_menu():
    """Interactive uninstall menu with numbered options"""
    1) Everything (cluster + tools + dashboard + examples)
    2) Just the Kubernetes cluster (includes dashboard)
    3) Just the dashboard
    4) Just installed tools (kubectl, kind, helm, go)
    5) Cancel
```

### 7.2 Uninstall Functions

#### `uninstall_all()`
**Purpose**: Complete system removal.

**Removal Process**:
1. **Kubernetes Resources**: Remove all namespaces and resources
2. **Cluster Removal**: Delete Kind cluster
3. **Tool Removal**: Remove all installed command-line tools
4. **Docker Cleanup**: Remove Kind-related Docker resources
5. **File Cleanup**: Remove work directories and state files

#### `uninstall_cluster()`
**Purpose**: Remove Kubernetes cluster while preserving tools.

**Process**:
1. **Cluster Deletion**: `kind delete cluster --name keeper-cluster`
2. **Context Cleanup**: Remove kubectl contexts and cluster references
3. **Error Suppression**: Ignore missing context errors

#### `uninstall_tools()`
**Purpose**: Remove all installed command-line tools.

**Tools Removed**:
- **kubectl**: `/usr/local/bin/kubectl`
- **kind**: `/usr/local/bin/kind`
- **helm**: `/usr/local/bin/helm`
- **go**: `/usr/local/go` directory
- **ksm**: Python package via pip3

**Features**:
- **Existence Check**: Only remove if tools exist
- **Permission Handling**: Use sudo for non-root removals
- **PATH Cleanup**: Remove Go PATH modifications from `.bashrc`
- **Error Handling**: Continue on individual tool removal failures

#### `clean_external_secrets_only()`
**Purpose**: Specialized cleanup for corrupted External Secrets installations.

**Process**:
1. **Helm Removal**: Uninstall external-secrets release
2. **Namespace Deletion**: Remove external-secrets-system namespace
3. **Namespace Wait**: Wait for complete namespace deletion
4. **CRD Cleanup**: Manually remove all External Secrets CRDs
5. **User Guidance**: Provide next steps for re-installation

---

## 8. Main Installer Class

### 8.1 KubernetesKeeperInstaller

#### Class Architecture
```python
class KubernetesKeeperInstaller:
    def __init__(self, config: BuildConfig)
    def _load_state(self) -> BuildState
    def _save_state(self) -> None
    def run(self) -> bool
    def _show_usage_info(self) -> None
```

#### State Management

##### `_load_state() -> BuildState`
**Purpose**: Loads persistent state from JSON file.
**Location**: `{work_dir}/state.json`
**Fallback**: Creates new BuildState if file doesn't exist or is corrupted.

##### `_save_state() -> None`
**Purpose**: Persists current state to JSON file.
**Timing**: Called after each successful stage completion.
**Error Handling**: Logs errors but doesn't fail installation.

#### Stage Orchestration

##### Stage Definition
```python
self.stages = [
    ("01", PrerequisitesStage("01", "Prerequisites Check", config, self.state)),
    ("02", ToolsInstallationStage("02", "Tools Installation", config, self.state)),
    ("03", ClusterSetupStage("03", "Cluster Setup", config, self.state)),
    ("04", DashboardStage("04", "Kubernetes Dashboard", config, self.state)),
    ("05", ExternalSecretsStage("05", "External Secrets Operator", config, self.state)),
    ("06", KeeperIntegrationStage("06", "Keeper Integration", config, self.state)),
    ("07", ValidationStage("07", "Validation", config, self.state)),
]
```

##### `run() -> bool`
**Purpose**: Main installation orchestration.

**Execution Flow**:
1. **Initialization**: Log installation start and work directory
2. **Resume Logic**: Determine starting stage based on `--resume` flag
3. **Stage Execution**: Execute stages sequentially
4. **State Persistence**: Save state after each successful stage
5. **Failure Handling**: Stop on first failure and provide resume guidance
6. **Success Handling**: Display usage information on completion

##### Resume Capability
- **Stage Identification**: Stages identified by numeric IDs ("01" - "07")
- **Starting Point**: Installation can resume from any stage
- **State Preservation**: Previous stage results and user choices preserved
- **Command Generation**: Automatic resume command generation on failure

#### Post-Installation Guidance

##### `_show_usage_info()`
**Purpose**: Comprehensive post-installation information display.

**Information Provided**:
1. **Dashboard Access**: URL, token, and usage instructions
2. **Keeper Configuration**: Steps to configure real secrets
3. **Cluster Access**: HTTP/HTTPS endpoints with selected ports
4. **Useful Commands**: Common kubectl commands for management
5. **File Locations**: Log and state file locations

**Dynamic Content**:
- **Port Information**: Shows actual selected ports
- **Token Display**: Shows generated dashboard token
- **Conditional Sections**: Based on installation mode and components

---

## 9. Command Line Interface

### 9.1 Argument Parser Configuration

#### Build Options
- `--allow-root`: Permit execution as root user
- `--production-only`: Install only production components
- `--dev-only`: Install only development components
- `--skip-dashboard`: Skip Kubernetes Dashboard installation

#### Keeper Options
- `--ksm-token`: Keeper Secrets Manager one-time token
- `--ksm-config`: Path to existing KSM configuration file

#### Port Configuration
- `--auto-ports`: Automatically select available ports
- `--interactive-ports`: Interactive port selection interface
- `--http-port`: Custom HTTP port
- `--https-port`: Custom HTTPS port
- `--dashboard-port`: Custom dashboard port
- `--nodeport`: Custom NodePort base

#### Build Control
- `--resume`: Resume from specific stage (01-07)
- `--skip-stages`: Skip specified stages (comma-separated)
- `--work-dir`: Custom work directory
- `--debug`: Enable debug mode
- `--debug-output`: Debug output file
- `--reconfigure`: Interactive reconfigure mode
- `--allow-destructive`: Allow destructive operations

#### Uninstall Options
- `--uninstall`: Interactive uninstall menu
- `--uninstall-all`: Remove everything
- `--uninstall-cluster`: Remove only cluster
- `--uninstall-tools`: Remove only tools
- `--uninstall-dashboard`: Remove only dashboard
- `--clean-external-secrets`: Clean corrupted External Secrets

### 9.2 Argument Processing Logic

#### Uninstall Priority
- Uninstall operations processed before installation
- Require root privileges or `--allow-root` flag
- Specialized cleanup operations available

#### Port Selection Logic
```python
auto_port_selection = None
if args.auto_ports:
    auto_port_selection = True
elif args.interactive_ports:
    auto_port_selection = False
# None triggers interactive prompt
```

#### Custom Port Processing
```python
custom_ports = {}
if args.http_port:
    custom_ports['http'] = args.http_port
# ... additional port processing
```

---

## 10. Error Handling & Recovery

### 10.1 Error Handling Philosophy

#### Graceful Degradation
- **Non-critical failures**: Generate warnings but continue installation
- **Critical failures**: Stop installation with clear error messages
- **Recovery guidance**: Provide specific commands for issue resolution

#### Error Suppression System
- **Expected failures**: Suppress errors for existence checks
- **Diagnostic operations**: Suppress errors during troubleshooting
- **Cleanup operations**: Suppress errors during uninstall

### 10.2 Recovery Mechanisms

#### State-based Recovery
- **Resume capability**: Installation can be resumed from any stage
- **State preservation**: User choices and configuration preserved
- **Incremental progress**: Only failed stages need re-execution

#### Automatic Recovery
- **Port conflicts**: Automatic alternative port selection
- **Tool installation**: Multiple installation method attempts
- **CRD issues**: Advanced validation and recovery mechanisms

#### Manual Recovery
- **Cleanup commands**: Specific cleanup commands for various scenarios
- **Diagnostic information**: Detailed troubleshooting output
- **Recovery guidance**: Step-by-step recovery instructions

---

## 11. Security Considerations

### 11.1 Privilege Management

#### Root Execution Protection
- **Default denial**: Refuses to run as root by default
- **Explicit override**: Requires `--allow-root` flag
- **Privilege escalation**: Uses sudo only when necessary

#### Destructive Operation Protection
- **Explicit confirmation**: Requires specific confirmation phrases
- **Operation limiting**: Restricts destructive operations to specific modes
- **Reversibility**: Provides uninstall capabilities for all operations

### 11.2 Credential Management

#### Keeper Token Handling
- **Memory only**: Tokens processed in memory when possible
- **Temporary files**: Automatic cleanup of temporary credential files
- **Base64 encoding**: Proper encoding for Kubernetes secret storage

#### Dashboard Security
- **HTTPS only**: Dashboard exposed only via HTTPS
- **Token authentication**: Long-lived but revocable tokens
- **Admin privileges**: Full cluster access for development convenience

---

## 12. Performance & Scalability

### 12.1 Performance Optimizations

#### Port Scanning Optimization
- **Range limiting**: Limits scan ranges to prevent excessive delays
- **Early termination**: Stops when sufficient alternatives found
- **Caching**: Reuses port information within single execution

#### Command Execution Optimization
- **Selective capture**: Only captures output when needed
- **Error suppression**: Avoids expensive error processing when not needed
- **Parallel-safe**: Safe for concurrent execution

### 12.2 Scalability Considerations

#### Resource Requirements
- **Minimal footprint**: Designed for development environments
- **Single-node clusters**: Optimized for local development
- **Development focus**: Not intended for production deployment

#### Extensibility
- **Stage-based architecture**: Easy to add new installation stages
- **Configuration-driven**: Behavior controlled by configuration objects
- **Modular design**: Individual components can be modified independently

---

## 13. Troubleshooting & Diagnostics

### 13.1 Diagnostic Systems

#### Comprehensive Logging
- **Dual logging**: Console and file logging with different detail levels
- **Color coding**: Visual indicators for different log levels
- **Debug information**: Detailed command execution logging

#### Advanced Diagnostics
- **Component status**: Individual component health checking
- **Resource validation**: Kubernetes resource existence and status
- **Network connectivity**: Port availability and service accessibility

### 13.2 Common Issues & Solutions

#### CRD Establishment Issues
- **Symptoms**: CRDs exist but show "Not Established"
- **Diagnosis**: Functional validation vs status checking
- **Recovery**: Controller restart and functional validation

#### Port Conflicts
- **Symptoms**: Services fail to start due to port conflicts
- **Diagnosis**: Port scanning and conflict detection
- **Recovery**: Automatic alternative port selection

#### Permission Issues
- **Symptoms**: Docker or kubectl permission denied
- **Diagnosis**: User group membership and sudo capabilities
- **Recovery**: User group addition or sudo usage

---

## 14. Development & Maintenance

### 14.1 Code Organization

#### Modular Architecture
- **Section separation**: Clear separation of functionality
- **Class hierarchy**: Inheritance-based stage architecture
- **Function grouping**: Related functions grouped in sections

#### Documentation Standards
- **Docstring coverage**: All public functions documented
- **Type annotations**: Full type annotation for clarity
- **Comment quality**: Explanatory comments for complex logic

### 14.2 Testing Considerations

#### Manual Testing
- **Multiple platforms**: Tested on various Linux distributions
- **Different scenarios**: Installation, resumption, and uninstall testing
- **Edge cases**: Port conflicts, permission issues, network problems

#### Automated Testing Potential
- **Unit testing**: Individual function testing potential
- **Integration testing**: End-to-end installation testing
- **Mock environments**: Docker-based testing environments

---

## 15. Future Enhancements

### 15.1 Potential Improvements

#### Enhanced Security
- **Credential encryption**: Encrypted credential storage
- **Role-based access**: More granular permission management
- **Audit logging**: Comprehensive operation auditing

#### Additional Integrations
- **CI/CD integration**: Pipeline-friendly execution modes
- **Cloud provider support**: Support for cloud Kubernetes services
- **Additional secret managers**: Support for other secret management systems

#### User Experience
- **Web interface**: Browser-based installation interface
- **Progress indicators**: Visual progress reporting
- **Configuration validation**: Pre-installation validation

### 15.2 Maintenance Considerations

#### Version Management
- **Tool versions**: Regular updates to tool versions
- **Compatibility testing**: Testing with new Kubernetes versions
- **Dependency management**: Monitoring of external dependencies

#### Documentation Maintenance
- **Usage examples**: Regular example updates
- **Troubleshooting guides**: Expanded troubleshooting documentation
- **Architecture documentation**: Continued architectural documentation

---

## Conclusion

The Kubernetes + Keeper Installer represents a comprehensive solution for automated Kubernetes development environment setup with integrated secrets management. Its modular architecture, robust error handling, intelligent port management, and comprehensive diagnostics make it suitable for both novice and experienced users. The script's design prioritizes user experience while maintaining security and reliability, providing a solid foundation for Kubernetes development with Keeper Secrets Manager integration.
