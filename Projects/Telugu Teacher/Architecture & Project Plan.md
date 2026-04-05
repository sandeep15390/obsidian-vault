# Architecture & Project Plan

## System Overview

```
Student (Browser)
      |
      | WebRTC (audio/video)
      v
LiveKit Server (Home Server)
      |
      | LiveKit Agent SDK
      v
Teacher Voice AI Agent (joins room on student connect)
```

## Components

### 1. LiveKit Server
- Self-hosted on home server
- Manages real-time media rooms ("classes")
- Each class session = one LiveKit room
- Handles WebRTC signaling and media routing between student and AI agent

### 2. Web UI
- Student-facing interface to browse and join a class
- "Attend a Class" button creates or joins a LiveKit room
- Connects to LiveKit room via LiveKit JS/React SDK
- Handles microphone input and audio playback

### 3. Teacher Voice AI Agent
- Runs as a LiveKit agent (Python or Node)
- Listens for room participant events — triggers when a student joins
- Joins the LiveKit room automatically on student connect
- Pipeline:
  - **STT** — transcribes student speech (e.g. Deepgram / Whisper)
  - **LLM** — Telugu teaching logic, lesson flow, corrections (e.g. Claude / GPT-4o)
  - **TTS** — speaks responses back in the room (e.g. ElevenLabs / Cartesia)

## Data Flow

1. Student opens Web UI and clicks "Attend a Class"
2. Web UI requests a LiveKit token from backend and joins a room
3. Room join event triggers the Teacher Agent to connect to the same room
4. Student speaks → STT transcribes → LLM generates teacher response → TTS speaks audio back into room
5. Session continues until student leaves; agent disconnects and room closes

## Infrastructure

| Component | Where |
|---|---|
| LiveKit Server | Home server (Docker) |
| Agent Runner | Home server (alongside LiveKit or separate process) |
| Web UI | Served from home server or static host (e.g. Vercel) |
| Backend (token server) | Home server (small API — Node/Python) |

## Tech Stack

| Layer | Choice |
|---|---|
| Real-time media | LiveKit (self-hosted) |
| Agent framework | LiveKit Agents SDK (Python) |
| STT | Deepgram or OpenAI Whisper |
| LLM | Claude or GPT-4o |
| TTS | ElevenLabs or Cartesia |
| Web UI | React + LiveKit React Components SDK |
| Token server | FastAPI or Express |

## Milestones

- [ ] Stand up LiveKit server on home server (Docker Compose)
- [ ] Build token server — generates LiveKit JWT for student on room join
- [ ] Build Web UI — "Attend a Class" page connects student to LiveKit room
- [ ] Build basic Teacher Agent — joins room when student connects, echoes speech back
- [ ] Integrate STT pipeline (Deepgram / Whisper)
- [ ] Integrate LLM with Telugu teaching prompt / lesson logic
- [ ] Integrate TTS — agent speaks responses as audio in room
- [ ] End-to-end test: student joins → teacher greets → lesson begins
- [ ] Lesson curriculum design (beginner lesson 1+)
- [ ] Polish Web UI and deploy

## Open Questions
- Single shared classroom or private room per student?
- Lesson structure: free conversation vs. structured curriculum?
- How to handle Telugu script vs. Romanized Telugu (Tenglish) in prompts?
- Authentication — open access or student accounts?
