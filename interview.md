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

**In plain words: the whole interview is recorded right inside the candidate's own desktop app, and once it's done the video is saved to our online storage (S3) so the employer can watch it later. We do not use any outside recording tool to do this.**

Think of it like recording a video on your phone and then backing it up to the cloud — except it all happens automatically inside our app while the interview runs.

### Step 1 — The desktop app records everything (on the candidate's computer)

The interview takes place inside our **desktop app**, which runs on the candidate's own laptop. While the interview is going on, the app quietly records the whole session — the candidate's camera, their screen, and the audio (both the candidate's voice and the AI interviewer's voice mixed together). It saves all of this as one normal video file.

- The tool that does the recording is the **MediaRecorder** — this is a recording feature **already built into the app itself**. We did not buy or plug in any third-party recorder (no Zoom, no screen-recording software). The video is saved in the standard **WebM** format that any browser can play.
- As it records, the app keeps writing the video to the computer's disk in small pieces, so even a long interview never overloads memory or gets lost.

### Step 2 — The finished video is sent to online storage (S3)

When the interview ends, the recorded video needs to move from the candidate's laptop to a safe place online where the employer can later watch it. That safe place is **Amazon S3** — basically a secure online hard drive (like Google Drive or Dropbox, but for our system).

The candidate's app is **not** trusted with the password to our storage. Instead, we use a safer approach:

1. The app asks our **backend**: "I'm ready to upload the recording."
2. The backend replies with a **one-time upload link** (called a "presigned URL") — a temporary pass that allows uploading exactly one file, then expires.
3. The app uploads the video **straight to S3** using that link. If the internet drops mid-upload, it automatically retries.
4. The app tells the backend "the video is now stored here," along with the transcript and any proctoring flags, which the backend saves in the database.

### The flow, step by step

```
  CANDIDATE'S DESKTOP APP                    OUR BACKEND                ONLINE STORAGE (S3)
  ───────────────────────                    ───────────                ───────────────────
  Records the interview
  as a video file (WebM)
        │
        │ 1. "Ready to upload" ───────────▶
        │ 2. ◀─── one-time upload pass ────
        │
        │ 3. Uploads the video directly ───────────────────────────────▶  Video saved
        │
        │ 4. "Done — saved here" + transcript ──▶  Saves the details
        │                                          in the database
```

**Why it's done this way:** the video goes **straight from the app to storage** instead of passing through our backend — that makes it fast and keeps our costs low. The backend only hands out the temporary pass and remembers where the video was stored, so it can find it again when the employer wants to watch it.

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
