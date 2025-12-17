# GitHub Copilot Instructions for hetzner-ocp4

This document provides context and guidelines for GitHub Copilot to assist effectively with this Ansible infrastructure-as-code project.

## Project Overview

**Purpose:** Deploy and manage Red Hat OpenShift Container Platform 4 (OCP4) clusters on Hetzner bare-metal servers as self-managed, automated sandbox environments.

**Key Capabilities:**
- Automated infrastructure provisioning on Hetzner dedicated servers
- Full cluster lifecycle management (provision → create → start/stop → destroy)
- Multi-provider DNS support (Route53, Cloudflare, GCP, Azure, Hetzner, Gandi, DigitalOcean)
- Post-installation add-ons system for extensibility
- Let's Encrypt certificate automation
- Support for multiple cluster topologies (single-node, compact, normal)

## Technology Stack

### Core Technologies
- **Ansible** - Automation orchestration (ansible-core 2.15.13+)
- **KVM/Libvirt** - Hypervisor for virtual machine management
- **OpenShift 4.x** - Container orchestration platform
- **RHCOS (Red Hat CoreOS)** - Immutable OS for cluster nodes
- **Hetzner Robot API** - Server provisioning and management

### Key Ansible Collections
- `community.libvirt` - VM management
- `community.crypto` - Certificate operations
- `community.general` - General utilities
- `kubernetes.core` - Kubernetes/OpenShift operations
- `community.aws`, `google.cloud`, `azure.azcollection` - Cloud providers

### Supporting Tools
- Ansible Navigator (execution environment runner)
- Butane/Ignition for server configuration
- HAProxy for load balancing
- Podman for container runtime

## Project Structure

```
hetzner-ocp4/
├── ansible/                          # Main automation code
│   ├── setup.yml                     # Master orchestration playbook
│   ├── 00-provision-hetzner.yml      # OS provisioning
│   ├── 01-prepare-host.yml           # Hypervisor setup
│   ├── 02-create-cluster.yml         # Cluster installation
│   ├── 03-stop-cluster.yml           # Graceful shutdown
│   ├── 04-start-cluster.yml          # Start cluster VMs
│   ├── 99-destroy-cluster.yml        # Cleanup all resources
│   ├── renewal-certificate.yml       # Let's Encrypt renewal
│   ├── run-add-ons.yml               # Post-install extensions
│   ├── group_vars/all/               # Global variables & vaults
│   ├── roles/                        # Core automation roles
│   │   ├── provision-hetzner/        # Server provisioning
│   │   ├── openshift-4-cluster/      # Cluster lifecycle
│   │   ├── openshift-4-loadbalancer/ # HAProxy setup
│   │   ├── letsencrypt/              # Certificate management
│   │   └── public_dns/               # DNS provider integration
│   └── add-on-roles/                 # Optional post-install roles
├── docs/                             # Comprehensive documentation
├── cluster-example.yml               # Example configuration template
├── execution-environment.yml         # Ansible-ee build definition
├── ee-requirements.yml               # Galaxy collection versions
├── ee-python-requirements.txt        # Python dependencies
├── ansible.cfg                       # Ansible configuration
└── inventory/hosts.yaml              # Inventory definition
```

## Naming Conventions

### Playbooks
- **Numbered prefix** for sequential workflows: `00-`, `01-`, `02-`, etc.
- **Descriptive names** in kebab-case: `provision-hetzner`, `prepare-host`, `create-cluster`
- **Special playbooks**: `setup.yml` (orchestrator), `renewal-certificate.yml`, `run-add-ons.yml`

### Roles
- Descriptive kebab-case names: `provision-hetzner`, `openshift-4-cluster`
- Provider/component-specific: `letsencrypt`, `public_dns`
- Add-on roles follow same pattern: `ntp`, `web-terminal`, `garbagecollection`

### Variables
- **Prefixed by context**: `le_*` (letsencrypt), `vm_*` (virtual machine), `cluster_*` (cluster config)
- **Boolean flags**: `storage_nfs`, `letsencrypt_disabled`, `install_web_terminal`
- **Lists/dictionaries**: Plural names like `compute_nodes`, `dns_providers`

### Task Names
- Clear, action-oriented phrases: "Create disk for {{ vm_instance_name }}"
- Title case capitalization
- Include variables for clarity: "Download {{ openshift_version }} installer"

### Files
- YAML with `.yml` extension (not `.yaml` for playbooks)
- Templates use `.j2` extension: `haproxy.cfg.j2`, `ignition.j2`
- OS-specific files: `prepare-host-RedHat-8.yml`, `prepare-host-Rocky-9.yml`

## Code Standards

### YAML Formatting
```yaml
# Document start marker required
---
# 2-space indentation
- name: Example task
  ansible.builtin.file:
    path: /example
    state: directory
    mode: '0755'

# Line length: max 180 characters (warning level)
# Truthy values: true, false, yes, no
```

### Task Structure
```yaml
- name: Descriptive task name with {{ variable }} context
  ansible.builtin.module_name:
    parameter: value
  when: conditional_expression
  register: result_variable
  tags:
    - relevant_tag
```

### Idempotency Patterns
```yaml
# Use 'creates' for file operations
- name: Download artifact
  ansible.builtin.get_url:
    url: "{{ artifact_url }}"
    dest: "{{ artifact_path }}"
  args:
    creates: "{{ artifact_path }}"

# Use 'when' guards for conditional execution
- name: Configure service
  ansible.builtin.template:
    src: config.j2
    dest: /etc/service/config
  when: service_enabled | default(false)
```

### Error Handling
```yaml
# Strategic ignore_errors with justification
- name: Attempt optional operation
  ansible.builtin.command: optional-command
  register: result
  ignore_errors: true  # skip_ansible_lint - intentional fallback

# Block/rescue for complex error handling
- name: Critical operation with fallback
  block:
    - name: Primary approach
      ansible.builtin.command: primary-command
  rescue:
    - name: Fallback approach
      ansible.builtin.command: fallback-command
```

## Role Organization

### Standard Role Structure
```
role-name/
├── defaults/main.yml        # Default variables (lowest priority)
├── tasks/
│   ├── main.yml             # Entry point
│   ├── provision.yml        # Logical task grouping
│   ├── configure.yml
│   └── cleanup.yml
├── handlers/main.yml        # Event handlers
├── templates/               # Jinja2 templates
├── vars/                    # Role-specific variables (high priority)
├── meta/main.yml            # Role metadata and dependencies
└── README.md                # Role documentation
```

### Task File Organization
- Split large roles logically: `download.yml`, `create-vms.yml`, `post-install.yml`
- OS-specific variations: `tasks/{{ ansible_distribution }}-{{ ansible_distribution_major_version }}.yml`
- Use `import_tasks` for static inclusion, `include_tasks` for dynamic

## Template Patterns

### Jinja2 Best Practices
```jinja2
{# Comment explaining template purpose #}
{% for item in items %}
{{ item.name }}: {{ item.value }}
{% endfor %}

{# Conditional blocks #}
{% if feature_enabled | default(false) %}
feature_config: enabled
{% endif %}

{# Default values #}
timeout: {{ timeout | default(300) }}
```

### Common Template Types
- HAProxy configuration: `haproxy.cfg.j2`
- Ignition configs: `*.ign.j2`
- SystemD units: `*.service.j2`
- DNS zone files: `zone.j2`

## Configuration Management

### Cluster Configuration (`cluster.yml`)
Single source of truth for cluster-specific settings:
- Hetzner credentials and server details
- Cluster topology (master/compute counts)
- DNS provider configuration
- Authentication options
- Storage preferences
- Network settings (IPv4/IPv6)

### Variables Precedence (lowest to highest)
1. Role defaults (`defaults/main.yml`)
2. Group variables (`group_vars/all/`)
3. Playbook vars_files
4. User cluster configuration (`cluster.yml`)
5. Extra vars (`-e`)

## Multi-OS Support

Supported operating systems for the Hetzner host:
- RHEL 8, RHEL 9
- Rocky Linux 9
- CentOS Stream 9, CentOS Stream 10
- Debian 11

Use OS detection patterns:
```yaml
- name: Include OS-specific tasks
  ansible.builtin.include_tasks: "prepare-host-{{ ansible_distribution }}-{{ ansible_distribution_major_version }}.yml"
```

## Add-On Framework

Post-installation add-ons follow this pattern:
```yaml
# In cluster.yml
cluster_role_bindings:
  - name: ntp
  - name: web-terminal
    vars:
      web_terminal_version: "1.10"
```

Add-on roles in `ansible/add-on-roles/` are dynamically loaded by `run-add-ons.yml`.

## Security Practices

- Use Ansible Vault for sensitive data in `group_vars/all/`
- SSH key-based authentication (password auth disabled)
- Let's Encrypt for production TLS certificates
- Secrets passed via environment variables or vault

## Linting Rules

The project uses `.ansible-lint` with these considerations:
- Skipped rules: `role-name`, `ignore-errors`, `no-changed-when`, `package-latest`
- Use `# noqa: rule-name` for intentional violations with justification
- Pre-commit hooks enforce linting before commits

## Common Patterns

### VM Lifecycle
```yaml
# Start VM
- name: Start {{ vm_name }}
  community.libvirt.virt:
    name: "{{ vm_name }}"
    state: running

# Stop VM gracefully
- name: Shutdown {{ vm_name }}
  community.libvirt.virt:
    name: "{{ vm_name }}"
    state: shutdown
```

### DNS Provider Integration
```yaml
# Multi-provider support via public_dns role
- name: Create DNS record
  ansible.builtin.include_role:
    name: public_dns
  vars:
    dns_action: create
    dns_record: "{{ record_name }}"
    dns_value: "{{ ip_address }}"
```

### OpenShift Operations
```yaml
# Wait for cluster API
- name: Wait for API availability
  kubernetes.core.k8s_info:
    api_version: v1
    kind: Node
  register: nodes
  until: nodes.resources | length > 0
  retries: 60
  delay: 10
```

## Contributing Guidelines

When contributing:
1. Update relevant documentation in `docs/`
2. Add entry to `docs/release-notes.md`
3. Follow existing code patterns and naming conventions
4. Test changes with `ansible-lint`
5. Use descriptive commit messages

## Getting Help

- Check `docs/` for comprehensive guides
- Review `cluster-example.yml` for configuration options
- Consult role-specific `README.md` files
- Open GitHub issues for feature requests or bugs
