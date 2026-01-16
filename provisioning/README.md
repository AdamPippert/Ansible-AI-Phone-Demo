# CallAnsible Provisioning Playbook

Ansible playbook to provision a Framework 16 machine for the CallAnsible voice-controlled Ansible Automation Platform demo.

## Prerequisites

- Fedora 40+ on target machine (framework16 at 192.168.0.218)
- Ansible 2.14+ on control machine
- SSH access to target machine
- `ansible.posix` and `containers.podman` collections installed

## Quick Start

### 1. Create Credentials File

The playbook requires a credentials file with SSH and system passwords:

```bash
cd provisioning

# Copy the example file
cp credentials.yml.example credentials.yml

# Edit with your actual passwords
vim credentials.yml

# Secure the file permissions
chmod 600 credentials.yml
```

**credentials.yml format:**
```yaml
credentials:
  # SSH username for the Framework 16 machine
  username: adam

  # SSH/sudo password for the user
  password: "your_actual_ssh_password"

  # LUKS disk encryption password (if applicable)
  luks_password: "your_actual_luks_password"
```

> **SECURITY WARNING:** Never commit `credentials.yml` to version control! It is already listed in `.gitignore`.

### 2. Install Ansible Collections

```bash
ansible-galaxy collection install ansible.posix containers.podman
```

### 3. Run the Playbook

```bash
cd provisioning

# Full provisioning
ansible-playbook -i inventory.yml site.yml -e @credentials.yml

# Or run specific tags
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags firmware
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags os
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags nvidia
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags podman
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags services
```

## Integration with Lab Inventory

If `framework16` is already in your lab inventory, you can use this playbook directly:

```bash
# Using your existing inventory (still need credentials)
ansible-playbook -i /path/to/lab/inventory site.yml -l framework16 -e @credentials.yml
```

Or include the roles in your existing playbooks:

```yaml
- hosts: framework16
  roles:
    - path/to/provisioning/roles/firmware-update
    - path/to/provisioning/roles/os-setup
    - path/to/provisioning/roles/podman-setup
    - path/to/provisioning/roles/demo-services
```

## What Gets Installed

### Firmware Update Role (runs first)
- BIOS updates via fwupd/LVFS
- Embedded Controller firmware
- GPU firmware updates
- Other component firmware
- **Note:** May require reboot before continuing

### OS Setup Role
- System packages (git, curl, jq, python3, etc.)
- NVIDIA drivers and container toolkit
- Tailscale VPN
- Firewall rules for all services

### Podman Setup Role
- Podman and podman-compose
- Rootless container configuration
- NVIDIA GPU integration for containers
- Pre-pulled container images

### Demo Services Role
- CallAnsible directory structure
- Asterisk configuration (pjsip, extensions, ARI)
- Voice-bridge scaffolding
- Demo playbooks (health_check, scale_deployment, self_healing_workflow)
- Storm mode playbooks (precheck, apply, verify, rollback)
- EDA rulebooks and test events
- Daytona sandbox configuration
- OpenProse incident workflow
- docker-compose.yml for all services
- Helper scripts

## Directory Structure Created

```
CallAnsible/
├── docker-compose.yml
├── .env.example
├── .gitignore
├── asterisk/
│   ├── pjsip.conf
│   ├── extensions.conf
│   ├── ari.conf
│   ├── http.conf
│   └── rtp.conf
├── voice-bridge/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── config.py
│   ├── prompts.py
│   └── *.py (placeholder files)
├── playbooks/
│   ├── self_healing_workflow.yml
│   ├── health_check.yml
│   ├── scale_deployment.yml
│   ├── storm_mode_precheck.yml
│   ├── storm_mode_apply.yml
│   ├── storm_mode_verify.yml
│   └── storm_mode_rollback.yml
├── eda/
│   ├── rulebooks/storm_mode.yml
│   ├── inventory/hosts.yml
│   └── test_events/*.json
├── openprose/
│   ├── storm_mode_incident.prose
│   └── schemas/weather_alert.schema.json
├── daytona/
│   ├── policy.yml
│   └── mcp_config.json
├── models/piper/
├── audio/
├── aap-data/
└── scripts/
    ├── setup.sh
    ├── download-models.sh
    ├── test-call.sh
    └── inject-weather-alert.sh
```

## Post-Provisioning Steps

1. **Configure secrets:**
   ```bash
   cd /home/adam/Development/Ansible-AI-Phone-Demo/CallAnsible
   cp .env.example .env
   vim .env  # Fill in your passwords
   ```

2. **Download Piper TTS model:**
   ```bash
   ./scripts/download-models.sh
   ```

3. **Start services:**
   ```bash
   podman-compose up -d
   ```

4. **Verify services:**
   ```bash
   ./scripts/test-call.sh
   ```

5. **Configure SIP client:**
   - Server: `<tailscale-ip>:5060`
   - Username: `speaker`
   - Password: from `.env` `SIP_PASSWORD`
   - Extension to dial: `100`

## Tags

| Tag | Description |
|-----|-------------|
| `firmware` | BIOS and component firmware updates (runs first) |
| `os` | System packages and configuration |
| `nvidia` | NVIDIA drivers and container toolkit |
| `tailscale` | Tailscale VPN setup |
| `podman` | Podman container runtime |
| `services` | Demo services and configurations |

## Variables

Key variables can be overridden in `group_vars/all.yml` or via extra-vars:

```bash
ansible-playbook site.yml -e @credentials.yml -e "vllm_gpu_memory_utilization=0.60" -e "whisper_model=medium.en"
```

See `group_vars/all.yml` for all configurable options.

## Firmware Update Notes

The firmware-update role uses fwupd/LVFS to update:
- System BIOS
- Embedded Controller
- GPU firmware
- Other components

**Important considerations:**
- AC power must be connected for firmware updates
- The LVFS testing channel is enabled by default (Framework releases there first)
- If updates are staged, a reboot is required
- After reboot, re-run the playbook to continue provisioning

To skip firmware updates:
```bash
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --skip-tags firmware
```

## Reboot Notice

Reboots may be required for:
- Firmware updates (BIOS, EC, GPU)
- NVIDIA driver installation

The playbook will notify you when a reboot is needed. After rebooting, re-run the playbook to continue.

## Troubleshooting

### SSH Connection Issues
```bash
# Test SSH connectivity
ssh adam@192.168.0.218

# Verify credentials file is correct
cat credentials.yml
```

### Ansible Errors
```bash
# Run with verbose output
ansible-playbook -i inventory.yml site.yml -e @credentials.yml -vvv
```

### Firmware Update Issues
```bash
# Check fwupd status on target
ssh adam@192.168.0.218 'sudo fwupdmgr get-devices'
ssh adam@192.168.0.218 'sudo fwupdmgr get-updates'
```
