# Mira: Layer-by-Layer Implementation Plan

To build Mira's multi-tenant architecture efficiently, we need to construct it layer by layer, starting from the foundation (infrastructure) up to the specific integrations. We will ensure each layer is functional before moving to the next.

## Phase 1: The Core Compute Infrastructure
Before we can run models, we need the hardware and the environment to host them.

### Step 1.1: Provision the Agent Server
1.  **Hardware Specification:** Secure a server with an NVIDIA GPU (e.g., RTX 4090, A5000, or rent a cloud instance like RunPod/AWS g5.xlarge).
2.  **OS Preparation:** Install Ubuntu Server 22.04 LTS.
3.  **Drivers & Runtimes:** Install NVIDIA drivers and the NVIDIA Container Toolkit to allow Docker containers to access the GPU. 
4.  **Network Setup:** Ensure the server has a static, public IP Address, or a secure tunnel (like Cloudflare Tunnel or ngrok) so the PBX can communicate with it over SIP.

### Step 1.2: Host the Local AI Models
We will run the models inside Docker containers for easy management and scaling.
1.  **LLM Deployment (vLLM):** Pull the vLLM docker image and deploy `Qwen/Qwen2.5-7B-Instruct`. Verify it’s running by sending a test prompt to its OpenAI-compatible API endpoint.
2.  **STT Deployment (Faster-Whisper):** Pull a pre-built Faster-Whisper API container (or write a small FastAPI wrapper around it). Verify it can accept audio files and return text quickly.
3.  **TTS Deployment (XTTSv2):** Pull the Coqui XTTS docker image. Verify it can accept a text string and stream generated audio back.

## Phase 2: The Multi-Tenant Database & API Fencing
We need to create the system that stores doctor profiles and prevents cross-talk.

### Step 2.1: Database Setup
1.  **Install PostgreSQL:** Set up a local or hosted PostgreSQL instance.
2.  **Define Schema:** Create the `doctor_profiles` table:
    *   `doctor_id` (Primary Key)
    *   `inbound_phone_number` (The extension on the PBX)
    *   `name`
    *   `specialization`
    *   `location`
    *   `custom_rules` (e.g., "Always ask for age first")

### Step 2.2: Build the Routing API
1.  **Create a Python API (FastAPI):** Write a service that connects to PostgreSQL.
2.  **Implement `get_profile(phone_number)`:** When called, it looks up the inbound DID (phone number) and returns the JSON profile for that specific doctor.
3.  **Implement API Fencing:** Create wrapper endpoints for the external tools (Calendar/WhatsApp) that *require* the `doctor_id` as an argument from the orchestrator, preventing the LLM from trying to access other doctors' data.

## Phase 3: The Orchestrator (Vocode)
This is the most critical layer. It ties the telephony, the models, and the database together.

### Step 3.1: Set up Vocode Telephony
1.  **Install Vocode:** Initialize a new Python project and install the `vocode` package.
2.  **Configure SIP Inbound (FusionPBX/FreeSWITCH):** Set up the `TelephonyServer` in Vocode. Since you are using FreeSWITCH, you will create a new SIP Profile or Gateway in FusionPBX that points directly to the Vocode server's IP and Port, routing specific extensions (DID) to it.
3.  **Configure Model Endpoints:** Point Vocode's configuration to the local IPs/ports we set up in Phase 1 for vLLM, Faster-Whisper, and XTTS.

### Step 3.2: Implement Dynamic Prompting
1.  **Extract the Caller ID/DID:** Modify the Vocode call handler to capture the number the patient dialed.
2.  **Call the Routing API:** Before instantiating the Vocode `StreamingConversation`, call the API developed in Step 2.2 to fetch the correct doctor's profile.
3.  **Inject System Prompt:** Use the fetched data to inject the highly specific, isolated system prompt into the LLM configuration for this active call. 

## Phase 4: Integrations (Calendar & WhatsApp)
Now we build the tools the LLM can trigger.

### Step 4.1: Calendar Booking Service
1.  **Develop the Tool:** Write a Python function `check_availability(doctor_id, date)` that connects to a generic calendar backend (e.g., Google Calendar API via Service Accounts). Use the `doctor_id` to select the right authenticated token/calendar ID.
2.  **Develop Booking Function:** Write `book_appointment(doctor_id, time, patient_name)`.
3.  **Register Tools:** Add these functions to the Vocode/LLM integration layer so Qwen knows it can use them mid-conversation.

### Step 4.2: WhatsApp Notifications
1.  **Set up Baileys / Meta API:** Choose a WhatsApp provider.
2.  **Develop Notification Function:** Write a simple webhook or script that sends a message: *"New Appointment at {time} for {name}"* to the doctor's registered phone number (fetched from the database).
3.  **Trigger on Booking:** Call this function immediately *after* the `book_appointment` function successfully exits.

## Phase 5: Testing & Go-Live Strategy

1.  **Latency Benchmarking:** Test the Time-To-First-Byte. If it's over 1 second, profile the pipeline to see if STT, LLM generation, or TTS is the bottleneck.
2.  **"Red Teaming" (Jailbreak Testing):** Deliberately call Doctor 1's number and try to trick Mira into revealing information about Doctor 2 or bypassing the calendar tools. Ensure the prompt logic holds firm.
3.  **PBX Pilot:** Route real (but controlled test) calls from the PBX into the Agent Server and monitor audio quality (RTP packet loss) over the SIP trunk.
4.  **Doctor 1 Onboarding:** Launch with one trusted doctor client before scaling up.
