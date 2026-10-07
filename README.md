# Meeting Copilot

Real-time AI assistant for live conversations (sales calls, client meetings, internal syncs, language practice, mock interviews). A Chrome extension captures call audio, a Node.js backend transcribes it in real time, detects questions, and streams suggested answers into a side panel.

> Status: design docs only. No code yet.

## Scope

In scope: audio capture, streaming transcription, question detection, answer generation, user context (resume/docs/instructions), side-panel UI, post-call notes, practice mode, security/privacy, observability.

Out of scope (for now): billing, auth provider choice, metering. The backend exposes a usage-event hook so these can be added later (see `docs/05-backend-gateway.md`).

Explicitly **not** a goal: evading detection by other call participants, proctoring tools, or employers. The product is transparent by design (see `docs/10-security-privacy.md`). Use only where AI assistance is permitted.

## Doc index

| # | Doc | Covers |
|---|-----|--------|
| 01 | [Overview & scope](docs/01-overview-and-scope.md) | Users, use cases, non-goals, success metrics |
| 02 | [Architecture](docs/02-architecture.md) | Components, data flow, latency budget, repo layout |
| 03 | [Audio capture](docs/03-extension-audio-capture.md) | MV3 tabCapture, offscreen doc, mic mixing, PCM pipeline |
| 04 | [Streaming STT](docs/04-streaming-stt.md) | Provider choice, protocol, reconnection, transcript model |
| 05 | [Backend gateway](docs/05-backend-gateway.md) | WebSocket protocol, session lifecycle, usage-event hook |
| 06 | [Question detection](docs/06-question-detection.md) | Heuristics + classifier, debouncing, dedupe |
| 07 | [Answer generation](docs/07-answer-generation.md) | Prompting, streaming, cancellation, model routing, caching |
| 08 | [User context](docs/08-user-context.md) | Resume/JD/doc ingestion, instructions, session config |
| 09 | [Side-panel UI](docs/09-side-panel-ui.md) | Screens, states, controls, accessibility |
| 10 | [Security & privacy](docs/10-security-privacy.md) | Threat model, keys, consent, data retention |
| 11 | [Observability & testing](docs/11-observability-testing.md) | Metrics, tracing, test strategy, eval set |
| 12 | [Roadmap & risks](docs/12-roadmap-and-risks.md) | Milestones, risk register |
| ADR | [0001 Extension vs desktop app](docs/adr/0001-extension-vs-desktop.md) | Decision record |

## Planned stack

TypeScript everywhere. Extension: MV3 + React + Vite. Backend: Node.js (Fastify + `ws`), Redis, Postgres. STT: Deepgram (primary), AssemblyAI (fallback). LLMs: provider-agnostic adapter.
