# Technology Stack & Responsibilities

This document explains **which technology we use for each capability and what exactly it is responsible for** — focused on the two core flows: (1) the real-time AI voice interview, and (2) recording the session and storing it in S3. Everything is built on **our own streaming infrastructure** — we do **not** use Zoom, Twilio Video, or any third-party meeting product.

---

## Technology at a Glance

| Technology | Layer | Responsible for |
|---|---|---|
| **WebRTC** | Transport | The underlying real-time audio/video protocol browsers use |
| **LiveKit** | Transport | Carries the live audio between the candidate and the AI in real time (built on WebRTC). Our own infra — replaces Zoom. |
| **LiveKit Agents** | Backend | The "AI interviewer" program that joins the call and runs the speech pipeline |
| **Deepgram** | AI / Speech-to-Text | Converts the candidate's spoken words into text |
| **OpenAI (LLM)** | AI / Reasoning | The brain — reads the transcript and decides what to ask/say next |
| **OpenAI (TTS)** | AI / Text-to-Speech | Converts the AI's text reply into a spoken voice |
| **Silero VAD** | AI / Voice detection | Detects when the candidate has started/stopped speaking (turn-taking) |
| **MediaRecorder API** | Recording | Browser-native API that records the session video/audio — no third-party recorder |
| **AWS S3** | Storage | Stores the recorded interview file |
| **HTML5 `<video>`** | Playback | Plays the recording back in the employer's browser |

---

## Capability 1 — Real-Time AI Voice Interview

**The question: when the candidate speaks, how does it reach the AI, and how does the AI talk back?**

There are two participants in a live "room": the **candidate's microphone** and the **AI agent**. They are connected by **LiveKit**.

### What each piece does

- **LiveKit** — the real-time pipe. It streams the candidate's microphone audio up to the AI agent, and streams the AI's voice back down to the candidate, with very low latency. It uses **WebRTC** under the hood (the same technology video-call apps use), so we get noise suppression, echo cancellation, and auto-reconnection for free. **This is why we don't need Zoom — LiveKit is our own real-time transport.**

- **LiveKit Agents (the AI agent)** — a program running on **our backend** that joins the same LiveKit room as a "participant." It is the orchestrator: it takes incoming audio, runs it through the speech pipeline below, and speaks the answer back into the room. It also keeps the interview on track (asks the planned questions, manages time, shows code on screen when needed).

- **Deepgram (Speech-to-Text)** — as the candidate speaks, their audio is fed to Deepgram, which transcribes it into text in real time (model `nova-3`).

- **OpenAI LLM (the brain)** — the transcribed text is sent to the language model, which understands the answer and generates the next thing to say (a follow-up question, a clarification, or the next question).

- **OpenAI TTS (Text-to-Speech)** — the model's text reply is converted into a natural spoken voice and played back to the candidate through LiveKit.

- **Silero VAD (Voice Activity Detection)** — listens for when the candidate starts and stops talking, so the AI knows when it's their turn to respond and doesn't talk over the candidate.

### The flow, step by step

```
 Candidate speaks into mic
        │
        ▼
 ┌──────────────┐   live audio    ┌─────────────────────────────────────────┐
 │   LiveKit    │ ──────────────▶ │           AI Agent (backend)            │
 │ (WebRTC)     │                 │                                         │
 │              │                 │  1. Silero VAD → "candidate stopped"    │
 │              │                 │  2. Deepgram   → speech becomes text    │
 │              │                 │  3. OpenAI LLM → decides what to say     │
 │              │                 │  4. OpenAI TTS → text becomes voice      │
 │              │ ◀────────────── │                                         │
 └──────────────┘   AI voice      └─────────────────────────────────────────┘
        │
        ▼
 Candidate hears the AI's reply
```

So the loop is: **candidate's voice → LiveKit → Deepgram (text) → OpenAI LLM (decision) → OpenAI TTS (voice) → LiveKit → candidate's ears.** This whole round-trip happens in roughly a second, which is why it feels like a natural conversation.

> Note: if a question needs code, the LLM writes it in its reply, and we **strip the code out before TTS** so the AI doesn't read code aloud — instead the code is pushed to the candidate's screen over a LiveKit data channel.

---

## Capability 2 — Recording the Interview & Storing in S3

**The question: how do we record the session video and get it into S3 — and do we need a third-party tool for it? No.**

### What each piece does

- **MediaRecorder API** — a **built-in browser API** (available in our Electron app — no external library, no Zoom recorder). It captures the screen + camera + the mixed audio (candidate's voice plus the AI's voice) and packages it as a standard **WebM** video file. To avoid memory issues, it writes the file to disk in 1 MB chunks as it goes.

- **AWS S3** — where the finished recording lives. We use Amazon's official AWS SDK on the backend.

- **Presigned URL** — the key trick. The desktop app never holds AWS passwords. Instead, when it's ready to upload, it asks our backend for a **presigned upload URL** — a one-time, time-limited link that grants permission to upload exactly one file. The app then uploads the video straight to S3 using that link (with automatic retry if the network hiccups).

### The flow, step by step

```
  Desktop app (MediaRecorder records WebM)
        │
        │ 1. "I'm ready to upload" ──────────────▶  Backend
        │ 2.  ◀────── one-time presigned upload URL ──
        │
        │ 3. Upload the WebM file directly ──────────────────▶  AWS S3  (stored)
        │
        │ 4. "Here's the file location + transcript + proctor events" ──▶ Backend (saves to DB)
```

The upload goes **directly from the app to S3** (not through our backend), which keeps it fast and cheap. The backend only hands out the permission slip and records where the file landed.

---

## Capability 3 — Playing the Recording Back (Employer Side)

When the employer opens a finished interview in the web dashboard, they see the recording in a plain **HTML5 `<video>` player** — again, no third-party video product.

- The browser does **not** get a direct S3 link. Instead it asks our backend, "give me this recording."
- The backend fetches the file from S3 and **streams it back** to the browser as a blob.
- **Why route through the backend?** (1) We can enforce permissions — only the employer who owns that interview can watch it. (2) S3 file locations stay private and can't be shared or leaked.

```
  Employer browser ──"show recording"──▶ Backend ──fetch──▶ S3
                   ◀── streamed video ──         ◀── file ──
        │
        ▼
  Plays in a standard <video> tag
```

---

## Capability 4 — Live Proctoring (Anti-Cheating)

While the interview runs, the desktop app watches the candidate **locally on their machine** (nothing is sent to a third party):

- **MediaPipe (Google's on-device vision library)** — tracks the candidate's face and eyes using a 478-point face mesh: detects looking away, multiple faces, no face, or reading off-screen.
- The app also scans for known cheating apps (ChatGPT, Cluely, screen-share tools, etc.), watches the clipboard for paste attempts, and detects when the candidate switches away from the window.

---

## Why We Don't Use Zoom or Other Third-Party Meeting Tools

- **Real-time audio:** We run our **own LiveKit (WebRTC) infrastructure**. This lets the AI agent join the call as a true participant and process audio in real time — something a closed product like Zoom would not allow.
- **Recording:** We use the **browser's built-in MediaRecorder** — no external recorder, no per-minute recording fees.
- **Storage:** Files go to **our own S3 bucket**, fully under our control.
- **Speech AI:** Deepgram (speech-to-text) and OpenAI (reasoning + voice) are best-in-class building blocks we wire together ourselves — so we control the conversation logic, cost, and data.

The result: the entire pipeline — from the candidate's first word to the stored recording the employer watches — runs on **infrastructure and AI components we own and control**, with no dependency on a third-party meeting platform.
