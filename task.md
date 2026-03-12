# Mira — AI Receptionist Project Checklist

## Planning & Architecture ✅
- [x] Document High-Level Architecture
- [x] Define Open-Source Technology Stack (100% OSS locked in)
- [x] Diagram the System Flow
- [x] Design Multi-Tenant (Multi-Doctor) Isolation Strategy
- [x] Draft Initial Prompts and Function Call Definitions
- [x] Lock in Implementation Plan

## Implementation (Layer by Layer)
### Phase 1: Core Compute Infrastructure
- [ ] Provision Agent Server (GPU, Docker, NVIDIA Toolkit)
- [ ] Deploy Sarvam 30B via Ollama
- [ ] Deploy Faster-Whisper (STT) via Docker
- [ ] Deploy XTTSv2 (TTS) via Docker
- [ ] Install Pipecat with WebSocket + Silero VAD

### Phase 2: Multi-Tenant Database & API
- [ ] Setup PostgreSQL & `doctor_profiles` schema
- [ ] Build FastAPI Routing API (`get_profile`)
- [ ] Implement API Fencing (scope tools to `doctor_id`)

### Phase 3: Telephony Bridge (FusionPBX → Pipecat)
- [ ] Install `mod_audio_stream` on FreeSWITCH
- [ ] Configure FusionPBX Dialplan (no-answer → WebSocket)
- [ ] Build Pipecat WebSocket server with dynamic prompt injection

### Phase 4: Integrations (Tools)
- [ ] Calendar `check_availability` & `book_appointment`
- [ ] WhatsApp notification via Baileys
- [ ] Register tools with LLM/Pipecat

### Phase 5: Testing & Go-Live
- [ ] Local voice loop test (mic → STT → LLM → TTS → speaker)
- [ ] SIP softphone test via FusionPBX
- [ ] Red Team testing for cross-tenant data leaks
- [ ] Pilot with Doctor 1
