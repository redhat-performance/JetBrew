# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

JetBrew is an Ansible-based automation tool for deploying Red Hat OpenStack Services on OpenShift (RHOSO) on baremetal infrastructure in Scale Lab environments. It uses Validated Architecture (VA) and Deployment Templates (DT) patterns via Kustomize.

## Key Commands

```bash
# Deploy RHOSO (full end-to-end)
ansible-playbook ansible/main.yml

# Delete all RHOSO resources from the cluster
ansible-playbook ansible/delete-rhoso.yml

# Deploy external Ceph cluster
ansible-playbook ansible/deploy_external_ceph.yaml

# Reprovision nodes
ansible-playbook utils/reprovision.yml
```

All playbooks require `ansible/group_vars/all.yml` to be configured (copy from `all.sample.yml`).

## Architecture

### Playbook Execution Flow (`ansible/main.yml`)

The main playbook runs on `localhost` with `ocp_environment` (KUBECONFIG) set, executing roles in order:

1. **bootstrap** — Clones the `architecture` repo to `dt_path`, installs Kustomize, downloads OCP inventory from the lab, enables IP forwarding, optionally sets up observability operator
2. **values-prep** — Discovers OCP node names/IPs/MACs, sets up SSH, finds network interfaces by MAC address, identifies common disks across nodes, renders Jinja2 templates (NNCP, service values, LVMS, kustomization.yaml) into the architecture repo
3. **values-prep-dp** — Prepares EDPM (External Data Plane Management) nodeset values for compute nodes, renders dataplane templates
4. **lvms** — Applies LVMS (Local Volume Manager Storage) CRs via Kustomize
5. **common-osp** — Deploys core OpenStack operators
6. **controlplane** — Applies NNCP, networking, and control plane CRs via Kustomize; waits for control plane readiness
7. **dataplane** — Applies EDPM nodeset and deployment CRs; waits for deployment; runs nova host discovery

Optional Ceph integration: If `ceph_backend: true`, a pre-play connects to the Ceph admin node and runs the `ceph-osp-prep` role (creates pools, CephX keys, exports config) before the main deployment.

### Template Rendering Pattern

The `values-prep` and `values-prep-dp` roles render Jinja2 templates into the cloned architecture repo at `dt_path` (default `/root/test/architecture`). The `controlplane` and `dataplane` roles then use `kustomize build` on those paths to generate and apply Kubernetes CRs.

Templates are in:
- `ansible/roles/values-prep/templates/` — NNCP, service values, LVMS, kustomization, Ceph service values
- `ansible/roles/values-prep-dp/templates/` — EDPM nodeset values, kustomization

### Key Variables (`ansible/group_vars/all.yml`)

- `cloud` / `lab` — Scale Lab cloud identifier and lab type
- `compute_count` — Number of compute nodes
- `ssh_password` / `ssh_username` / `ssh_key_file` — Baremetal node access
- `ctlplane_start_ip` — Control plane IP allocation start
- `ocp_environment.KUBECONFIG` — Path to kubeconfig
- `ceph_backend` — Enable Ceph storage integration (requires prior `deploy_external_ceph.yaml` run)
- `dt_path` — Where the architecture repo is cloned

### Deletion (`ansible/roles/cleanup-openstack`)

The delete playbook uses NAD-based discovery: it finds all NetworkAttachmentDefinitions in the `openstack` namespace, then deletes only MetalLB IPAddressPools and L2Advertisements with matching names — preserving non-OpenStack MetalLB resources.
