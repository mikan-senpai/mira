# AI Receptionist Project Checklist

## Planning & Architecture
- [x] Document High-Level Architecture
- [x] Define Open-Source Technology Stack
- [x] Diagram the System Flow
- [ ] Draft Initial Prompts and Function Call Definitions

## Proof of Concept (PoC) Implementation
- [ ] Setup PBX SIP Trunk connection (Vocode/Pipecat)
- [ ] Integrate local LLM via vLLM (Qwen/Llama-3)
- [ ] Integrate faster-whisper STT
- [ ] Integrate XTTS/ChatTTS
- [ ] Implement Calendar Booking function call
- [ ] Implement WhatsApp notification function call

## Testing & Refinement
- [ ] Test latency (goal < 800ms Time-To-First-Byte)
- [ ] Test Indian accent recognition
- [ ] Refine system prompts for medical context
