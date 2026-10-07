# 11. Observability & Testing

## Metrics (per session and aggregate)
| Metric | Why |
|--------|-----|
| `stt_final_latency_ms` (speech end to final) | STT health |
| `detect_latency_ms`, `detect_path` (heuristic/classifier) | Detector cost/latency |
| `llm_ttft_ms`, `llm_total_ms`, tokens in/out | Core latency and cost |
| `e2e_question_to_first_token_ms` | North-star latency |
| `ws_reconnects`, `audio_dropped_frames` | Capture reliability |
| `provider_failovers` | Vendor reliability |
| Active sessions, concurrent STT streams | Capacity |
| Question precision/recall (offline) | Quality |

## Tracing and logging
OpenTelemetry traces with a span per stage, correlated by `sessionId`/`questionId`. Structured JSON logs with no transcript text or audio by default (IDs and timings only). Error tracking (e.g. Sentry) with PII scrubbing.

## Alerting
p95 e2e latency over threshold, STT error rate, LLM error/timeout rate, WebSocket close-code spikes, cost per session anomaly.

## Test strategy
| Layer | Tooling | Focus |
|-------|---------|-------|
| Unit | Vitest | Detector heuristics, transcript merge, protocol parsing, prompt builder |
| Contract | zod + shared types | Client/server message compatibility |
| Integration | Vitest + fake STT/LLM servers | Session lifecycle, cancellation, reconnect/resume |
| Audio pipeline | Recorded fixtures | Resampling correctness, frame timing |
| Extension e2e | Playwright (Chromium with extension) | Capture flow on a local test page playing known audio |
| Load | k6/Artillery with synthetic audio | Concurrent sessions, backpressure behavior |
| Eval | `packages/eval` | Detection and answer quality (below) |

Note: `tabCapture` and mic permission flows are hard to automate in CI; keep a manual test script for permission and real Meet/Teams checks per release.

## Eval harness
1. **Detection set:** labeled transcript windows (question / not question / ambiguous). Report precision, recall, F1 per path; track regressions in CI.
2. **Answer set:** (context, question, reference notes) tuples. Score with a rubric: faithfulness to context, relevance, brevity, tone. Use an LLM judge with spot human review; a judge is a noisy proxy, so calibrate against human labels periodically.
3. **STT set:** audio clips with ground truth, in target accents/domains; track WER by provider.
4. **Latency replays:** replay recorded call audio through the gateway, report e2e distribution.

## Release gates
No merge if detection F1 or faithfulness drop beyond agreed deltas; no release if p95 latency regresses past budget in replay tests.
