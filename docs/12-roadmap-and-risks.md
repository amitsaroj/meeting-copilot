# 12. Roadmap & Risks

## Milestones
| M | Goal | Exit criteria |
|---|------|---------------|
| M0 | Feasibility spike | tabCapture + mic → offscreen → gateway → Deepgram → transcript in panel; measure latency |
| M1 | Core loop | Detection (heuristics), streamed answers, manual Answer, session lifecycle, reconnect |
| M2 | Context | Resume/doc upload, extraction, stuffing with caching, instructions |
| M3 | Quality | Classifier stage, eval harness, model routing, failover, observability |
| M4 | Notes & practice | Post-call notes, practice mode, history/export, deletion flows |
| M5 | Hardening | Security review, load test, privacy docs, store submission |
| P2 | Extensions | Screenshot/code explanation, multi-language auto-detect, group-call speaker separation, desktop app (ADR-0001) |

Billing/auth integration slots in after M1 using the usage-event hook (doc 05).

## Risk register
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Latency target missed | Medium | High | M0 spike measures before building more; shorten endpointing, fast model, prompt caching |
| Chrome Web Store rejection/removal | Medium | High | Clear single purpose, transparent disclosures, no deceptive features; keep a sideload/enterprise distribution path |
| Question detection noisy | High | Medium | Suggest mode default, precision-weighted tuning, manual button |
| Hallucinated/fabricated experience in answers | Medium | High | Context-only rule, faithfulness eval, "missing info" behavior |
| Legal exposure (recording consent, voice data) | Medium | High | Consent helper, disclosures, counsel review, retention controls |
| Browser-only coverage too narrow | Medium | Medium | Validate with users in M1; ADR-0001 desktop path |
| Provider cost/price changes | Medium | Medium | Adapters, cost metrics per session, model routing |
| Market saturation / low retention | High | High | Focus on a recurring use case (meetings/sales) rather than one-off events; validate demand before scaling |
| MV3 lifecycle bugs (service worker/offscreen) | Medium | Medium | Audio and socket in offscreen doc, heartbeat, e2e tests |

## Open questions
1. Which single segment first: sales calls, general meetings, or practice? Decide via 10-15 user conversations.
2. Group-call handling: is channel-level separation enough?
3. Self-host STT later for cost or privacy?
