# Mira: Multi-Tenant Architecture Plan

## 1. The Core Concept: "Multi-Tenancy"
To sell Mira to multiple independent doctors, we need a **Multi-Tenant Architecture**. This means instead of running a separate, costly AI server for *each* doctor, ONE central Agent Server will handle all calls, but it will dynamically load the correct "Persona" and "Database" based on *who* the patient is calling.

## 2. Strict Isolation Strategy (No Cross-Talk)
To guarantee that a patient calling Dr. A never hears about Dr. B, we enforce strict isolation at three levels:

### Level 1: Telephony Routing (The Trigger)
When the PBX receives a call, it knows which phone number or extension was dialed.
*   **DID Routing:** If a patient calls `555-1001`, the PBX sends SIP headers to our Agent Server tagged with `tenant_id: doc_1`.
*   **Action:** Before the LLM even wakes up, the system knows this session belongs strictly to Doctor 1.

### Level 2: Dynamic System Prompt Injection (The Brain)
With the `tenant_id` established, the Agent Server pulls Doctor 1's specific profile from a secure database (not a simple text file, but a structured config).

```python
# Pseudo-code representation of the isolation
profile = db.get_profile(tenant_id="doc_1") 

system_prompt = f"""
You are Mira, the exclusive receptionist for {profile.name}.
Location: {profile.location}
Specialization: {profile.specialization}
Wait Time: {profile.avg_wait_time}

CRITICAL RULES:
1. You only know about {profile.name}. If asked about ANY other doctor, respond that you only manage the schedule for {profile.name}.
2. {profile.custom_rules}
"""
```
**Security:** The LLM is initialized with *only* this prompt. It physically cannot access Doctor 2's information because Doctor 2's data was never loaded into its context window.

### Level 3: Database & API Segmentation (The Tools)
When the LLM triggers `book_appointment`, it doesn't just send the time. The backend function automatically forces the `tenant_id` into the API call.

```json
// The LLM only sends the time
{
  "function_name": "book_appointment",
  "arguments": {
    "datetime": "2024-10-25T14:00:00"
  }
}
```
*   **Action:** The backend wrapper intercepts this and injects the isolation key: `backend_book(tenant_id="doc_1", datetime="...").` 
*   **Result:** The API physically prevents the LLM from accidentally booking an appointment on Doctor 2's calendar, because the backend enforces the `tenant_id` restriction.

## 3. Database Schema for Doctor Profiles

We will use a central database (e.g., PostgreSQL) to store the configuration for each doctor client.

| id       | phone_number | doctor_name   | specialization | location                 | whatsapp_api_key | calendar_id      |
| :---     | :---         | :---          | :---           | :---                     | :---             | :---             |
| doc_1 | 555-1001     | Dr. Sharma    | Cardiologist   | 101 MG Road, Bangalore   | key_123xyz       | gcal_id_sharma   |
| doc_2 | 555-2002     | Dr. Patel     | Pediatrician   | 24 Linking Road, Mumbai | key_456abc       | gcal_id_patel    |
| doc_3 | 555-3003     | Dr. Gupta     | General        | Sector 4, Noida          | key_789def       | gcal_id_gupta    |

## 4. Why This Approach Wins
1. **Cost Effective:** You run one powerful GPU server (running Qwen/XTTS) that services all 3 doctors simultaneously. You don't need to rent 3 separate GPUs.
2. **Highly Scalable:** Adding "Doctor 4" just means adding a new row to the database and giving them a phone number on the PBX. No new code needs to be written.
3. **Provably Secure:** Because the routing engine (outside the AI) controls which data is fed to the AI, a "jailbreak" is impossible. The LLM simply doesn't have the data of other doctors in its memory to leak.
