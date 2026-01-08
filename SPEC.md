# CallAnsible - Technical Specification

## Overview

Voice-controlled Ansible Automation Platform (AAP) via self-hosted Asterisk PBX, local LLM inference, and containerized infrastructure. Designed for live conference demo on January 29, 2025.

**Demo scenario:** Speaker calls their own infrastructure from stage, speaks natural language commands, audience watches AAP dashboard execute playbooks in real-time.

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

## Notes

- All containers use `--network host` for simplicity with RTP
- GPU is dedicated to vLLM; STT and TTS run on CPU (Ryzen 9 handles this easily)
- Audio files are transient, stored in ./audio, cleared on restart
- AAP dashboard should be visible in browser during demo - this is the visual feedback for audience
