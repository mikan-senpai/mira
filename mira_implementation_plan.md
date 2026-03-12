# Mira: Implementation Plan (LOCKED IN ✅)

> **All components are 100% open source and self-hosted. Zero API costs. Zero vendor lock-in.**

## Final Technology Stack

| Layer | Tool | License | Runs On |
|---|---|---|---|
| **PBX** | FusionPBX / FreeSWITCH | Open Source | On-Prem Server |
| **Audio Bridge** | `mod_audio_stream` | Open Source | FreeSWITCH Module |
| **Orchestrator** | **Pipecat** (Python) | Open Source | Agent Server |
| **STT (Ears)** | **Faster-Whisper** (`distil-large-v3`) | MIT | Agent Server (GPU) |
| **LLM (Brain)** | **Sarvam 30B** via Ollama | Apache 2.0 | Agent Server (GPU) |
| **TTS (Voice)** | **XTTSv2** (Coqui) / **ChatTTS** | Open Source | Agent Server (GPU) |
| **Calendar** | Google Calendar API / CalDAV | Free | Cloud / Self-Hosted |
| **WhatsApp** | **Baileys** (Node.js) | Open Source | Agent Server |

### Hardware Requirements
- **GPU:** NVIDIA RTX 4090 (24GB) or A5000 — runs all 3 models simultaneously
- **CPU:** 8+ cores recommended
- **RAM:** 32GB minimum
- **OS:** Ubuntu Server 22.04 LTS (Dockerized)
- **Network:** Static IP or tunnel (for PBX → Agent Server communication)

---

## Phase 1: Core Compute Infrastructure

### Step 1.1: Prepare the Agent Server
```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Install NVIDIA Container Toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

### Step 1.2: Deploy Sarvam 30B via Ollama
```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull Sarvam 30B (Apache 2.0, Indian languages)
ollama pull sarvam-30b

# Verify — should respond fluently in Hindi/English
ollama run sarvam-30b "Namaste, aap kaise hain? Mujhe ek appointment book karni hai."
```

### Step 1.3: Deploy Faster-Whisper (STT)
```bash
# Using Docker
docker run -d --gpus all -p 8001:8000 \
  --name whisper-server \
  fedirz/faster-whisper-server:latest \
  --model distil-whisper/distil-large-v3
```

### Step 1.4: Deploy XTTSv2 (TTS)
```bash
# Using Docker
docker run -d --gpus all -p 8002:8000 \
  --name xtts-server \
  ghcr.io/coqui-ai/xtts-streaming-server:latest
```

### Step 1.5: Install Pipecat (Orchestrator)
```bash
python3 -m venv mira-env
source mira-env/bin/activate
pip install "pipecat-ai[websocket,silero]"
```

---

## Phase 2: Multi-Tenant Database & API Fencing

### Step 2.1: PostgreSQL Setup
```bash
sudo apt install postgresql postgresql-contrib
sudo -u postgres createdb mira
```

### Step 2.2: Doctor Profiles Schema
```sql
CREATE TABLE doctor_profiles (
    doctor_id       TEXT PRIMARY KEY,
    did_number      TEXT UNIQUE NOT NULL,
    name            TEXT NOT NULL,
    specialization  TEXT,
    location        TEXT,
    working_hours   JSONB,
    calendar_id     TEXT,
    whatsapp_number TEXT,
    custom_rules    TEXT
);

-- Example: Onboard Doctor 1
INSERT INTO doctor_profiles VALUES (
    'DR001', '1001', 'Dr. Sharma', 'Cardiologist',
    '101 MG Road, Bangalore',
    '{"mon":"09:00-17:00","tue":"09:00-17:00","wed":"09:00-13:00"}',
    'gcal_dr_sharma@group.calendar.google.com',
    '+919876543210',
    'Always ask for patient age and existing conditions before booking.'
);
```

### Step 2.3: FastAPI Routing Service
A Python microservice that:
1. `GET /profile?doctor_id=DR001` → Returns the doctor JSON.
2. `POST /book` → Books appointment, **forces** `doctor_id` from session (LLM can't override).
3. `POST /notify` → Sends WhatsApp message to the doctor's number.

---

## Phase 3: Telephony Bridge (FusionPBX → Pipecat)

### Step 3.1: Install `mod_audio_stream`
```bash
sudo apt install freeswitch-mod-audio-stream
# Enable in /etc/freeswitch/autoload_configs/modules.conf.xml
# <load module="mod_audio_stream"/>
sudo systemctl restart freeswitch
```

### Step 3.2: FusionPBX Dialplan (No-Answer → Mira)
```xml
<!-- Doctor's extension rings for 15 seconds -->
<action application="set" data="call_timeout=15"/>
<action application="bridge" data="user/${dialed_extension}@${domain_name}"/>

<!-- If no answer, stream audio to Mira Agent Server -->
<anti-action application="audio_stream"
  data="wss://AGENT_SERVER_IP:8765/ws?doctor_id=DR001"/>
```

### Step 3.3: Pipecat WebSocket Agent
The Python agent that:
1. Receives μ-law 8kHz audio from `mod_audio_stream`.
2. Extracts `doctor_id` from the WebSocket URL query parameter.
3. Fetches the doctor profile from the Routing API.
4. Dynamically builds the system prompt (isolated to this doctor only).
5. Runs the pipeline: **Audio → Faster-Whisper → Sarvam 30B → XTTSv2 → Audio back**.

---

## Phase 4: Integrations (Calendar + WhatsApp)

### Step 4.1: Calendar Functions (LLM Tools)
```python
# Registered as tool functions the LLM can call
def check_availability(date: str) -> list[str]:
    """Returns available time slots for the doctor on the given date."""
    # Uses doctor_id from the session — LLM cannot override
    ...

def book_appointment(date: str, time: str, patient_name: str) -> dict:
    """Books appointment, creates calendar event, triggers WhatsApp notification."""
    # Scoped to doctor_id from session
    ...
```

### Step 4.2: WhatsApp via Baileys
```javascript
// Node.js microservice using Baileys
const { makeWASocket } = require('@whiskeysockets/baileys');
// Sends: "📅 New appointment: [Name] on [Date] at [Time]"
```

---

## Phase 5: Testing & Go-Live

| Test | Method | Pass Criteria |
|---|---|---|
| **Voice Loop** | Microphone → STT → LLM → TTS → Speaker | Coherent response in < 2s |
| **SIP Softphone** | Zoiper → FusionPBX → Mira | Full booking conversation works |
| **Latency** | Measure TTFB across pipeline | < 1 second |
| **Isolation** | Call Dr.1's line, ask about Dr.2 | Mira refuses / says "I only manage Dr. Sharma's schedule" |
| **Jailbreak** | Prompt injection attempts | No cross-tenant data leaked |
| **Pilot** | Live with 1 real doctor | Successful appointment + WhatsApp notification |

---

## Build Order (Start Here ↓)
| # | What to Build | Depends On | Estimated Time |
|---|---|---|---|
| **1** | Install Ollama + Sarvam 30B | Server + GPU | 30 mins |
| **2** | Deploy Faster-Whisper + XTTSv2 Docker | Server + GPU | 30 mins |
| **3** | Write Pipecat voice loop (mic test) | Steps 1-2 | 2-3 hours |
| **4** | Setup PostgreSQL + doctor profiles | Server | 1 hour |
| **5** | Install `mod_audio_stream` + dialplan | FusionPBX access | 2-3 hours |
| **6** | Build Pipecat WebSocket server | Steps 3-5 | 3-4 hours |
| **7** | Calendar + WhatsApp integrations | Step 6 | 2-3 hours |
| **8** | Red team testing + pilot | Everything | 1-2 days |
