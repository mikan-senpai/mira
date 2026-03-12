Let me search for the latest on open-source voice agents and relevant Indian STT/TTS options before architecting this.Now I have everything I need. Let me build the full system architecture for you.Now let me show you the per-doctor isolation design — this is the most critical part of Mira as a product.Now let me explain the full blueprint in detail.


![System Architecture](docs/mira_system_architecture.svg)
---
![Doctor Illustration](docs/mira_doctor_isolation.svg)

---

## Mira — Complete System Blueprint

### The Core Stack (all open source / India-optimised)

| Layer | Tool | Why |
|---|---|---|
| Telephony | FusionPBX (yours) | On-prem, already there |
| Voice pipeline | **Pipecat** (open source Python) | Orchestrates audio, AI services, and conversation pipelines; natively supports Sarvam |
| STT | **Sarvam Saarika v2** | Supports 22 Indian languages with real-time transcription, code-mixing (Hinglish etc.), and auto language detection |
| LLM | **Ollama** (local) | Open source, runs on your agent server, no API cost |
| TTS | **Sarvam Bulbul v3** | Authentic Indian accents, 11 Indian languages, PCM/μ-law output formats, streaming WebSocket for real-time agents |
| Notification | **WhatsApp Cloud API** (Meta) | Free tier available, official and reliable |

---


### How the Call Flow Works (Step by Step)

**Step 1 — Call arrives at FusionPBX**
The doctor's phone number (DID) is registered in FusionPBX. When a call comes in, the dialplan first tries to ring the doctor's extension. If it goes unanswered (you configure this as, say, 15 seconds), the dialplan fires the "no answer" route.

**Step 2 — FusionPBX hands off to Mira via WebSocket**
Using FreeSWITCH's `mod_audio_stream`, the call audio is streamed over a WebSocket connection directly to your Mira agent server. This is the bridge between your on-prem PBX and the AI layer. No Twilio needed, no cloud middleman.

In FusionPBX's dialplan, the no-answer action looks like:
```xml
<action application="audio_stream" data="wss://your-agent-server:8765/ws?doctor_id=DR001"/>
```
The `doctor_id` is passed as a query parameter — this is how Mira knows which doctor's context to load.

**Step 3 — Mira's Pipecat pipeline activates**
Pipecat integrates Sarvam's Saarika for STT and Bulbul for TTS, and you can deploy a voice agent in under 10 minutes using this combination. The pipeline is: incoming audio → Saarika (speech to text) → your LLM (Ollama with a doctor-specific system prompt) → Bulbul (text to speech) → audio back to patient.

**Step 4 — Conversation with strict isolation**
The LLM's system prompt is built dynamically from the doctor's profile in your database:
```
You are Mira, the receptionist for Dr. [NAME], a [SPECIALISATION] 
based in [LOCATION]. You ONLY know about Dr. [NAME]. 
Never mention or acknowledge any other doctor. 
If asked about other doctors, politely say you can only help 
with Dr. [NAME]'s appointments.
Working hours: [HOURS]. Calendar ID: [CALENDAR_ID].
```
This is how you prevent cross-contamination. Each call session gets a fresh system prompt scoped to exactly one doctor. There is no shared memory between doctors.

**Step 5 — Appointment booking**
Mira calls a tool function (Pipecat supports LLM tool use) that checks the doctor's Google Calendar (or CalDAV for open source). The conversation flow is:
1. Greet → "Namaste, you've reached Dr. X's clinic. How can I help you?"
2. Understand → figure out if they want an appointment, a query, etc.
3. Suggest → "Dr. X is available Thursday at 3pm or Friday at 11am. Which works for you?"
4. Confirm → patient agrees → Mira books the slot in the calendar.

**Step 6 — WhatsApp notification to doctor**
On confirmed booking, Mira calls the WhatsApp Cloud API and sends the doctor a message like:
> "📅 New appointment: [Patient Name] on [Date] at [Time]. Reason: [brief note from call]."

---

### Multi-Tenant Isolation (Jailbreak Prevention)

Three layers of protection:

1. **Dialplan level** — each DID maps to exactly one `doctor_id`. A caller reaching Doctor 1's number physically cannot reach Doctor 2's context because the PBX routes them to separate WebSocket sessions.

2. **System prompt level** — the LLM is instructed to only know about one doctor. Any question about other doctors is handled by a refusal response.

3. **Database level** — the `doctor_id` is the primary key for every data lookup (calendar, profile, hours). No query in the code ever pulls "all doctors" during a live call.

---

### What You Need to Build / Set Up

**Server infrastructure:**
- 1 agent server (can be the same machine as FusionPBX or separate) running Python 3.12+, Pipecat, Ollama
- A small PostgreSQL or SQLite database to store doctor profiles
- Ollama running an open source model (Llama 3 8B works well for appointment booking dialogue)

**Accounts/APIs:**
- Sarvam AI account (₹1,000 free credits to start, ~₹1/min at scale)
- Meta WhatsApp Business API (free for the first ~1,000 conversations/month)
- Google Calendar API (free) or Nextcloud CalDAV (fully open source)

**FusionPBX configuration:**
- Per-DID dialplan with `mod_audio_stream` for no-answer routing
- A "fail-safe" in the dialplan: if Mira's server is down, fall back to voicemail

---

### What to Build First (Suggested Order)

1. Stand up Pipecat + Sarvam STT/TTS, test a basic "Hello, how can I help" conversation locally
2. Add Ollama with a doctor system prompt, test the full voice loop
3. Connect FusionPBX WebSocket bridge (this is the trickiest part — test with a SIP softphone first)
4. Build the calendar check/book tool function
5. Add WhatsApp notification
6. Add the doctor profile DB and multi-tenant routing

The hardest technical piece is step 3 — bridging FusionPBX's G.711 audio to Pipecat. The codec is μ-law 8kHz; Sarvam's STT accepts this directly, so you won't need to transcode. Want a deep-dive into the WebSocket bridge code specifically?