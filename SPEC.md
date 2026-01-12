# CallAnsible - Technical Specification

## Overview

Voice-controlled Ansible Automation Platform (AAP) via self-hosted Asterisk PBX, local LLM inference, and containerized infrastructure. Designed for live conference demo on January 29, 2025.

**Demo scenario:** Speaker calls their own infrastructure from stage, speaks natural language commands, audience watches AAP dashboard execute playbooks in real-time.

**Extended scenario (Storm Mode):** Event-driven automation where a weather alert triggers an OpenProse incident workflow, with Daytona providing sandboxed validation and EDA routing events to AAP job templates.

-----

## Hardware

- **Framework Desktop 16**
  - Ryzen 9 CPU (integrated Radeon graphics - unused for inference)
  - NVIDIA RTX 5070 Mobile (8GB GDDR7 VRAM)
  - Fedora 43
  - Podman + nvidia-container-toolkit

-----

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Framework 16 (Podium) - Tailscale IP: 100.x.x.x                   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Podman Pod: callansible                   │   │
│  │                                                              │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐ │   │
│  │  │ asterisk  │  │ voice-    │  │ vllm      │  │ aap      │ │   │
│  │  │           │  │ bridge    │  │           │  │          │ │   │
│  │  │ Port 5060 │  │ Port 8080 │  │ Port 8000 │  │ Port 443 │ │   │
│  │  │ UDP/TCP   │  │ HTTP/WS   │  │ HTTP      │  │ HTTPS    │ │   │
│  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └────┬─────┘ │   │
│  │        │              │              │              │       │   │
│  │        │     ARI      │   OpenAI API │    AAP API   │       │   │
│  │        └──────────────┴──────────────┴──────────────┘       │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌───────────────┐  ┌───────────────┐                              │
│  │ faster-whisper│  │ piper-tts     │  (CPU, outside pod or in    │
│  │ (STT)         │  │ (TTS)         │   voice-bridge container)    │
│  └───────────────┘  └───────────────┘                              │
│                                                                     │
│  Browser: AAP Dashboard (visible to audience)                       │
└─────────────────────────────────────────────────────────────────────┘
        ▲
        │ SIP over Tailscale VPN
        │
┌───────┴───────┐
│ Speaker Phone │
│ - Omate watch │
│   or          │
│ - Zoiper/     │
│   Linphone    │
│ - Tailscale   │
│   connected   │
└───────────────┘
```

-----

## Container Definitions

### 1. vllm

**Purpose:** Local LLM inference with OpenAI-compatible API

**Image:** `docker.io/vllm/vllm-openai:latest`

**Resources:**

- GPU: NVIDIA RTX 5070 (8GB VRAM)
- Memory limit: 16GB

**Command:**

```bash
--model Qwen/Qwen2.5-7B-Instruct-AWQ \
--quantization awq \
--gpu-memory-utilization 0.55 \
--max-model-len 4096 \
--port 8000
```

**Environment:**

```
NVIDIA_VISIBLE_DEVICES=all
```

**Ports:** 8000:8000

**Health check:** `GET http://localhost:8000/health`

-----

### 2. asterisk

**Purpose:** SIP PBX for voice calls

**Base image:** `docker.io/andrius/asterisk:latest` or custom Fedora-based

**Ports:**

- 5060:5060/udp (SIP signaling)
- 5060:5060/tcp (SIP signaling)
- 8088:8088/tcp (ARI HTTP)
- 8089:8089/tcp (ARI WebSocket)
- 10000-10100:10000-10100/udp (RTP media)

**Volumes:**

- `./asterisk/pjsip.conf:/etc/asterisk/pjsip.conf:ro`
- `./asterisk/extensions.conf:/etc/asterisk/extensions.conf:ro`
- `./asterisk/ari.conf:/etc/asterisk/ari.conf:ro`
- `./asterisk/http.conf:/etc/asterisk/http.conf:ro`
- `./asterisk/rtp.conf:/etc/asterisk/rtp.conf:ro`
- `./audio:/var/lib/asterisk/sounds/custom:rw`

**Network:** Host mode (simplifies RTP)

-----

### 3. voice-bridge

**Purpose:** Orchestrates call flow via ARI, STT, LLM, TTS

**Base image:** `python:3.11-slim`

**Ports:** 8080:8080

**Environment:**

```
ASTERISK_ARI_URL=http://localhost:8088
ASTERISK_ARI_USER=voice-bridge
ASTERISK_ARI_PASSWORD=${ARI_PASSWORD}
VLLM_URL=http://localhost:8000
AAP_URL=https://localhost:443
AAP_USER=admin
AAP_PASSWORD=${AAP_PASSWORD}
PIPER_MODEL_PATH=/models/piper/en_US-lessac-medium.onnx
WHISPER_MODEL=small.en
WHISPER_DEVICE=cpu
```

**Volumes:**

- `./models/piper:/models/piper:ro`
- `./audio:/audio:rw`

**Dependencies (requirements.txt):**

```
fastapi>=0.109.0
uvicorn>=0.27.0
ari-py>=0.1.3
websockets>=12.0
httpx>=0.26.0
faster-whisper>=1.0.0
piper-tts>=1.2.0
numpy>=1.26.0
soundfile>=0.12.0
python-dotenv>=1.0.0
```

-----

### 4. aap (Ansible Automation Platform)

**Purpose:** Execute Ansible playbooks, provide visual dashboard

**Image:** `quay.io/ansible/awx:latest` (or your AAP image)

**Ports:** 443:443, 8043:8043

**Volumes:**

- `./playbooks:/var/lib/awx/projects:rw`
- `./aap-data:/var/lib/awx/data:rw`

**Note:** AWX is the upstream open-source version. If you have AAP subscription, use that image instead.

-----

## File Structure

```
CallAnsible/
├── SPEC.md                          # This file
├── README.md                        # User-facing documentation
├── docker-compose.yml               # Podman compose orchestration
├── .env.example                     # Environment template
├── .gitignore
│
├── asterisk/
│   ├── Dockerfile                   # Custom Asterisk image (optional)
│   ├── pjsip.conf                   # SIP endpoint configuration
│   ├── extensions.conf              # Dialplan
│   ├── ari.conf                     # ARI user configuration
│   ├── http.conf                    # HTTP server for ARI
│   └── rtp.conf                     # RTP port range
│
├── voice-bridge/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py                      # FastAPI app entry point
│   ├── config.py                    # Environment/settings
│   ├── ari_client.py                # Asterisk ARI integration
│   ├── stt.py                       # faster-whisper wrapper
│   ├── tts.py                       # Piper TTS wrapper
│   ├── llm.py                       # vLLM client
│   ├── aap.py                       # AAP API client
│   ├── call_handler.py              # Call state machine
│   └── prompts.py                   # LLM system prompts
│
├── models/
│   └── piper/
│       └── .gitkeep                 # Downloaded at build/runtime
│
├── playbooks/
│   ├── self_healing_workflow.yml    # Demo playbook 1
│   ├── health_check.yml             # Demo playbook 2
│   └── scale_deployment.yml         # Demo playbook 3
│
├── audio/
│   └── .gitkeep                     # Runtime audio files
│
└── scripts/
    ├── setup.sh                     # Initial setup script
    ├── download-models.sh           # Download Piper voice models
    └── test-call.sh                 # Test SIP registration
```

-----

## Asterisk Configuration

### pjsip.conf

```ini
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

[transport-tcp]
type=transport
protocol=tcp
bind=0.0.0.0:5060

; Speaker's phone endpoint
[speaker-phone]
type=endpoint
context=callansible
disallow=all
allow=ulaw
allow=alaw
allow=opus
auth=speaker-phone-auth
aors=speaker-phone-aor
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[speaker-phone-auth]
type=auth
auth_type=userpass
username=speaker
password=${SIP_PASSWORD}

[speaker-phone-aor]
type=aor
max_contacts=1
remove_existing=yes
```

### extensions.conf

```ini
[callansible]
; Main entry point - hand off to ARI
exten => 100,1,NoOp(Incoming call to CallAnsible)
 same => n,Answer()
 same => n,Stasis(voice-bridge)
 same => n,Hangup()

; Direct extension for testing
exten => 200,1,NoOp(Test extension)
 same => n,Answer()
 same => n,Playback(hello-world)
 same => n,Hangup()
```

### ari.conf

```ini
[general]
enabled=yes
pretty=yes

[voice-bridge]
type=user
read_only=no
password=${ARI_PASSWORD}
```

### http.conf

```ini
[general]
enabled=yes
bindaddr=0.0.0.0
bindport=8088

[websocket]
enabled=yes
```

### rtp.conf

```ini
[general]
rtpstart=10000
rtpend=10100
```

-----

## Voice Bridge API Contract

### ARI Event Flow

```
1. StasisStart event → Call enters Stasis app
2. voice-bridge answers, plays greeting via ARI
3. Record audio (silence detection)
4. Send audio to faster-whisper → text
5. Send text to vLLM → response + tool calls
6. If tool call: execute AAP job, wait for result
7. Send response text to Piper → audio file
8. Play audio via ARI
9. Loop to step 3 until hangup or timeout
```

### LLM System Prompt (prompts.py)

```python
SYSTEM_PROMPT = """You are an infrastructure automation assistant controlling Ansible Automation Platform.

When the user requests infrastructure changes, respond with a JSON tool call.

Available tools:

1. launch_job_template
   - template_name: string (one of: "self_healing_workflow", "health_check", "scale_deployment")
   - extra_vars: object (optional variables to pass)

2. get_job_status
   - job_id: integer

3. speak_only
   - message: string (just respond verbally, no action)

Examples:

User: "Deploy the self-healing workflow for production"
Response: {"tool": "launch_job_template", "template_name": "self_healing_workflow", "extra_vars": {"tier": "production"}}

User: "What's the current health status?"
Response: {"tool": "launch_job_template", "template_name": "health_check", "extra_vars": {}}

User: "Scale the web tier to 5 instances"
Response: {"tool": "launch_job_template", "template_name": "scale_deployment", "extra_vars": {"target": "web", "replicas": 5}}

User: "Thanks, that's all"
Response: {"tool": "speak_only", "message": "You're welcome. Call back anytime."}

Always respond with valid JSON. Be concise in spoken responses - the user is on a phone call.
"""
```

### AAP API Integration (aap.py)

```python
# Key endpoints

# List job templates
GET /api/v2/job_templates/

# Launch job template
POST /api/v2/job_templates/{id}/launch/
Body: {"extra_vars": {...}}

# Get job status
GET /api/v2/jobs/{id}/

# Job stdout (for detailed results)
GET /api/v2/jobs/{id}/stdout/?format=txt
```

-----

## Environment Variables

### .env.example

```bash
# SIP
SIP_PASSWORD=changeme_sip_password

# Asterisk ARI
ARI_PASSWORD=changeme_ari_password

# AAP
AAP_URL=https://localhost
AAP_USER=admin
AAP_PASSWORD=changeme_aap_password

# vLLM
VLLM_MODEL=Qwen/Qwen2.5-7B-Instruct-AWQ

# Tailscale (informational)
TAILSCALE_IP=100.x.x.x
```

-----

## Call State Machine

```
┌─────────────┐
│   IDLE      │
└──────┬──────┘
       │ StasisStart
       ▼
┌─────────────┐
│  GREETING   │ ──► Play: "Infrastructure control ready."
└──────┬──────┘
       │ Playback complete
       ▼
┌─────────────┐
│  LISTENING  │ ◄─────────────────────────────────┐
└──────┬──────┘                                   │
       │ Silence detected                         │
       ▼                                          │
┌─────────────┐                                   │
│ TRANSCRIBING│ ──► faster-whisper                │
└──────┬──────┘                                   │
       │ Text ready                               │
       ▼                                          │
┌─────────────┐                                   │
│  THINKING   │ ──► vLLM inference                │
└──────┬──────┘                                   │
       │ Response ready                           │
       ▼                                          │
┌─────────────────┐                               │
│ EXECUTING (opt) │ ──► AAP API call              │
└──────┬──────────┘                               │
       │ Job complete                             │
       ▼                                          │
┌─────────────┐                                   │
│ SYNTHESIZING│ ──► Piper TTS                     │
└──────┬──────┘                                   │
       │ Audio ready                              │
       ▼                                          │
┌─────────────┐                                   │
│  SPEAKING   │ ──► Play audio ───────────────────┘
└──────┬──────┘
       │ Hangup detected
       ▼
┌─────────────┐
│   ENDED     │
└─────────────┘
```

-----

## Demo Playbooks

### playbooks/self_healing_workflow.yml

```yaml
---
- name: Deploy Self-Healing Workflow
  hosts: localhost
  gather_facts: false
  vars:
    tier: "{{ tier | default('production') }}"

  tasks:
    - name: Display deployment start
      debug:
        msg: "Deploying self-healing workflow for {{ tier }} tier"

    - name: Simulate self-healing agent deployment
      pause:
        seconds: 5

    - name: Configure monitoring endpoints
      debug:
        msg: "Monitoring endpoints configured for {{ tier }}"

    - name: Enable auto-recovery policies
      debug:
        msg: "Auto-recovery policies enabled"

    - name: Deployment complete
      debug:
        msg: "Self-healing workflow active on {{ tier }} tier"
```

### playbooks/health_check.yml

```yaml
---
- name: Infrastructure Health Check
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Check node status
      debug:
        msg: "All nodes healthy"

    - name: Check CPU utilization
      set_fact:
        cpu_usage: "{{ range(8, 20) | random }}"

    - name: Check memory utilization
      set_fact:
        memory_usage: "{{ range(25, 45) | random }}"

    - name: Report status
      debug:
        msg: "CPU: {{ cpu_usage }}%, Memory: {{ memory_usage }}%, No alerts"
```

### playbooks/scale_deployment.yml

```yaml
---
- name: Scale Deployment
  hosts: localhost
  gather_facts: false
  vars:
    target: "{{ target | default('web') }}"
    replicas: "{{ replicas | default(3) }}"

  tasks:
    - name: Display scaling operation
      debug:
        msg: "Scaling {{ target }} tier to {{ replicas }} replicas"

    - name: Simulate scaling
      pause:
        seconds: 3

    - name: Scaling complete
      debug:
        msg: "{{ target }} tier now running {{ replicas }} replicas"
```

-----

## Latency Budget

Target: < 5 seconds end-to-end for natural conversation feel

|Stage                             |Target     |Notes                        |
|----------------------------------|-----------|-----------------------------|
|Audio capture + silence detection |800ms      |Configurable                 |
|STT (faster-whisper small.en, CPU)|1500ms     |For ~5s audio                |
|LLM inference (Qwen2.5-7B, GPU)   |1000ms     |~50 tokens                   |
|AAP job launch                    |500ms      |API call only, job runs async|
|TTS (Piper, CPU)                  |500ms      |~20 words                    |
|Audio playback                    |Variable   |Depends on response length   |
|**Total (excluding playback)**    |**~4300ms**|Within budget                |

-----

## Testing Checklist

### Pre-demo (Jan 28)

- [ ] Full cold boot → all containers healthy in < 2 minutes
- [ ] SIP registration from phone succeeds
- [ ] Call connects, greeting plays
- [ ] Voice command transcribes correctly
- [ ] LLM returns valid tool call
- [ ] AAP job launches and completes
- [ ] Response speaks correctly
- [ ] Multi-turn conversation works (3+ exchanges)
- [ ] Graceful hangup

### Day-of (Jan 29)

- [ ] Tailscale connected on Framework
- [ ] Tailscale connected on phone
- [ ] Test call 10 minutes before talk
- [ ] Browser open to AAP dashboard
- [ ] Audio output routed to venue PA (optional)

-----

## Failure Modes & Mitigations

|Failure             |Detection               |Mitigation                                  |
|--------------------|------------------------|--------------------------------------------|
|vLLM OOM            |Container restart       |Reduce gpu-memory-utilization to 0.50       |
|STT timeout         |No transcription in 10s |Prompt: "I didn't catch that, please repeat"|
|LLM invalid JSON    |Parse error             |Retry once, then generic error response     |
|AAP job fails       |Job status != successful|Report failure verbally, continue           |
|Tailscale disconnect|SIP registration fails  |Mobile hotspot backup                       |
|Asterisk crash      |Container unhealthy     |Auto-restart policy                         |

-----

## Commands Reference

### Start stack

```bash
podman-compose up -d
```

### View logs

```bash
podman-compose logs -f voice-bridge
podman-compose logs -f asterisk
```

### Test SIP registration

```bash
podman exec asterisk asterisk -rx "pjsip show endpoints"
```

### Test ARI connection

```bash
curl -u voice-bridge:${ARI_PASSWORD} http://localhost:8088/ari/asterisk/info
```

### Download Piper model

```bash
./scripts/download-models.sh
```

-----

## Dependencies to Install on Framework

```bash
# Container runtime
sudo dnf install podman podman-compose nvidia-container-toolkit

# Tailscale
sudo dnf install tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up

# Verify GPU
nvidia-smi
```

-----

## SIP Client Configuration (Phone)

### Zoiper / Linphone Settings

- **Domain/Server:** [Tailscale IP of Framework]:5060
- **Username:** speaker
- **Password:** [SIP_PASSWORD from .env]
- **Transport:** UDP (or TCP)
- **Extension to dial:** 100

### Omate TrueSmart (if SIP capable)

Same settings, entered via watch interface.

-----

## OpenProse Integration (Input Level Isolation)

### Overview

OpenProse provides structured incident orchestration with clear boundaries between AI decision-making and infrastructure execution. The key principle: **the LLM accelerates decision-making, but the platform enforces safe, pre-approved execution.**

OpenProse runs as a Claude Code plugin (beta status - appropriate for demos and POCs).

### Component Responsibilities

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CONTROL FLOW ARCHITECTURE                            │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │   EVENT     │    │  OPENPROSE  │    │   DAYTONA   │    │    EDA      │  │
│  │   SOURCE    │───►│   SESSION   │───►│   SANDBOX   │───►│  RULEBOOK   │  │
│  │             │    │             │    │             │    │             │  │
│  │ Weather API │    │ Ingest      │    │ Validation  │    │ Deterministic│ │
│  │ Monitoring  │    │ Summarize   │    │ Schema check│    │ Policy gate │  │
│  │ Manual      │    │ Extract     │    │ Diff render │    │             │  │
│  └─────────────┘    │ Propose     │    │ Egress ctrl │    │ run_job_    │  │
│                     └─────────────┘    └─────────────┘    │ template    │  │
│                                                           └──────┬──────┘  │
│                                                                  │         │
│                     ┌────────────────────────────────────────────┘         │
│                     ▼                                                       │
│              ┌─────────────┐                                                │
│              │     AAP     │                                                │
│              │             │                                                │
│              │ Job Template│  ◄── Approved actions only                     │
│              │ RBAC/Audit  │  ◄── Token least privilege                     │
│              │ Execution   │  ◄── Full audit trail                          │
│              └─────────────┘                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**EDA (Event-Driven Ansible):** Event router with deterministic gates. Ingests alerts, matches conditions, triggers `run_job_template` actions.

**AAP Job Templates:** Governance boundary. Repeatable, RBAC-controlled, auditable "approved actions" for emergency changes.

**OpenProse:** Incident orchestration session logic. Readable workflow for multi-step handling: ingest → assess → propose → validate → approve → execute → verify → rollback.

**Daytona:** Safe compute plane for analysis and validation. Explicit network egress limiting (`networkBlockAll` / `networkAllowList`) prevents data exfiltration and reduces attack surface.

### Storm Mode Use Case

**Narrative:** A severe weather alert threatens power/ISP stability for a region. The system quickly shifts the network into "storm mode" to protect critical services:

- Restrict nonessential traffic
- Preserve bandwidth for critical apps (VPN, VoIP, monitoring, management)
- Verify health
- Roll back when the alert clears

**What the LLM contributes:**

- Parse messy alerts and extract structured intent (`region=X, severity=high, duration=60min`)
- Map event → runbook and event → target scope
- Produce human-grade change summary
- Enforce checklists (precheck must pass before apply)

**What it must NOT do:**

- Invent arbitrary config commands
- Reach devices directly
- Hold broad credentials
- Bypass template gates

### Extended Architecture (with OpenProse/EDA/Daytona)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Framework 16 (Podium) - Tailscale IP: 100.x.x.x                           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Podman Pod: callansible                      │   │
│  │                                                                      │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │asterisk │ │ voice-  │ │  vllm   │ │   aap   │ │   eda   │       │   │
│  │  │         │ │ bridge  │ │         │ │         │ │         │       │   │
│  │  │Port 5060│ │Port 8080│ │Port 8000│ │Port 443 │ │Port 5000│       │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘       │   │
│  │       │           │           │           │           │             │   │
│  │       └───────────┴───────────┴───────────┴───────────┘             │   │
│  │                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                   │
│  │ faster-whisper│  │   piper-tts   │  │    daytona    │                   │
│  │ (STT - CPU)   │  │  (TTS - CPU)  │  │   (sandbox)   │                   │
│  └───────────────┘  └───────────────┘  └───────────────┘                   │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                         OpenProse Session                              │ │
│  │  (Claude Code plugin - orchestrates incident workflow)                 │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  Browser: AAP Dashboard + EDA Event Log (visible to audience)              │
└─────────────────────────────────────────────────────────────────────────────┘
        ▲                              ▲
        │ SIP over Tailscale           │ Webhook (weather alert)
        │                              │
┌───────┴───────┐              ┌───────┴───────┐
│ Speaker Phone │              │  Event Source │
│               │              │ - Weather API │
│               │              │ - Manual JSON │
└───────────────┘              └───────────────┘
```

### Container Definitions (Additional)

#### 5. eda (Event-Driven Ansible)

**Purpose:** Event router with deterministic policy gates

**Image:** `quay.io/ansible/ansible-rulebook:latest`

**Ports:** 5000:5000

**Volumes:**

- `./eda/rulebooks:/rulebooks:ro`
- `./eda/inventory:/inventory:ro`

**Environment:**

```
AAP_URL=https://localhost
AAP_TOKEN=${AAP_TOKEN}
```

**Command:**

```bash
ansible-rulebook -i /inventory/hosts.yml \
  --rulebook /rulebooks/storm_mode.yml \
  --websocket-address 0.0.0.0:5000
```

#### 6. daytona (Sandbox Runtime)

**Purpose:** Isolated compute for validation steps with egress controls

**Integration:** Via Daytona MCP server for Claude Code / OpenProse

**Network Policy:**

```yaml
# Strict isolation - block all outbound except controller
networkBlockAll: true
networkAllowList:
  - "100.x.x.x/32"  # Tailscale IP of AAP controller
```

**Capabilities:**

- File system isolation
- Process sandboxing
- Explicit egress allowlisting (up to 5 CIDR blocks)
- Programmatic sandbox creation via MCP

-----

## EDA Rulebook Configuration

### eda/rulebooks/storm_mode.yml

```yaml
---
- name: Storm Mode Network Response
  hosts: all
  sources:
    - ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000

  rules:
    - name: Trigger storm mode on severe weather
      condition: >
        event.alert_type == "weather" and
        event.severity >= 3 and
        event.action == "activate"
      action:
        run_job_template:
          name: storm_mode_precheck
          organization: Default
          job_args:
            extra_vars:
              region: "{{ event.region }}"
              severity: "{{ event.severity }}"
              duration_minutes: "{{ event.duration | default(60) }}"
              request_id: "{{ event.request_id }}"

    - name: Apply storm mode after precheck approval
      condition: >
        event.alert_type == "storm_mode" and
        event.action == "apply" and
        event.precheck_passed == true
      action:
        run_job_template:
          name: storm_mode_apply
          organization: Default
          job_args:
            extra_vars:
              region: "{{ event.region }}"
              request_id: "{{ event.request_id }}"

    - name: Rollback storm mode
      condition: >
        event.alert_type == "weather" and
        event.action == "clear"
      action:
        run_job_template:
          name: storm_mode_rollback
          organization: Default
          job_args:
            extra_vars:
              region: "{{ event.region }}"
              request_id: "{{ event.request_id }}"

    - name: Verify storm mode status
      condition: >
        event.alert_type == "storm_mode" and
        event.action == "verify"
      action:
        run_job_template:
          name: storm_mode_verify
          organization: Default
          job_args:
            extra_vars:
              region: "{{ event.region }}"
              request_id: "{{ event.request_id }}"
```

### Sample Weather Alert Event

```json
{
  "alert_type": "weather",
  "severity": 4,
  "action": "activate",
  "region": "us-east-1",
  "duration": 120,
  "request_id": "storm-2025-01-29-001",
  "source": "national_weather_service",
  "headline": "Severe Thunderstorm Warning",
  "description": "Damaging winds and large hail expected",
  "effective": "2025-01-29T14:00:00Z",
  "expires": "2025-01-29T16:00:00Z"
}
```

-----

## OpenProse Workflow Definition

### openprose/storm_mode_incident.prose

```prose
# Storm Mode Incident Response Workflow

## Session Context
This workflow handles automated network hardening in response to severe weather alerts.
The LLM advises; EDA/AAP enforces. All analysis runs in Daytona sandbox with egress controls.

## Input Schema
- alert_type: string (must be "weather")
- severity: integer (1-5, where 5 is most severe)
- region: string (affected geographic region)
- duration: integer (expected duration in minutes)
- request_id: string (unique identifier for audit trail)

## Workflow Steps

### Step 1: Ingest Event
Accept incoming weather alert payload.
Validate against input schema.
Reject malformed events with clear error message.

### Step 2: Assess Severity
Extract structured fields:
- severity level
- affected region
- expected duration
- confidence score

Map severity to response level:
- severity >= 4: immediate storm mode
- severity == 3: storm mode with extended precheck
- severity < 3: monitor only, no action

### Step 3: Compute Scope
Map region to infrastructure targets:
- us-east-1 → lab_edge_east
- us-west-2 → lab_edge_west
- default → lab_edge_primary

### Step 4: Validate in Sandbox (Daytona)
Run validation steps with strict egress:
- Confirm AAP controller reachable
- Confirm job templates exist
- Render expected configuration diff
- Validate TTL/timebox values
- Run schema validation tests

Sandbox policy: networkAllowList limited to controller IP only.

### Step 5: Generate Decision Envelope
Produce structured recommendation:
```json
{
  "recommended_action": "storm_mode_apply",
  "scope": {"limit": "site=lab_edge_east"},
  "ttl_minutes": 60,
  "rollback_plan": "storm_mode_rollback",
  "change_summary": "Apply emergency ACLs to protect critical services",
  "request_id": "<from input>"
}
```

### Step 6: Request Approval
Present 6-10 line change plan to operator.
Require explicit approval: "approve storm mode for [duration] minutes"
Log approval with timestamp and approver identity.

### Step 7: Trigger Execution
Submit decision envelope to EDA as normalized event.
EDA rulebook gates and triggers run_job_template to AAP.
Do NOT call AAP API directly from this workflow.

### Step 8: Monitor Execution
Watch job output via AAP API.
Report status updates to operator.
Expected completion: < 30 seconds for ACL changes.

### Step 9: Verify
Trigger storm_mode_verify template.
Confirm critical services accessible.
Confirm nonessential services blocked.
If verification fails: auto-trigger rollback.

### Step 10: Schedule Rollback
Set timer for TTL expiration.
When "all clear" event received OR TTL expires:
- Trigger storm_mode_rollback
- Verify restoration
- Close incident session

## Error Handling
- Validation failure: reject with specific error
- AAP unreachable: retry 3x with backoff, then alert operator
- Job failure: report verbally, offer manual intervention
- Verification failure: auto-rollback, alert operator

## Audit Trail
Log all decisions, approvals, and actions with:
- Timestamp
- Request ID
- Actor (operator or system)
- Action taken
- Result
```

-----

## Storm Mode Playbooks

### playbooks/storm_mode_precheck.yml

```yaml
---
- name: Storm Mode Precheck
  hosts: localhost
  gather_facts: false
  vars:
    region: "{{ region | default('us-east-1') }}"
    severity: "{{ severity | default(3) }}"
    request_id: "{{ request_id | default('unknown') }}"

  tasks:
    - name: Log precheck start
      debug:
        msg: "Starting storm mode precheck for {{ region }} (severity: {{ severity }}, request: {{ request_id }})"

    - name: Verify target host reachability
      wait_for:
        host: "{{ hostvars['lab_edge']['ansible_host'] | default('127.0.0.1') }}"
        port: 22
        timeout: 10
      register: reachability
      ignore_errors: true

    - name: Check current firewall state
      command: "iptables -L -n"
      register: current_rules
      changed_when: false
      delegate_to: localhost

    - name: Validate rollback script exists
      stat:
        path: /opt/storm_mode/rollback.sh
      register: rollback_script
      delegate_to: localhost

    - name: Generate expected diff
      debug:
        msg: |
          Expected changes for {{ region }}:
          + ACCEPT tcp dport 22 (SSH)
          + ACCEPT tcp dport 443 (HTTPS)
          + ACCEPT udp dport 51820 (WireGuard)
          + ACCEPT tcp dport 5060 (SIP)
          - DROP all nonessential traffic

    - name: Precheck summary
      set_fact:
        precheck_result:
          passed: "{{ reachability is success }}"
          region: "{{ region }}"
          request_id: "{{ request_id }}"
          rollback_available: "{{ rollback_script.stat.exists | default(false) }}"

    - name: Report precheck status
      debug:
        msg: "Precheck {{ 'PASSED' if precheck_result.passed else 'FAILED' }} for request {{ request_id }}"
```

### playbooks/storm_mode_apply.yml

```yaml
---
- name: Storm Mode Apply
  hosts: localhost
  gather_facts: false
  vars:
    region: "{{ region | default('us-east-1') }}"
    request_id: "{{ request_id | default('unknown') }}"

  tasks:
    - name: Log storm mode activation
      debug:
        msg: "ACTIVATING STORM MODE for {{ region }} (request: {{ request_id }})"

    - name: Backup current firewall rules
      shell: "iptables-save > /opt/storm_mode/backup_{{ request_id }}.rules"
      delegate_to: localhost

    - name: Apply emergency ACL - Allow SSH
      iptables:
        chain: INPUT
        protocol: tcp
        destination_port: 22
        jump: ACCEPT
        comment: "Storm mode - SSH"
      delegate_to: localhost

    - name: Apply emergency ACL - Allow HTTPS
      iptables:
        chain: INPUT
        protocol: tcp
        destination_port: 443
        jump: ACCEPT
        comment: "Storm mode - HTTPS"
      delegate_to: localhost

    - name: Apply emergency ACL - Allow VPN
      iptables:
        chain: INPUT
        protocol: udp
        destination_port: 51820
        jump: ACCEPT
        comment: "Storm mode - WireGuard VPN"
      delegate_to: localhost

    - name: Apply emergency ACL - Allow SIP
      iptables:
        chain: INPUT
        protocol: tcp
        destination_port: 5060
        jump: ACCEPT
        comment: "Storm mode - SIP signaling"
      delegate_to: localhost

    - name: Apply emergency ACL - Allow monitoring
      iptables:
        chain: INPUT
        protocol: tcp
        destination_port: 9090
        jump: ACCEPT
        comment: "Storm mode - Prometheus"
      delegate_to: localhost

    - name: Block nonessential traffic (demo port)
      iptables:
        chain: INPUT
        protocol: tcp
        destination_port: 8080
        jump: DROP
        comment: "Storm mode - Block nonessential"
      delegate_to: localhost

    - name: Storm mode activated
      debug:
        msg: "Storm mode ACTIVE for {{ region }}. Critical services protected. Request: {{ request_id }}"
```

### playbooks/storm_mode_verify.yml

```yaml
---
- name: Storm Mode Verify
  hosts: localhost
  gather_facts: false
  vars:
    region: "{{ region | default('us-east-1') }}"
    request_id: "{{ request_id | default('unknown') }}"

  tasks:
    - name: Verify SSH accessible
      wait_for:
        host: 127.0.0.1
        port: 22
        timeout: 5
      register: ssh_check
      ignore_errors: true

    - name: Verify HTTPS accessible
      uri:
        url: "https://localhost:443/api/v2/ping/"
        validate_certs: false
        timeout: 5
      register: https_check
      ignore_errors: true

    - name: Verify nonessential blocked
      wait_for:
        host: 127.0.0.1
        port: 8080
        timeout: 3
      register: blocked_check
      ignore_errors: true

    - name: Compile verification results
      set_fact:
        verify_result:
          ssh_accessible: "{{ ssh_check is success }}"
          https_accessible: "{{ https_check is success }}"
          nonessential_blocked: "{{ blocked_check is failed }}"
          overall_pass: "{{ ssh_check is success and https_check is success and blocked_check is failed }}"

    - name: Report verification status
      debug:
        msg: |
          Storm Mode Verification for {{ region }} ({{ request_id }}):
          - SSH: {{ 'OK' if verify_result.ssh_accessible else 'FAIL' }}
          - HTTPS: {{ 'OK' if verify_result.https_accessible else 'FAIL' }}
          - Nonessential blocked: {{ 'OK' if verify_result.nonessential_blocked else 'FAIL' }}
          - Overall: {{ 'PASS' if verify_result.overall_pass else 'FAIL' }}

    - name: Fail if verification unsuccessful
      fail:
        msg: "Storm mode verification FAILED - triggering rollback"
      when: not verify_result.overall_pass
```

### playbooks/storm_mode_rollback.yml

```yaml
---
- name: Storm Mode Rollback
  hosts: localhost
  gather_facts: false
  vars:
    region: "{{ region | default('us-east-1') }}"
    request_id: "{{ request_id | default('unknown') }}"

  tasks:
    - name: Log rollback start
      debug:
        msg: "ROLLING BACK storm mode for {{ region }} (request: {{ request_id }})"

    - name: Check for backup rules
      stat:
        path: "/opt/storm_mode/backup_{{ request_id }}.rules"
      register: backup_file

    - name: Restore from backup if available
      shell: "iptables-restore < /opt/storm_mode/backup_{{ request_id }}.rules"
      when: backup_file.stat.exists
      delegate_to: localhost

    - name: Flush storm mode rules if no backup
      shell: |
        iptables -D INPUT -p tcp --dport 22 -j ACCEPT -m comment --comment "Storm mode - SSH" 2>/dev/null || true
        iptables -D INPUT -p tcp --dport 443 -j ACCEPT -m comment --comment "Storm mode - HTTPS" 2>/dev/null || true
        iptables -D INPUT -p udp --dport 51820 -j ACCEPT -m comment --comment "Storm mode - WireGuard VPN" 2>/dev/null || true
        iptables -D INPUT -p tcp --dport 5060 -j ACCEPT -m comment --comment "Storm mode - SIP signaling" 2>/dev/null || true
        iptables -D INPUT -p tcp --dport 9090 -j ACCEPT -m comment --comment "Storm mode - Prometheus" 2>/dev/null || true
        iptables -D INPUT -p tcp --dport 8080 -j DROP -m comment --comment "Storm mode - Block nonessential" 2>/dev/null || true
      when: not backup_file.stat.exists
      delegate_to: localhost

    - name: Rollback complete
      debug:
        msg: "Storm mode DEACTIVATED for {{ region }}. Normal operations restored. Request: {{ request_id }}"
```

-----

## OpenProse Incident State Machine

```
┌─────────────┐
│   IDLE      │
└──────┬──────┘
       │ Weather alert received
       ▼
┌─────────────┐
│   INGEST    │ ──► Validate event schema
└──────┬──────┘
       │ Valid event
       ▼
┌─────────────┐
│   ASSESS    │ ──► Extract severity, region, duration
└──────┬──────┘
       │ Severity >= threshold
       ▼
┌─────────────┐
│   SCOPE     │ ──► Map region → targets
└──────┬──────┘
       │ Targets identified
       ▼
┌─────────────────┐
│ VALIDATE        │ ──► Daytona sandbox
│ (sandboxed)     │     - Controller reachable?
└──────┬──────────┘     - Templates exist?
       │ Validation pass  - Render diff
       ▼
┌─────────────┐
│  PROPOSE    │ ──► Generate decision envelope
└──────┬──────┘
       │ Plan ready
       ▼
┌─────────────┐
│  APPROVE    │ ◄── Human approval required
└──────┬──────┘
       │ "Approved"
       ▼
┌─────────────┐
│  EXECUTE    │ ──► Submit to EDA → AAP
└──────┬──────┘
       │ Job launched
       ▼
┌─────────────┐
│  MONITOR    │ ──► Watch job status
└──────┬──────┘
       │ Job complete
       ▼
┌─────────────┐     ┌─────────────┐
│   VERIFY    │────►│  ROLLBACK   │ (if verification fails)
└──────┬──────┘     └─────────────┘
       │ Verification pass
       ▼
┌─────────────┐
│   ACTIVE    │ ◄── Storm mode engaged
└──────┬──────┘
       │ "All clear" OR TTL expires
       ▼
┌─────────────┐
│  ROLLBACK   │ ──► Restore normal operations
└──────┬──────┘
       │ Rollback complete
       ▼
┌─────────────┐
│   CLOSED    │ ──► Log audit trail
└─────────────┘
```

-----

## Updated File Structure

```
CallAnsible/
├── SPEC.md                          # This file
├── README.md                        # User-facing documentation
├── docker-compose.yml               # Podman compose orchestration
├── .env.example                     # Environment template
├── .gitignore
│
├── asterisk/
│   ├── Dockerfile
│   ├── pjsip.conf
│   ├── extensions.conf
│   ├── ari.conf
│   ├── http.conf
│   └── rtp.conf
│
├── voice-bridge/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── config.py
│   ├── ari_client.py
│   ├── stt.py
│   ├── tts.py
│   ├── llm.py
│   ├── aap.py
│   ├── call_handler.py
│   └── prompts.py
│
├── eda/                              # NEW: Event-Driven Ansible
│   ├── rulebooks/
│   │   └── storm_mode.yml           # Storm mode rulebook
│   ├── inventory/
│   │   └── hosts.yml                # EDA inventory
│   └── test_events/
│       └── weather_alert.json       # Sample test event
│
├── openprose/                        # NEW: OpenProse workflows
│   ├── storm_mode_incident.prose    # Incident workflow definition
│   └── schemas/
│       └── weather_alert.schema.json # Input validation schema
│
├── daytona/                          # NEW: Daytona sandbox config
│   ├── policy.yml                   # Network egress policy
│   └── mcp_config.json              # MCP server configuration
│
├── models/
│   └── piper/
│       └── .gitkeep
│
├── playbooks/
│   ├── self_healing_workflow.yml
│   ├── health_check.yml
│   ├── scale_deployment.yml
│   ├── storm_mode_precheck.yml      # NEW
│   ├── storm_mode_apply.yml         # NEW
│   ├── storm_mode_verify.yml        # NEW
│   └── storm_mode_rollback.yml      # NEW
│
├── audio/
│   └── .gitkeep
│
└── scripts/
    ├── setup.sh
    ├── download-models.sh
    ├── test-call.sh
    └── inject-weather-alert.sh      # NEW: Test event injection
```

-----

## Daytona Sandbox Configuration

### daytona/policy.yml

```yaml
# Daytona sandbox network policy for validation steps
sandbox:
  name: callansible-validator

  # Block all outbound by default
  networkBlockAll: true

  # Allowlist only the AAP controller
  networkAllowList:
    - "127.0.0.1/32"      # localhost for testing
    - "100.x.x.x/32"      # Tailscale IP of AAP controller

  # Resource limits
  resources:
    cpu: "1"
    memory: "512Mi"

  # Timeout for validation operations
  timeout: 60s
```

### daytona/mcp_config.json

```json
{
  "name": "daytona-callansible",
  "version": "1.0.0",
  "description": "Daytona MCP server for CallAnsible validation",
  "capabilities": {
    "sandbox": {
      "create": true,
      "destroy": true,
      "execute": true,
      "file_read": true,
      "file_write": true
    }
  },
  "default_policy": "daytona/policy.yml"
}
```

-----

## Updated Environment Variables

### .env.example (additions)

```bash
# ... existing variables ...

# EDA
EDA_WEBHOOK_PORT=5000

# AAP API Token (for EDA integration)
AAP_TOKEN=changeme_aap_token

# Daytona
DAYTONA_ENABLED=true
DAYTONA_NETWORK_POLICY=strict

# OpenProse
OPENPROSE_APPROVAL_TIMEOUT=300
OPENPROSE_AUTO_ROLLBACK=true
```

-----

## Security Posture (OpenProse Integration)

### Defensible Claims

1. **Approved actions only:** The agent can only launch pre-built job templates. No arbitrary playbooks.

2. **Token least privilege:** EDA uses a token scoped to launching specific templates only.

3. **Sandboxed analysis + egress controls:** Daytona blocks all outbound or allowlists specific networks. Prevents data exfiltration, reduces attack surface.

4. **Human-in-loop for apply:** The "apply" step requires explicit approval. Precheck runs automatically.

5. **Rollback is first-class:** Rollback template exists and is easy to trigger manually or automatically on verification failure.

### The "Why LLM?" Answer

When someone asks: "Why do we need the LLM? Couldn't EDA just do this?"

**Answer:** EDA is excellent at if/then on structured events and triggering actions. The LLM is valuable when:
- Input is messy and contextual (unstructured alerts, incident notes)
- Correlation is needed ("what this means" → runbooks)
- Scope and parameters must be selected dynamically
- Operator-grade summaries and checklists are needed

AAP Job Templates remain the controlled action surface.

-----

## Testing Checklist (Storm Mode)

### Pre-demo additions

- [ ] EDA container healthy and receiving webhooks
- [ ] Daytona sandbox creation succeeds
- [ ] Weather alert injection triggers precheck
- [ ] Precheck passes and requests approval
- [ ] Approval triggers apply
- [ ] Apply completes and verification passes
- [ ] "All clear" event triggers rollback
- [ ] Rollback restores normal state

### Sample test commands

```bash
# Inject weather alert
curl -X POST http://localhost:5000/endpoint \
  -H "Content-Type: application/json" \
  -d @eda/test_events/weather_alert.json

# Check EDA logs
podman-compose logs -f eda

# Verify storm mode active
iptables -L -n | grep "Storm mode"

# Inject all-clear
curl -X POST http://localhost:5000/endpoint \
  -H "Content-Type: application/json" \
  -d '{"alert_type": "weather", "action": "clear", "region": "us-east-1", "request_id": "storm-2025-01-29-001"}'
```

-----

## Notes

- All containers use `--network host` for simplicity with RTP
- GPU is dedicated to vLLM; STT and TTS run on CPU (Ryzen 9 handles this easily)
- Audio files are transient, stored in ./audio, cleared on restart
- AAP dashboard should be visible in browser during demo - this is the visual feedback for audience
- OpenProse runs as Claude Code plugin (beta) - appropriate for demos and POCs
- EDA provides deterministic policy gates between LLM recommendations and execution
- Daytona sandboxes all validation with explicit egress controls
- The core value proposition: "AI accelerates decision-making; the platform enforces safe, pre-approved execution"
