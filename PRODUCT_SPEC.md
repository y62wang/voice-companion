# Voice Companion — Product Spec & Software Design
**Version 0.1 | Author: Dandi AI | Date: 2026-04-06**

---

## 1. Product Overview

**Voice Companion** is an always-on AI companion app for iOS and Android. Users open the app once, activate listening, and can talk to their AI naturally — while doing other things, without looking at their phone. The AI has persistent memory, knows the user personally, and can take real actions (check calendar, send messages, query email, look up stocks, etc.) via a connected OpenClaw backend.

**Core differentiator vs ChatGPT Voice / Siri / Alexa:**
- Persistent long-term memory across sessions
- Knows who you are, your habits, goals, relationships
- Can take real-world actions through OpenClaw integrations
- Feels like a companion, not a tool

---

## 2. Target Users

- Professionals who want a smart personal assistant they can talk to hands-free
- People who want an AI companion with real continuity and memory
- Power users of OpenClaw who want a better mobile interface

---

## 3. Core Features (MVP)

### 3.1 Always-On Voice
- User opens app and taps "Start Listening"
- App stays in foreground (or background audio session active)
- Detects voice activity (VAD) automatically
- Sends speech → STT → OpenClaw → TTS response
- Works while screen is off (background audio)

### 3.2 Text Chat Fallback
- Full chat interface alongside voice
- User can type instead of speaking
- AI responses shown as text + optionally read aloud

### 3.3 Chat History
- All conversations stored locally + synced to OpenClaw
- Swipe up or tap to view full history
- Searchable

### 3.4 AI Personality
- AI persona configured on OpenClaw side
- App displays AI name, avatar, voice
- TTS voice powered by ElevenLabs (configured per user)

### 3.5 Real Actions
- App transparently shows when AI is "doing something" (checking email, calendar, etc.)
- Results surfaced naturally in conversation

---

## 4. User Flow

### 4.1 Onboarding

```
Download App
     ↓
Create Account (email + password OR Sign in with Apple/Google)
     ↓
Pair with OpenClaw instance
     ↓
Set AI name + voice (optional)
     ↓
Start talking
```

### 4.2 OpenClaw Pairing (like Telegram Bot setup)

**Option A — QR Code Pairing (Recommended)**
1. User opens OpenClaw web dashboard (or runs `openclaw pair-app`)
2. Dashboard shows a one-time QR code with a pairing token
3. User scans QR code in the app
4. App stores: `{ openclaw_url, pairing_token }`
5. OpenClaw verifies token, issues a long-lived session key
6. Done — app is now paired

**Option B — Manual Token Entry**
1. User runs `openclaw generate-app-token` on their server
2. Gets a token like `vc_abc123xyz...`
3. Pastes into app manually

**Security:**
- Pairing tokens are one-time use, expire in 10 minutes
- Session keys are long-lived (30 days), rotatable
- All communication over HTTPS/WSS
- Session key stored in iOS Keychain / Android Keystore
- No plaintext credentials ever stored on device

### 4.3 Daily Use

```
Open App → Tap "Start Listening"
     ↓
App connects to OpenClaw via WebSocket
     ↓
User speaks → VAD detects → audio chunk sent to STT
     ↓
Text sent to OpenClaw → AI processes → response text back
     ↓
TTS plays response via ElevenLabs
     ↓
Chat history updated
```

---

## 5. Architecture

### 5.1 High-Level Diagram

```
[Mobile App iOS/Android]
        |
        | HTTPS/WSS (TLS 1.3)
        |
[OpenClaw Gateway] ← user's own server (or cloud)
        |
   ┌────┴────┐
   │  Agent  │ ← LLM, memory, tools
   └────┬────┘
        |
   ┌────┴──────────────────┐
   │ Integrations          │
   │ Gmail / Calendar      │
   │ Notion / Things       │
   │ Stock data / Weather  │
   └───────────────────────┘
```

### 5.2 Mobile App Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| Framework | **React Native** | Single codebase for iOS + Android |
| Language | **TypeScript** | Type safety, good ecosystem |
| Voice/Audio | **expo-av** + native modules | Cross-platform audio |
| STT | **Whisper API** (OpenAI) or **iOS/Android native STT** | Whisper = best accuracy; native = free + offline |
| TTS | **ElevenLabs API** | Best voice quality; user configures voice ID |
| State | **Zustand** | Lightweight, simple |
| Storage | **MMKV** (local) + **SQLite** (chat history) | Fast, persistent |
| Keychain | **react-native-keychain** | Secure credential storage |
| Networking | **WebSocket** (persistent) + **fetch** (REST) | Real-time + fallback |
| Navigation | **React Navigation** | Standard |
| UI | **Custom components** + **Reanimated 3** | Smooth animations |

### 5.3 OpenClaw Backend (existing, extend slightly)

OpenClaw already handles:
- LLM inference
- Memory (MEMORY.md, daily logs)
- Tool execution (Gmail, Calendar, etc.)
- Telegram channel

**New additions needed:**
- `app` channel plugin (similar to Telegram plugin)
- WebSocket endpoint for real-time messaging: `wss://your-openclaw/app/ws`
- App pairing token generation + management
- REST endpoint: `POST /app/pair`, `GET /app/session`

### 5.4 Communication Protocol

**WebSocket messages (JSON):**

```json
// App → Server: send message
{
  "type": "message",
  "text": "帮我查一下今天的日历",
  "session_key": "vc_abc123"
}

// Server → App: AI response
{
  "type": "response",
  "text": "今天下午3点有个meeting，没有其他安排",
  "tts_url": "https://...elevenlabs.../audio.mp3",  // optional
  "done": true
}

// Server → App: streaming partial response
{
  "type": "response",
  "text": "今天下午3点",
  "done": false
}

// Server → App: tool activity
{
  "type": "activity",
  "text": "Checking your calendar..."
}
```

---

## 6. Security Design

### 6.1 Authentication
- Account auth: **JWT tokens** (access 15min + refresh 30 days)
- OpenClaw pairing: **one-time token** → permanent session key
- Session key: stored in **iOS Keychain / Android Keystore** (hardware-backed when available)

### 6.2 Transport Security
- All traffic over **TLS 1.3**
- Certificate pinning for OpenClaw connection (optional, for power users)
- WebSocket connections authenticated via session key in header

### 6.3 Data Privacy
- Voice audio: never stored on servers (process and discard)
- Chat history: stored locally on device + on user's own OpenClaw server
- No central servers have access to user conversations
- Each user connects to **their own** OpenClaw instance — no shared infrastructure

### 6.4 Multi-Device
- Multiple devices can pair with same OpenClaw
- Each device gets its own session key
- User can revoke device access from OpenClaw dashboard

---

## 7. UI/UX Design Direction

**Primary mode: Voice-First (Design 3)**
- Full-screen waveform visualizer
- Floating transcript (last 2-3 exchanges)
- Swipe up → chat history panel

**Secondary mode: Chat**
- Standard chat bubbles
- Input bar with mic button
- Voice/text toggle

**Key interactions:**
- Tap screen → toggle listening on/off
- Hold mic button → push-to-talk mode
- Swipe up → chat history
- Long press message → copy / share / delete

---

## 8. MVP Scope (v1.0)

### In Scope
- [x] iOS + Android app (React Native)
- [x] OpenClaw pairing (QR + manual token)
- [x] Always-on voice listening (foreground)
- [x] STT (Whisper API)
- [x] TTS (ElevenLabs)
- [x] Chat history (local SQLite)
- [x] Text input fallback
- [x] Basic settings (AI name, voice, server URL)

### Out of Scope (v2+)
- [ ] Background listening (system-level, like Siri)
- [ ] Apple Watch companion
- [ ] Alexa / HomePod skill
- [ ] Multiple AI personas
- [ ] App Store listing
- [ ] Payments / subscription

---

## 9. Development Phases

### Phase 1 — Foundation (2-3 weeks)
- React Native project setup
- OpenClaw WebSocket channel plugin
- Pairing flow (QR + token)
- Basic text chat working end-to-end

### Phase 2 — Voice (2 weeks)
- STT integration (Whisper)
- TTS integration (ElevenLabs)
- VAD (voice activity detection)
- Voice UI (waveform, transcript)

### Phase 3 — Polish (1-2 weeks)
- Chat history + search
- Settings screen
- Animations (Reanimated)
- Error handling + reconnection logic

### Phase 4 — Testing (1 week)
- iOS + Android device testing
- Performance optimization
- Beta release (TestFlight + Play Store internal)

**Total estimated: 6-8 weeks solo, 3-4 weeks with a frontend partner**

---

## 10. Open Questions

1. **STT cost**: Whisper API ~$0.006/min. For heavy users, this adds up. Consider on-device Whisper (whisper.cpp) for v2.
2. **Background listening on iOS**: Apple restricts this. Users must keep app in foreground, or use AirPlay audio session hack. Full background = needs Siri integration (much harder).
3. **OpenClaw hosting**: Users need to run their own OpenClaw. Barrier to entry is high for non-technical users. Consider managed OpenClaw cloud hosting as a paid tier.
4. **TTS latency**: ElevenLabs adds ~500ms-1s latency. Need to show "thinking" animation to mask this.
5. **App Store approval**: Apps with AI companions have faced review issues. Need clear privacy policy and content guidelines.

---

*This document is a living spec. Update as decisions are made.*
