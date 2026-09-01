# AI Call Tutor — Ask Questions, Get AI Voice Responses

A **voice-first AI tutor** that runs in the browser. The student speaks a
question through the microphone, the AI understands it, generates an answer
with Google Gemini, and **speaks the answer back aloud** — no typing required.

```
🎤 Student speaks
   ↓
🧠 AI understands the question (speech-to-text)
   ↓
🤖 AI generates the answer (Google Gemini, server-side)
   ↓
🔊 AI speaks the answer (text-to-speech)
   ↓
🔁 listens again for the next question
```

This version is a fully working **browser-based voice tutor prototype**. The
microphone / speaker layer is isolated from the AI logic so it can later be
connected to a real phone-call provider such as **Retell AI** or **Twilio**
(see [Future Real Phone Call Integration](#future-real-phone-call-integration)).

---

## Project purpose

Build a hands-free learning assistant where the entire interaction happens by
voice:

- Ask any question by speaking.
- Get a spoken, context-aware answer.
- Ask follow-up questions that reference earlier topics ("What are its types?").
- The tutor stays quiet until you speak, handles silence gracefully, ends on
  "goodbye", and moderates abusive language.

It is **not** a text chatbot. A small live transcript exists only as a
secondary aid — the primary experience is voice in / voice out.

---

## Architecture

```
Browser (React + Vite + TypeScript)
├── Microphone  →  Web Speech API (SpeechRecognition)  →  text
├── text  →  POST /functions/v1/ai-tutor  (Supabase Edge Function)
│            └── calls Google Gemini generateContent with GEMINI_API_KEY (secret)
├── answer  →  SpeechSynthesis API  →  spoken audio
└── loop: listen → process → speak → listen again
```

**Why a Supabase Edge Function?** The Gemini API key must never reach the
browser. The edge function (`supabase/functions/ai-tutor/index.ts`) holds the
key as a Supabase secret, calls Gemini server-side, and returns only the answer
text to the frontend.

---

## Technologies

- **React 18** + **TypeScript** + **Vite**
- **Tailwind CSS** for styling
- **lucide-react** for icons
- **Web Speech API** (`SpeechRecognition` / `webkitSpeechRecognition`) — speech-to-text
- **SpeechSynthesis API** — text-to-speech
- **MediaDevices.getUserMedia** — microphone permission
- **Supabase Edge Functions** (Deno) — secure Gemini proxy
- **Google Gemini API** (`gemini-1.5-flash`) — AI answers

---

## Features

- Voice-first flow: speak a question, hear the answer.
- Automatic greeting on start, then auto-listen — no extra clicks.
- Continuous conversation loop (listen → process → speak → listen again).
- Follow-up questions with conversation memory (last 10 turns sent for context).
- Silence handling: first prompt ("Are you still there?"), then end after a
  second silence.
- End-session spoken commands (`bye`, `goodbye`, `stop`, `end`, etc.).
- Abusive-language moderation with configurable warnings before ending.
- State machine: IDLE → STARTING → GREETING → LISTENING → PROCESSING → SPEAKING
  → (loop) → ENDING → ENDED (plus SILENCE_WARNING, ERROR).
- Session timer + question counter (only real questions counted).
- Microphone permission requested **only after** clicking START TUTOR.
- Error handling for mic denied, unsupported browser, AI/network failures.
- Hidden developer/debug panel.
- Responsive design (desktop, tablet, Android, iPhone).

---

## Project structure

```
src/
  components/
    VoiceTutor.tsx        main screen layout
    MicrophoneButton.tsx  large animated mic button
    StatusIndicator.tsx   status pill (Listening / Thinking / Speaking…)
    SessionTimer.tsx      timer + question counter
    Transcript.tsx        secondary live transcript (not the primary UI)
    DebugPanel.tsx        hidden developer log
  services/
    speechRecognition.ts  Web Speech API wrapper (STT)
    speechSynthesis.ts    SpeechSynthesis wrapper (TTS)
    tutorApi.ts           calls the secure edge function
    moderation.ts         abusive-language filter
  hooks/
    useVoiceTutor.ts      state machine + conversation loop
  types/
    index.ts              shared types
  utils/
    config.ts             env-driven config + spoken strings
    format.ts             time formatting helpers
  App.tsx
  main.tsx
  index.css

supabase/
  functions/
    ai-tutor/
      index.ts            secure Google Gemini proxy edge function
```

---

## Installation

Requirements: Node 18+.

```bash
# 1. install dependencies
npm install

# 2. copy the example env file (Supabase vars are pre-filled by Bolt)
cp .env.example .env

# 3. run the dev server
npm run dev
```

The Supabase URL and anon key are pre-populated by Bolt in `.env`. The Gemini
API key is set as a Supabase Edge Function secret (see below).

---

## Environment variables

`.env.example`:

```env
GEMINI_API_KEY=
SILENCE_TIMEOUT_MS=8000
SECOND_SILENCE_TIMEOUT_MS=10000
MAX_ABUSIVE_WARNINGS=2
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
```

| Variable | Where it's used | Notes |
|---|---|---|
| `GEMINI_API_KEY` | Edge Function **secret** only | NEVER in frontend code. Set as a Supabase secret. |
| `SILENCE_TIMEOUT_MS` | Frontend (VITE_) | First silence prompt delay. |
| `SECOND_SILENCE_TIMEOUT_MS` | Frontend (VITE_) | Second silence → end session. |
| `MAX_ABUSIVE_WARNINGS` | Frontend (VITE_) | Warnings before abusive language ends the session. |
| `VITE_SUPABASE_URL` | Frontend | Pre-populated by Bolt. |
| `VITE_SUPABASE_ANON_KEY` | Frontend | Pre-populated by Bolt. |

> The silence/abuse values are read from `VITE_`-prefixed env vars at build
> time (Vite only exposes `VITE_` vars to the browser). Sensible defaults apply
> if they are missing.

---

## Gemini API configuration

The API key is **not** read from `.env` by the browser. It is stored as a
Supabase Edge Function **secret** named `GEMINI_API_KEY` and read inside the
edge function via `Deno.env.get("GEMINI_API_KEY")`.

The edge function uses the stable model `gemini-1.5-flash-latest` via the
`generateContent` REST endpoint. No `OPENAI_MODEL` or `OPENAI_API_KEY` is
used anywhere in the project.

To set the secret, use the Supabase dashboard → Project Settings → Edge
Functions → Secrets, and add:

```
GEMINI_API_KEY=AIza...your key...
```

The edge function is deployed at:
`https://<your-supabase-project>.supabase.co/functions/v1/ai-tutor`

---

## Running locally

```bash
npm install
npm run dev
```

Open the printed URL (usually http://localhost:5173) in **Google Chrome** (the
Web Speech API has the best support there). Click **START TUTOR**, allow
microphone access, then speak your question.

---

## Microphone permissions

- The browser only asks for microphone permission **after** you click
  **START TUTOR** — never on page load.
- If you deny permission, the tutor shows an error and does not start.
- To reset permission: click the lock/site-info icon in the address bar →
  Allow microphone → reload.

---

## Browser compatibility

| Feature | Chrome (desktop/Android) | Edge | Safari (desktop/iOS) | Firefox |
|---|---|---|---|---|
| SpeechRecognition | ✅ | ✅ | ⚠️ partial | ❌ |
| SpeechSynthesis | ✅ | ✅ | ✅ | ✅ |
| getUserMedia | ✅ | ✅ | ✅ | ✅ |

**Recommended: Google Chrome on desktop or Android.** If speech recognition is
unavailable, the app shows a warning instead of crashing.

---

## Testing the complete flow

1. Open the app → click **START TUTOR** → allow microphone.
2. AI speaks: *"Hello! Welcome to AI Tutor. I'm your AI learning assistant.
   Please ask me your question."*
3. Say: **"What is Python?"** → AI speaks an answer → auto-listens again.
4. Say: **"What is a Python function?"** → AI speaks → auto-listens.
5. Say: **"Give me an example."** → AI uses context and speaks an example.
6. Say: **"That's all. Goodbye."** → AI speaks the goodbye → session ends.

### Silence test

Start the tutor, then say nothing. After ~8s the AI says *"Are you still
there?…"* and listens again. After another ~10s of silence it says *"It looks
like you're away…"* and ends the session.

### Abuse test

Say inappropriate language. The AI warns you by voice. Repeat past
`MAX_ABUSIVE_WARNINGS` and the AI ends the session by voice.

---

## Deployment

The frontend builds to static assets:

```bash
npm run build      # outputs to dist/
npm run preview    # preview the production build locally
```

Deploy `dist/` to any static host (Vercel, Netlify, Cloudflare Pages, etc.).
The edge function is already deployed on Supabase; just make sure the
`GEMINI_API_KEY` secret is set in the Supabase project.

---

## Future Real Phone Call Integration

This is a browser prototype — it is **not** a real PSTN phone call. The code is
structured so a telephony/voice-agent provider can be connected later:

- `services/tutorApi.ts` is the single integration point for AI answers. A
  phone provider (e.g. **Retell AI**) can call the same Gemini-backed logic or
  host its own LLM call.
- `services/speechRecognition.ts` and `services/speechSynthesis.ts` are the
  browser mic/speaker layer. With Retell/Twilio, this layer is replaced by the
  provider's audio stream (telephone audio ↔ provider media stream).
- `hooks/useVoiceTutor.ts` contains the conversation state machine (listen →
  process → speak → listen). The same state machine maps onto a telephony
  call's media events, so the logic is reusable.

Typical next step: add a Retell AI (or Twilio Voice + Twilio's Media Streams)
webhook endpoint as a second edge function that bridges telephone audio to the
same Gemini tutor prompt, while keeping the browser UI as a "web call" option.

---

## Notes

- Conversation history is kept only in memory for the current session and is
  cleared when a new session starts. No personal conversation data is stored.
- Only valid questions increment the question counter (silence, goodbye,
  abusive warnings, empty speech, and errors do not).
- The moderation filter is intentionally simple (prototype-grade). For
  production, use a dedicated moderation service.
