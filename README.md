# ansible-playbooks

A collection of Ansible playbooks for managing KubeVirt virtual machines and infrastructure automation using Ansible Automation Platform (AAP).

## Overview

This repository contains playbooks for:
- **KubeVirt VM Operations**: Deploy, snapshot, restore, and manage VMs on OpenShift
- **Image Management**: Build and push custom RHEL images
- **Web Application Deployment**: Setup and rolling upgrades for web apps
- **Windows Server Management**: Deploy and configure Windows VMs
- **GitOps Integration**: ArgoCD-managed AAP job template deployment

## Repository Structure

```
ansible-playbooks/
├── manifests/              # Kubernetes/ArgoCD manifests
│   ├── argocd-application.yaml           # ArgoCD Application definition
│   ├── argocd-rbac.yaml                  # RBAC for ArgoCD/AWX Operator
│   ├── kustomization.yaml                # Kustomize configuration
│   ├── project-ansible-playbooks.yaml    # AnsibleProject CR (reference only)
│   ├── create-project-job.yaml           # Job to create project via API
│   └── jobtemplate-*.yaml                # JobTemplate CRs (14 templates)
├── collections/            # Ansible collections requirements
├── *.yaml                  # Playbook files (17 playbooks)
└── add-surveys-test-2.yaml # Survey configuration playbook
```

## Playbooks

### VM Operations - RHEL
- **deploy_rhel_vm_auto_preference.yaml** - Deploy RHEL VMs with automatic preferences
- **manage-vms.yaml** - Power management (start, stop, restart VMs)
- **snapshot-vm-task.yaml** - Create VM snapshots
- **restore-vm-snapshot.yaml** - Restore VMs from snapshots
- **patch-kubevirt-vm-workflow.yaml** - Patch RHEL VMs with rolling upgrades
- **vm-hot-add-disk-survey.yaml** - Hot-add disks to running VMs
- **upgrade-vm-task.yaml** - Upgrade VM configurations

### VM Operations - Windows
- **deploy_win_server.yaml** - Deploy Windows Server VMs
- **windows-install-iis.yaml** - Install and configure IIS
- **install-windows-packages-aap.yaml** - Install packages via Chocolatey

### Web Application Management
- **webapp-setup.yaml** - Deploy Node.js web applications from Git
- **webapp-rolling-update-k8s-demo.yaml** - Perform rolling upgrades on webapp VMs
- **label-ecommerce-vms.yaml** - Label VMs for e-commerce workloads
- **service-ecommerce-portal.yaml** - Configure e-commerce portal service

### Image Management
- **image_builder.yaml** - Build custom RHEL images
- **push_image_2_quay.yaml** - Push images to Quay.io registry

### Configuration Management
- **add-surveys-test-2.yaml** - Configure job template surveys in AAP

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
2. **Kubernetes Secret** `aap-controller-access` in `aap` namespace containing AAP credentials
3. **Source Control Credential** named `srccontrol-github` in AAP Controller
4. **Inventory** named `your-inventory-name` in AAP Controller
5. **Credential** named `your-inventory-name` in AAP Controller (for KubeVirt access)
6. **Credential** named `quay-registry-credential` in AAP Controller (for image operations)

#### Step 1: Create the AAP Project

The `ansible-playbooks` project must be created manually in AAP Controller due to AWX Resource Operator limitations.

**Option A: Via AAP Web UI**
1. Navigate to Resources → Projects → Add
2. Configure:
   - Name: `ansible-playbooks`
   - Organization: `Default`
   - Source Control Type: `Git`
   - Source Control URL: `https://github.com/your-username/your-repo.git`
   - Source Control Branch: `main`
   - Source Control Credential: `srccontrol-github`

**Option B: Via API (using create-project-job.yaml)**
```bash
oc apply -f manifests/argocd-rbac.yaml
oc apply -f manifests/create-project-job.yaml
```

This creates a Kubernetes Job that uses the AAP Controller API to create/update the project.

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

For detailed bug report, see [BUG_REPORT.md](BUG_REPORT.md).

## Requirements

### Ansible Collections
See [collections/requirements.yaml](collections/requirements.yaml) for required collections:
- `kubernetes.core`
- `community.general`
- `ansible.posix`
- `infra.aap_configuration`
- Custom collections from GitHub

Install with:
```bash
ansible-galaxy collection install -r collections/requirements.yaml
```

### AAP Controller Configuration

Ensure these resources exist in your AAP Controller before deployment:

- **Credentials**:
  - `your-inventory-name` - KubeVirt/OpenShift access
  - `srccontrol-github` - GitHub source control access
  - `quay-registry-credential` - Quay.io registry access

- **Inventory**:
  - `your-inventory-name` - KubeVirt-based inventory

- **Execution Environment**:
  - `Default execution environment` - Must have required collections

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

## Usage

### Running Playbooks Directly

```bash
# Deploy a RHEL VM
ansible-playbook deploy_rhel_vm_auto_preference.yaml

# Create VM snapshot
ansible-playbook snapshot-vm-task.yaml

# Setup web application
ansible-playbook webapp-setup.yaml -e vm_name=web-01
```

### Running via AAP Controller

After ArgoCD deployment, job templates are available in AAP Controller and can be:
- Launched manually via the UI
- Called via the AAP API
- Triggered by workflows
- Scheduled for recurring execution

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

## Troubleshooting

### ArgoCD Application Issues

Check ArgoCD Application status:
```bash
oc get application ansible-automation-platform-config -n openshift-gitops
```

View sync status and errors:
```bash
oc describe application ansible-automation-platform-config -n openshift-gitops
```

### AWX Operator Issues

Check JobTemplate CR status:
```bash
oc get jobtemplates -n aap
oc describe jobtemplate ops-rhel-deploy-vm -n aap
```

View operator logs:
```bash
oc logs -n awx-operator deployment/awx-operator-controller-manager -f
```

### Force ArgoCD Sync

```bash
argocd app sync ansible-automation-platform-config
```

Or via kubectl:
```bash
kubectl patch application ansible-automation-platform-config -n openshift-gitops \
  --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"main"}}}'
```

### Delete and Redeploy

```bash
# Delete the application
oc delete application ansible-automation-platform-config -n openshift-gitops

# If stuck, remove finalizers
oc patch applications.argoproj.io ansible-automation-platform-config -n openshift-gitops \
  --type json -p='[{"op": "remove", "path": "/metadata/finalizers"}]'

# Redeploy
oc apply -f manifests/argocd-application.yaml
```

## Contributing

When adding new playbooks:
1. Add the playbook YAML file to the repository root
2. Create a corresponding `manifests/jobtemplate-*.yaml` file
3. Add the JobTemplate to `manifests/kustomization.yaml` resources list
4. If the job template requires a survey, add it to `add-surveys-test-2.yaml`
5. Commit and push - ArgoCD will automatically deploy

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [AWX Resource Operator](https://github.com/ansible/awx-resource-operator)
- [Ansible Automation Platform Documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/)
- [KubeVirt Documentation](https://kubevirt.io/user-guide/)
- [Kustomize Documentation](https://kustomize.io/)
