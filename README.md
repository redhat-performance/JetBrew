# JetBrew (RHOSO Baremetal Installer)

An Ansible-based automation tool for deploying Red Hat OpenStack Services on OpenShift (RHOSO) on baremetal infrastructure in Scale Lab environments, leveraging the Validated Architecture (VA) and Deployment Templates (DT) architecture patterns.

## Overview

This installer streamlines the deployment of RHOSO by automating the entire process from initial setup to a fully functional OpenStack cloud. It requires minimal user input and handles all the complex orchestration of OpenStack components on OpenShift.

## Features

- **Automated Deployment**: End-to-end automation of RHOSO installation
- **Minimal Configuration**: Requires only essential parameters like lab type, cloud number, and credentials
- **Pre-provisioned Support**: Currently supports pre-provisioned External Data Plane Management (EDPM) deployments
- **Scale Lab Integration**: Optimized for Red Hat Scale Lab environments
- **Validated Architecture**: Uses proven architecture patterns and best practices

## Prerequisites

Before running this installer, ensure your environment meets the following requirements:

### Infrastructure Requirements
- **OpenShift Cluster**: Must be deployed using [Jetlag](https://github.com/redhat-performance/jetlag)
- **Compute Nodes**: Should be deployed with RHEL 9.4
- **Network Configuration**: Proper network setup for OpenStack services
- **Storage**: Adequate storage configuration for OpenStack workloads

### Software Requirements
- **Ansible**: Version 2.9 or higher
- **OpenShift CLI (oc)**: Latest version
- **Kustomize**: Automatically downloaded during bootstrap
- **SSH Access**: To baremetal nodes with proper key configuration

## Configuration

### Required Variables

Edit the `ansible/group_vars/all.yml` file and configure the following mandatory variables:

```yaml
# Cloud and Lab Configuration
cloud: cloud40                    # Cloud identifier (e.g., cloud40)
lab: scalelab                     # Lab type: scalelab or performancelab
compute_count: 1                  # Number of compute nodes

# SSH Configuration
ssh_password: your-password       # SSH password for baremetal nodes
ssh_username: cloud-admin         # SSH username for baremetal nodes
ssh_key_file: ~/.ssh/id_rsa      # SSH private key file path

# Network Configuration
ctlplane_start_ip: 172.16.20.100  # Control plane start IP address

# OpenShift Configuration
ocp_environment:
  KUBECONFIG: /root/mno/kubeconfig  # Path to kubeconfig file
```

### Optional Variables

Additional variables can be configured based on your specific environment needs:

```yaml
# Architecture Repository
dt_path: /root/test/architecture   # Local path for architecture repository

# Nova Migration
nova_migration_key: ~/.ssh/nova_migration  # Nova migration SSH key
```

## Installation

### Quick Start

1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd RHOSO
   ```

2. **Configure variables**:
   ```bash
   vim ansible/group_vars/all.yml
   ```

3. **Run the installer**:
   ```bash
   ansible-playbook ansible/main.yml
   ```

### Deployment Phases

The installer executes the following phases in order:

1. **Bootstrap Phase**: Environment setup and tool installation
2. **Values Preparation**: Generate configuration values for OpenStack components
3. **EDPM Values Preparation**: Prepare External Data Plane Management values
4. **LVMS Setup**: Configure Local Volume Manager Storage
5. **Common OpenStack Services**: Deploy core OpenStack operators
6. **Control Plane Deployment**: Deploy OpenStack control plane services
7. **Data Plane Deployment**: Deploy and configure compute nodes

## How It Works

### Architecture Overview

The installer follows a systematic approach:

1. **Repository Cloning**: Clones the validated architecture repository containing deployment templates
2. **Environment Analysis**: Analyzes your lab environment and generates appropriate configuration
3. **Custom Resource Generation**: Creates Kubernetes Custom Resources (CRs) tailored to your environment
4. **Orchestrated Deployment**: Applies resources in the correct order with proper dependencies
5. **Validation**: Verifies successful deployment and service availability

### Key Components

- **Kustomize Templates**: Uses Kustomize for template management and customization
- **OpenStack Operators**: Leverages OpenStack operators for service lifecycle management
- **EDPM Integration**: Manages External Data Plane nodes for compute services
- **Network Configuration**: Automated network setup using NNCP (Node Network Configuration Policy)
- **Storage Management**: Automated storage configuration using LVMS

## Post-Deployment

After successful deployment, the installer automatically:

1. **Discovers Compute Hosts**: Runs `nova-manage cell_v2 discover_hosts` to register compute nodes
2. **Validates Hypervisors**: Lists available hypervisors to confirm successful deployment
3. **Provides Status**: Displays deployment status and service availability

## RHOSO Deletion

To remove **any** OpenStack resources from your cluster (regardless of deployment tool), use the RHOSO deletion playbook:

```bash
ansible-playbook ansible/delete-rhoso.yml
```

This will **intelligently** discover and delete:
- All OpenStack resources (any names, any deployment tool)
- **MetalLB resources matching OpenStack NADs** (100% accurate identification)
- Network resources and OpenStack namespaces (`openstack` + `openstack-operators`)

### Complete RHOSO Deletion Process
The deletion process uses a **comprehensive approach** to safely remove all RHOSO components:

#### 1. **Smart MetalLB Deletion (NAD-based)**
- **Discovers** all NetworkAttachmentDefinitions (NADs) in `openstack` namespace
- **Matches** MetalLB resources in `metallb-system` namespace with same names as NADs
- **Deletes** only the matching IPAddressPools and L2Advertisements
- **Preserves** all other MetalLB resources used by other applications

**Example:**
- NAD found: `ctlplane` → Deletes: `ipaddresspool/ctlplane`, `l2advertisement/ctlplane`
- NAD found: `internalapi` → Deletes: `ipaddresspool/internalapi`, `l2advertisement/internalapi`  
- Other resource: `my-app-pool` → **Preserved** (no matching NAD)

#### 2. **Controlled Namespace Deletion**
1. **OpenStack namespace** (`openstack`):
   - Waits for all pods to terminate
   - Deletes namespace with timeout
2. **OpenStack operators namespace** (`openstack-operators`):
   - Deletes operator resources first
   - Waits for operator pods to terminate  
   - Deletes namespace with timeout

This approach is **100% accurate** and provides **complete RHOSO removal**!

## Troubleshooting

### Common Issues

1. **KUBECONFIG Not Found**: Ensure the kubeconfig path is correct and accessible
2. **SSH Authentication**: Verify SSH credentials and key file permissions
3. **Network Connectivity**: Check network configuration and firewall settings
4. **Resource Limits**: Ensure sufficient resources (CPU, memory, storage) are available

### Verification Commands

```bash
# Check OpenStack services
oc get pods -n openstack

# Verify hypervisors
oc rsh -n openstack openstackclient openstack hypervisor list

# Check dataplane deployment
oc get openstackdataplanedeployment -n openstack
```

## Support

For issues and questions:
- Check the troubleshooting section above
- Review OpenShift and OpenStack operator logs
- Consult the [Red Hat OpenStack Services on OpenShift documentation](https://docs.openstack.org/openstack-operator/)

## Contributing

Contributions are welcome! Please follow these guidelines:
1. Fork the repository
2. Create a feature branch
3. Test your changes thoroughly
4. Submit a pull request with detailed description

## License

This project is licensed under the Apache License 2.0. See the LICENSE file for details.
