# AGENTS.md - DeezChatz Infrastructure Guide

## Overview

This is an Ansible playbook repository for deploying rootless Podman containers (RMQTT broker, Redis) on OCI infrastructure. The playbooks target Red Hat-based systems (Rocky Linux) and use systemd quadlets for container management.

---

## Build, Lint, and Test Commands

### Running Playbooks

```bash
# Run deployment playbook with local config and encrypted secrets
ansible-playbook -i inventory.yml deployment.yml -e @config.yml -e @vault.yml --ask-vault-pass

# Or use a password file to avoid typing it every time
# ansible-playbook -i inventory.yml deployment.yml -e @config.yml -e @vault.yml --vault-password-file .vault_pass

# Run a specific play or task (limit to hosts)
ansible-playbook -i inventory.yml deployment.yml --limit public_instances -e @config.yml -e @vault.yml --ask-vault-pass

# Run with verbose output for debugging
ansible-playbook -i inventory.yml deployment.yml -e @config.yml -e @vault.yml --ask-vault-pass -v
```

### Secrets Management (Ansible Vault)

```bash
# Encrypt the secrets file
ansible-vault encrypt vault.yml

# Edit the encrypted file
ansible-vault edit vault.yml

# View the encrypted file without editing
ansible-vault view vault.yml
```

### Syntax Validation

```bash
# Check playbook syntax (built into ansible-playbook)
ansible-playbook --syntax-check deployment.yml

# Validate all YAML files for syntax
ansible-playbook --syntax-check *.yml

# Lint Ansible playbooks (install via: pip install ansible-lint)
ansible-lint deployment.yml
```

### YAML Validation

```bash
# Validate YAML syntax
python3 -c "import yaml; yaml.safe_load(open('deployment.yml'))"

# Or use yamllint if installed
yamllint .
```

### Dry Run (Check Mode)

```bash
# Run in check mode (no actual changes)
ansible-playbook -i inventory.yml deployment.yml -e @config.yml --check

# Check mode with diff
ansible-playbook -i inventory.yml deployment.yml -e @config.yml --check --diff
```

---

## Code Style Guidelines

### YAML Structure

- Use 2-space indentation (no tabs)
- Always use explicit `---` document start separator
- Add blank lines between plays and major sections for readability
- Use comments to document sections (`# ----- Section Name -----`)

### Naming Conventions

- **Play names**: Use Title Case (`Deploy Rootless RMQTT Broker`)
- **Task names**: Use Title Case (`Install Podman`)
- **Variable names**: Use snake_case (`ansible_user`, `rocky_uid`)
- **Host groups**: Use snake_case (`public_instances`, `private_instances`)
- **File paths**: Use snake_case for custom files (`rmqtt.container`)

### Playbook Organization

```yaml
---
# Play 1: System-Level Setup (Runs as Root)
- name: Setup Podman and Rootless Environment
  hosts: all
  become: yes  # Use explicit 'yes' for privilege escalation
  tasks:
    - name: Install Podman
      dnf:
        name: podman
        state: present

# Play 2: User-Level Services
- name: Deploy Services
  hosts: service_hosts
  become: no
  tasks:
    # ...
```

### Task Modules

- Prefer specific modules over shell/command when possible (`dnf`, `file`, `copy`, `sysctl`)
- Always use `ansible.builtin.` prefix for built-in modules for clarity
- Use `changed_when: false` for idempotent commands that don't modify state
- Use `ignore_errors: yes` sparingly and only with comments explaining why

### Handlers

- Define handlers at the play level with `handlers:` keyword
- Use notify to trigger handler execution
- Handlers should have descriptive names (`Reload and start RMQTT`)

### Variables

- Use descriptive variable names
- Define host variables in `inventory.yml` under `vars:` section
- Use `ansible_user`, `ansible_python_interpreter` for connection settings
- Reference variables with `{{ variable_name }}` (always quote strings with variables)

### Error Handling

- Use `ignore_errors: yes` only when necessary (e.g., optional dependencies)
- Use `failed_when:` for conditional failure conditions
- Use `register:` to capture command output and condition on it
- Always set `changed_when: false` for read-only commands

### Multi-Architecture Images

- Explicitly specify architecture tags for ARM64 servers:
  ```yaml
  Image=docker.io/rmqtt/rmqtt:latest-arm64
  ```
- For multi-arch images, add comments noting this:
  ```yaml
  # Redis official images are multi-arch; ARM64 is pulled automatically
  Image=docker.io/library/redis:7-alpine
  ```

### Quadlet Files

- Use absolute paths for user-specific quadlets: `/home/rocky/.config/containers/systemd/`
- Use `creates:` argument to prevent re-running idempotent commands
- Always include `[Service]`, `[Install]` sections with `Restart=always`

### Security

- Never commit secrets to version control
- Use `vault.yml` for sensitive variables and encrypt it with Ansible Vault
- Use `config.yml` for non-sensitive local variables (add both to `.gitignore`)
- Use `example.config.yml` and `example.vault.yml` as templates for your local setup
- Use `become: yes` only when necessary (prefer rootless where possible)
- Avoid storing credentials in inventory files

---

## Project Structure

```
ansible/
├── AGENTS.md              # This file
├── inventory.yml         # Ansible inventory with host groups
├── config.yml            # Non-sensitive local variables (GITIGNORED)
├── vault.yml             # Encrypted secrets (GITIGNORED)
├── example.config.yml    # Placeholder/template for config.yml
├── example.vault.yml     # Placeholder/template for vault.yml
├── deployment.yml        # Main deployment playbook
└── roles/                 # Modular Ansible roles
    ├── common/
    ├── aws_cli/
    ├── rmqtt/
    ├── redis/
    ├── api/
    └── nginx/
```

---

## Common Patterns

### Checking for Idempotency

```yaml
- name: Enable systemd lingering
  command: loginctl enable-linger rocky
  args:
    creates: "/var/lib/systemd/linger/rocky"  # Skip if already exists
```

### User systemd services

```yaml
- name: Reload service
  ansible.builtin.systemd:
    name: redis.service
    scope: user          # Required for rootless containers
    state: restarted
    enabled: yes
    daemon_reload: yes
```

### Loops with ignore_errors

```yaml
- name: Stop services
  systemd:
    name: "{{ item }}"
    state: stopped
  loop:
    - rmqtt.service
    - redis.service
  ignore_errors: yes  # Service may not exist
```
