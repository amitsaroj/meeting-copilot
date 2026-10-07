# 05. Backend Gateway

## Responsibilities
Authenticate the socket, own session state, bridge audio to STT, run question detection, orchestrate LLM answers, persist transcripts, emit usage events.

## Stack
Node.js + TypeScript, Fastify + `ws` (or NestJS WebSocket gateway if team preference). Redis for live state/pubsub, Postgres for durable data. Validate every message with zod schemas from `packages/shared`.

## Session lifecycle
```
CREATED → ACTIVE → (PAUSED) → ENDING → ENDED
```
| Transition | Trigger |
|------------|---------|
| CREATED | `POST /sessions` with config (mode, language, model, context IDs) |
| ACTIVE | first successful `hello` on WS |
| PAUSED | client `pause`, or socket lost (grace period 60 s) |
| ENDING | client `end` or grace period expired; flush STT, persist transcript |
| ENDED | notes job enqueued |

## REST API (minimal)
| Method | Path | Purpose |
|--------|------|---------|
| POST | `/sessions` | Create session, returns `sessionId` + short-lived WS token |
| GET | `/sessions/:id` | Session metadata + transcript |
| POST | `/contexts` | Upload resume/doc (see doc 08) |
| GET | `/contexts` | List user contexts |
| DELETE | `/contexts/:id` | Delete context and derived data |
| GET | `/sessions/:id/notes` | Post-call notes |
| DELETE | `/sessions/:id` | Delete session and transcript |

Authentication provider is deferred; the gateway only requires a verified `userId` from a JWT (middleware boundary: `authenticate(req) -> { userId }`).

## WebSocket protocol
Binary frames = audio. Text frames = JSON control/events.

Client → server:
```json
{ "type": "hello", "token": "...", "sessionId": "...", "lastSeq": 0 }
{ "type": "audio_meta", "channel": "remote", "seq": 12, "tsMs": 3400 }   // precedes binary frame, or use framed header
{ "type": "ask", "text": "optional typed prompt", "withLastN": 3 }
{ "type": "pause" } { "type": "resume" } { "type": "end" }
{ "type": "set_mode", "answerStyle": "concise" }
```
Server → client:
```json
{ "type": "ready", "sessionId": "..." }
{ "type": "transcript", "utterance": { ... } }              // interim and final
{ "type": "question_detected", "id": "q1", "text": "...", "source": "auto" }
{ "type": "answer_start", "questionId": "q1" }
{ "type": "answer_delta", "questionId": "q1", "text": "..." }
{ "type": "answer_end", "questionId": "q1", "usage": { "inputTokens": 0, "outputTokens": 0 } }
{ "type": "warning", "code": "AUDIO_BACKPRESSURE" }
{ "type": "error", "code": "STT_UNAVAILABLE", "retryable": true }
```
Prefer a single binary frame format with a small fixed header (channel byte, seq uint32, ts uint32) over alternating meta/binary messages; it avoids ordering bugs.

## Usage-event hook (billing-ready, billing not implemented)
Emit append-only events to Postgres/queue; no consumer yet.
```ts
type UsageEvent = {
  sessionId: string; userId: string; ts: string;
  kind: 'stt_seconds' | 'llm_tokens_in' | 'llm_tokens_out' | 'session_minutes';
  provider: string; model?: string; quantity: number;
};
```

## Resilience
- Token expiry: 5-minute WS token, refreshable via REST; server closes with code 4401 on expiry.
- Idempotent reconnect using `sessionId` + `lastSeq`.
- Per-session and per-user rate limits (asks/min, concurrent sessions).
- Hard session cap (e.g. 3 h) to bound cost from forgotten sessions.
- Graceful shutdown: drain sockets, flush transcripts.

## Data model (Postgres, abbreviated)
`users`, `sessions(id, user_id, mode, language, model, status, started_at, ended_at)`, `utterances(id, session_id, channel, text, start_ms, end_ms)`, `questions(id, session_id, utterance_ids, text, source)`, `answers(id, question_id, text, model, latency_ms)`, `contexts(id, user_id, type, title, storage_ref, status)`, `usage_events(...)`, `notes(session_id, summary, items jsonb)`.
