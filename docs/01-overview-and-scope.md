# 01. Overview & Scope

## Problem
During live conversations people lose time recalling facts, structuring answers, or capturing notes. Existing tools are either post-call only (transcripts/summaries) or desktop apps with heavy installs.

## Target users
1. Sales/CS reps on browser-based calls (objection handling, product facts, next steps).
2. Professionals in client meetings (live notes, recall of prior docs).
3. Language learners and candidates practicing (mock sessions).
4. Candidates in interviews **where AI assistance is permitted** (e.g. practice, open-book formats).

## Core use cases
| ID | Use case | Priority |
|----|----------|----------|
| UC1 | Live transcript of remote + local speakers | P0 |
| UC2 | Auto-detect questions addressed to the user and suggest an answer | P0 |
| UC3 | Manual "Answer" button and typed prompts | P0 |
| UC4 | Answers grounded in user-provided resume/docs/instructions | P0 |
| UC5 | Post-call notes: summary, questions asked, action items | P1 |
| UC6 | Practice mode against an AI counterpart | P1 |
| UC7 | Screen/code explanation via screenshot | P2 |
| UC8 | Multi-language (STT + answers) | P2 |

## Non-goals
- Evading detection or proctoring, hiding from other participants' tooling, or concealing use from an employer/school that prohibits it.
- Native-app call capture in v1 (Zoom desktop, Teams desktop). Browser-based calls only.
- Billing/auth/metering (deferred; hook points defined).

## Success metrics
| Metric | Target (initial, to validate) |
|--------|------------------------------|
| End-of-question to first answer token (p50 / p95) | < 1.5 s / < 3 s |
| Question-detection precision | > 90% |
| Question-detection recall | > 85% |
| STT word error rate on clean call audio | < 10% |
| Session crash/disconnect rate | < 1% |
| Answer faithfulness to user context (no fabricated experience) | > 95% on eval set |

Targets are assumptions, not measured values; set baselines in M1.

## Assumptions to validate early
1. Browser-based calls cover enough of the target market (ADR-0001).
2. 1.5 s first-token latency is achievable with chosen STT + model.
3. Users find suggested answers useful rather than distracting mid-call (needs user testing, not just metrics).
