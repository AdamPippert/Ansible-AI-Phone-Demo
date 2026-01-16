# CallAnsible - Voice-Controlled Ansible Automation Platform

Voice-controlled Ansible Automation Platform (AAP) via self-hosted Asterisk PBX, local LLM inference, and containerized infrastructure. Designed for live conference demos.

**Demo scenario:** Speaker calls their own infrastructure from stage, speaks natural language commands, audience watches AAP dashboard execute playbooks in real-time.

**Extended scenario (Storm Mode):** Event-driven automation where a weather alert triggers an OpenProse incident workflow, with Daytona providing sandboxed validation and EDA routing events to AAP job templates.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Framework 16 (Podium) - 192.168.0.218 / Tailscale: 100.x.x.x      │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Podman Pod: callansible                   │   │
│  │                                                              │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐ │   │
│  │  │ asterisk  │  │ voice-    │  │ vllm      │  │ aap      │ │   │
│  │  │           │  │ bridge    │  │           │  │          │ │   │
│  │  │ Port 5060 │  │ Port 8080 │  │ Port 8000 │  │ Port 443 │ │   │
│  │  │ UDP/TCP   │  │ HTTP/WS   │  │ HTTP      │  │ HTTPS    │ │   │
│  │  └───────────┘  └───────────┘  └───────────┘  └──────────┘ │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Browser: AAP Dashboard (visible to audience)                       │
└─────────────────────────────────────────────────────────────────────┘
        ▲
        │ SIP over Tailscale VPN
        │
┌───────┴───────┐
│ Speaker Phone │
│ (Zoiper/      │
│  Linphone)    │
└───────────────┘
```

## Hardware Requirements

- **Framework Desktop 16**
  - Ryzen 9 CPU
  - NVIDIA RTX 5070 Mobile (8GB GDDR7 VRAM)
  - Fedora 43+
  - Podman + nvidia-container-toolkit

## Quick Start

### 1. Set Up Credentials

```bash
cd provisioning

# Create credentials file from template
cp credentials.yml.example credentials.yml

# Edit with your passwords
vim credentials.yml

# Secure the file
chmod 600 credentials.yml
```

**credentials.yml format:**
```yaml
credentials:
  username: adam                      # SSH username
  password: "your_ssh_password"       # SSH/sudo password
  luks_password: "your_luks_password" # LUKS encryption password (if used)
```

> **IMPORTANT:** Never commit `credentials.yml` to version control. It is already in `.gitignore`.

### 2. Install Ansible Requirements

```bash
# Install required collections
ansible-galaxy collection install ansible.posix containers.podman
```

### 3. Run Provisioning

```bash
cd provisioning

# Full provisioning (firmware → OS → Podman → Services)
ansible-playbook -i inventory.yml site.yml -e @credentials.yml

# Or run specific stages:
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags firmware
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags os
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags podman
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags services
```

### 4. Configure Application Secrets

After provisioning, on the Framework 16 machine:

```bash
cd /home/adam/Development/Ansible-AI-Phone-Demo/CallAnsible

# Copy environment template
cp .env.example .env

# Edit with your secrets
vim .env
```

### 5. Download Models & Start Services

```bash
# Download Piper TTS voice model
./scripts/download-models.sh

# Start all services
podman-compose up -d

# Verify services are running
./scripts/test-call.sh
```

### 6. Configure SIP Client

On your phone (Zoiper, Linphone, etc.):

| Setting | Value |
|---------|-------|
| Server | `<tailscale-ip>:5060` |
| Username | `speaker` |
| Password | from `.env` `SIP_PASSWORD` |
| Extension | `100` |

## Provisioning Tags

| Tag | Description |
|-----|-------------|
| `firmware` | BIOS and component firmware updates (runs first, may require reboot) |
| `os` | System packages, NVIDIA drivers, Tailscale, firewall |
| `nvidia` | NVIDIA GPU drivers and container toolkit only |
| `tailscale` | Tailscale VPN setup only |
| `podman` | Podman container runtime and GPU integration |
| `services` | CallAnsible application and container configurations |

## Project Structure

```
Ansible-AI-Phone-Demo/
├── README.md                 # This file
├── SPEC.md                   # Technical specification
├── .gitignore
│
├── provisioning/             # Ansible provisioning
│   ├── site.yml              # Main playbook
│   ├── inventory.yml         # Host inventory
│   ├── credentials.yml.example
│   ├── group_vars/all.yml
│   └── roles/
│       ├── firmware-update/  # BIOS/component updates
│       ├── os-setup/         # OS packages, NVIDIA, Tailscale
│       ├── podman-setup/     # Container runtime
│       └── demo-services/    # Application configs
│
└── CallAnsible/              # Application (created by provisioning)
    ├── docker-compose.yml
    ├── .env.example
    ├── asterisk/             # SIP/PBX config
    ├── voice-bridge/         # Python orchestration app
    ├── playbooks/            # Demo Ansible playbooks
    ├── eda/                  # Event-Driven Ansible
    ├── openprose/            # Incident workflows
    ├── daytona/              # Sandbox config
    ├── models/piper/         # TTS models
    ├── audio/                # Runtime audio files
    └── scripts/              # Helper scripts
```

## Demo Playbooks

| Playbook | Description |
|----------|-------------|
| `health_check.yml` | Check infrastructure health status |
| `scale_deployment.yml` | Scale a deployment tier |
| `self_healing_workflow.yml` | Deploy self-healing agents |
| `storm_mode_precheck.yml` | Validate before storm mode |
| `storm_mode_apply.yml` | Activate emergency ACLs |
| `storm_mode_verify.yml` | Verify storm mode is working |
| `storm_mode_rollback.yml` | Restore normal operations |

## Voice Commands (Examples)

- "What's the current health status?"
- "Deploy the self-healing workflow for production"
- "Scale the web tier to 5 instances"
- "Activate storm mode for the east region"

## Storm Mode (Event-Driven Automation)

Trigger storm mode with a weather alert:

```bash
# Inject weather alert
./scripts/inject-weather-alert.sh activate

# View EDA logs
podman-compose logs -f eda

# Send all-clear to rollback
./scripts/inject-weather-alert.sh clear
```

## Firmware Updates

The provisioning playbook automatically checks for and applies firmware updates using fwupd/LVFS:

- BIOS updates
- Embedded Controller updates
- GPU firmware
- Other component firmware

If firmware updates are staged, a reboot is required. The playbook will notify you. After reboot, re-run the playbook to continue.

To run firmware updates only:

```bash
ansible-playbook -i inventory.yml site.yml -e @credentials.yml --tags firmware
```

## Troubleshooting

### Check Service Status

```bash
podman-compose ps
podman-compose logs -f <service>
```

### Test ARI Connection

```bash
source .env
curl -u voice-bridge:$ARI_PASSWORD http://localhost:8088/ari/asterisk/info
```

### Test SIP Registration

```bash
podman exec asterisk asterisk -rx "pjsip show endpoints"
```

### Check GPU Access

```bash
podman run --rm --device nvidia.com/gpu=all docker.io/nvidia/cuda:12.0-base nvidia-smi
```

## Security Notes

- `credentials.yml` contains sensitive data - never commit to git
- `.env` files contain service passwords - secured with `chmod 600`
- Tailscale provides encrypted VPN for SIP traffic
- AAP job templates are the only approved automation actions
- Daytona sandboxes provide egress controls for validation steps

## References

- [SPEC.md](./SPEC.md) - Full technical specification
- [Framework Knowledge Base](https://knowledgebase.frame.work/)
- [fwupd Documentation](https://fwupd.org/)
- [Ansible Automation Platform](https://www.ansible.com/products/automation-platform)
