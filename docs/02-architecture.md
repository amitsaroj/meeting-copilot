# 02. Architecture

## Component view
```
┌──────────────────────── Chrome (MV3) ────────────────────────┐
│ Service worker ── orchestrates, tabCapture.getMediaStreamId   │
│ Offscreen doc ─── getUserMedia(tab)+getUserMedia(mic)         │
│                   AudioWorklet → 16 kHz PCM16 frames          │
│                   WebSocket client (JWT)                      │
│ Side panel (React) ─ transcript, answers, controls            │
└───────────────┬───────────────────────────────────────────────┘
                │ WSS: audio frames up, events down
┌───────────────▼────────────── Gateway (Node.js) ──────────────┐
│ Session manager ─ per-session state (Redis)                   │
│ STT bridge ────── Deepgram WS (per channel)                   │
│ Question detector ─ heuristics + small LLM                    │
│ Answer service ─── context assembly + LLM streaming           │
│ Notes service ──── post-call summary (async job)              │
│ Usage-event emitter ─ hook for future billing                 │
└───────┬──────────────┬──────────────┬─────────────────────────┘
        │              │              │
   Postgres         Redis        STT / LLM providers
 (sessions,      (live session   (Deepgram, AssemblyAI,
  transcripts,    state, pubsub)  Anthropic/OpenAI/Google)
  contexts)
```

## Data flow (happy path)
1. User clicks Start in side panel. Service worker obtains a tab `streamId`; offscreen document opens tab + mic streams.
2. AudioWorklet downsamples to 16 kHz mono PCM16, frames of 20-100 ms, tagged channel `remote` (tab) or `local` (mic).
3. Frames go over WSS to the gateway, which forwards each channel to its own STT stream.
4. Interim/final transcript events flow back to the side panel and into the detector.
5. On a final `remote` utterance classified as a question, the answer service builds the prompt and streams tokens to the panel.
6. On session end, the full transcript is persisted and a notes job runs.

## Latency budget (target, p50)
| Stage | Budget |
|-------|--------|
| Audio frame capture + network up | 100 ms |
| STT endpointing + final result | 300-500 ms |
| Question detection (heuristic path) | < 20 ms |
| Question detection (classifier path) | 150-300 ms |
| LLM time to first token | 400-800 ms |
| Network down + render | 50 ms |
| **Total** | **~1.0-1.5 s** |

## Key design decisions
- **Server-side keys only.** Extension never holds STT/LLM keys.
- **Two channels, not diarization.** Tab audio = remote speakers, mic = user. Cheap and accurate for 1:1 and most small calls; group-call speaker separation within the tab channel is a later feature.
- **Offscreen document owns audio + WebSocket.** MV3 service workers are terminated when idle; they must not hold long-lived connections.
- **Provider-agnostic adapters** for STT and LLM (interface in `packages/shared`).
- **Cancel-on-new-question.** Only one answer stream active per session; a new question aborts the previous generation.

## Proposed repo layout
```
apps/
  extension/        # MV3, Vite, React (side panel), offscreen, service worker
  gateway/          # Fastify + ws, session/STT/LLM orchestration
packages/
  shared/           # protocol types (zod), provider interfaces
  eval/             # question-detection and answer-quality eval harness
docs/
```
Tooling: pnpm workspaces, TypeScript strict, ESLint, Vitest, Playwright (extension e2e).

## Scalability notes
- Gateway is stateful per connection (WebSockets + upstream STT sockets). Scale horizontally with sticky routing by session ID; keep durable state in Redis/Postgres so a reconnect can land on another node.
- Main cost and capacity drivers: concurrent STT streams (2 per session) and LLM tokens. Both scale linearly with active sessions.
