# ansible-playbooks

A collection of Ansible playbooks for managing KubeVirt virtual machines and infrastructure automation using Ansible Automation Platform (AAP).

## Overview

This repository contains playbooks for:
- **KubeVirt VM Operations**: Deploy, snapshot, restore, and manage VMs on OpenShift
- **Image Management**: Build and push custom RHEL images
- **Web Application Deployment**: Setup and rolling upgrades for web apps
- **Windows Server Management**: Deploy and configure Windows VMs
- **GitOps Integration**: ArgoCD-managed AAP job template deployment

## GitOps with ArgoCD

This repository is designed to be deployed using ArgoCD for GitOps-based AAP configuration management.

### How It Works

1. **ArgoCD Application** watches the `manifests/` directory in this Git repository
2. **Kustomize** patches all JobTemplate resources with sync-wave annotations
3. **AWX Resource Operator** creates job templates in AAP Controller
4. **Automated Sync** keeps AAP job templates in sync with Git

### ArgoCD Application Configuration

The ArgoCD Application ([manifests/argocd-application.yaml](manifests/argocd-application.yaml)) is configured with:

- **Source**: `https://github.com/your-username/your-repo.git` (main branch, manifests/ path)
- **Destination**: `aap` namespace on local cluster
- **Sync Policy**:
  - Automated with prune and self-heal enabled
  - Auto-creates namespace if needed
  - Retries with exponential backoff (5s → 3m max)
- **Ignore Differences**: Status fields on JobTemplate and AnsibleProject resources

### Deployment Workflow

#### Prerequisites

1. **AAP Controller** with AWX Resource Operator installed
2. **Kubernetes Secret** `your-credential` in `aap` namespace containing AAP credentials
3. **Source Control Credential** named `your-git-cred` in AAP Controller
4. **Inventory** named `your-inventory-name` in AAP Controller
5. **Credential** named `your-inventory-name` in AAP Controller (for KubeVirt access)
6. **Credential** named `quay-registry-credential` in AAP Controller (for image operations)

#### Step 1: Create the AAP Project

The `ansible-playbooks` project must be created manually (at the moment there is a bug in resource operator)

#### Step 2: Deploy ArgoCD Application

Apply the RBAC and ArgoCD Application:

```bash
# Apply RBAC for ArgoCD to manage AWX resources
oc apply -f manifests/argocd-rbac.yaml

# Deploy the ArgoCD Application
oc apply -f manifests/argocd-application.yaml
```

ArgoCD will automatically:
1. Sync the manifests from Git
2. Create 14 JobTemplate resources in the `aap` namespace
3. AWX Resource Operator reconciles these into AAP Controller job templates

#### Step 3: Add Surveys (Manual)

Surveys cannot be managed via the AWX Resource Operator CRs. Run the survey configuration playbook manually:

```bash
ansible-playbook add-surveys-test-2.yaml
```

This configures surveys for job templates that require user input.

### Job Templates Managed by ArgoCD

The following 14 job templates are deployed via ArgoCD (sync-wave 1):

**Image Management:**
- `image-create-rhel-image` - Build custom RHEL images
- `image-push-image-2-quay` - Push images to Quay.io

**VM Operations - RHEL:**
- `ops-restore-from-vm-snapshot` - Restore VMs from snapshots
- `ops-rhel-deploy-vm` - Deploy RHEL VMs
- `ops-rhel-patch-vms` - Patch RHEL VMs
- `ops-vm-hot-add-disk` - Hot-add disk to running VM (has survey)
- `ops-vm-power-management` - VM power operations
- `ops-vm-snapshot` - Create VM snapshots

**VM Operations - WebApps:**
- `ops-webapp-rolling-upgrade-demo` - Rolling upgrade for web apps (has survey)
- `ops-webapp-setup` - Initial webapp deployment

**VM Operations - Windows:**
- `ops-win-deploy-vm` - Deploy Windows Server VMs
- `ops-win-enable-iis` - Enable IIS on Windows
- `ops-win-install-packages` - Install Windows packages

**Post-Configuration:**
- `internal-it-aap-post-configuration` - AAP post-configuration tasks

### Kustomize Patches

All JobTemplate resources are automatically patched with:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
```

This ensures templates are created after the project exists (which should be in wave 0 or created manually).

### Known Limitations

**AWX Resource Operator Issues:**
- The operator's `platform-resource-runner-rhel9` image (AAP 2.6) is missing the `job_runner` role
- This prevents using `AnsibleProject` CRs reliably
- **Workaround**: Create the `ansible-playbooks` project manually in AAP Controller

**PostSync Hooks Disabled:**
- ArgoCD PostSync hooks using `AnsibleJob` CRs fail with the same operator issue
- **Workaround**: Run survey configuration manually via `add-surveys-test-2.yaml`


### AAP Controller Configuration

Ensure these resources exist in your AAP Controller before deployment:

- **Credentials**:
  - `your-inventory-name` - KubeVirt/OpenShift access
  - `your-git-cred` - GitHub source control access
  - `quay-registry-credential` - Quay.io registry access

- **Inventory**:
  - `your-inventory-name` - KubeVirt-based inventory

- **Execution Environment**:
  - `Default execution environment` - Must have required collections
  - For windows VM tasks a specific execution environment needs to be built. 

- **Organization**:
  - `Default` - Organization for all resources

### Kubernetes Secrets

Create the AAP Controller access secret:
```bash
oc create secret generic aap-controller-access \
  --from-literal=host=https://your-aap-controller.example.com \
  --from-literal=token=your-oauth-token \
  -n aap
```


### Updating Job Templates

1. Make changes to `manifests/jobtemplate-*.yaml` files
2. Commit and push to Git
3. ArgoCD automatically syncs changes to AAP Controller (if automated sync is enabled)
4. AWX Resource Operator updates the job templates in AAP

## Architecture

```
┌─────────────────┐
│   Git Repo      │
│ (this repo)     │
└────────┬────────┘
         │
         │ watches
         ▼
┌─────────────────┐
│     ArgoCD      │
│  Application    │
└────────┬────────┘
         │
         │ creates
         ▼
┌─────────────────┐      ┌──────────────────┐
│  JobTemplate    │─────▶│  AWX Resource    │
│   CRs (K8s)     │      │    Operator      │
└─────────────────┘      └────────┬─────────┘
                                  │
                                  │ reconciles
                                  ▼
                         ┌──────────────────┐
                         │  AAP Controller  │
                         │  Job Templates   │
                         └──────────────────┘
```

