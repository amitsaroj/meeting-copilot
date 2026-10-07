# 10. Security & Privacy

## Principles
Transparent by design, minimal data, user control. The product processes other people's voices in real time, which carries legal and ethical obligations.

## Consent and transparency
- **Recording/transcription consent:** many jurisdictions require notifying or getting consent from other participants (all-party consent laws exist in parts of the US, EU GDPR applies to voice data). Provide a one-click "post a disclosure message" helper and show a reminder at session start. Responsibility remains with the user; document this in Terms.
- **Prohibited-use stance:** the product does not include features that hide its use from other participants or from platforms, and does not target proctored/closed-book assessments. Terms prohibit use where AI assistance is disallowed.
- **Employer/organization blocklist:** support a domain-level opt-out so organizations can disable the extension on their meeting domains (policy-driven, not evasion).

## Threat model (summary)
| Asset | Threat | Mitigation |
|-------|--------|------------|
| STT/LLM API keys | Extraction from extension | Keys server-side only; extension holds short-lived WS token |
| Audio/transcripts | Interception, leaks | TLS 1.2+, WSS only; encrypt at rest; per-user isolation |
| User documents (resumes) | Cross-tenant access | Per-user prefixes, row-level checks on every query, signed URLs short TTL |
| Prompt injection via transcript/docs | Model manipulated by spoken or uploaded text | Delimit untrusted text, system-prompt rule that it is data, output length caps, no tool access in v1 |
| Abuse / cost attacks | Token/audio flooding | Per-user rate limits, session caps, concurrency limits, backpressure |
| Malicious web pages | Messaging into extension | Strict `externally_connectable` (none), validate all `runtime` messages, no `eval`, tight CSP |
| Supply chain | Compromised dependency | Lockfile, `pnpm audit`, minimal deps, pinned CI actions |

## Extension hardening
- Minimal permissions; no `<all_urls>` host access. Use `activeTab` + user gesture.
- Content-security policy without `unsafe-eval`; no remotely hosted code (required by MV3).
- Never log audio or transcripts to console/analytics.

## Data handling
| Data | Default retention | Notes |
|------|-------------------|-------|
| Raw audio | Not stored | Streamed through, discarded |
| Transcript | Stored only if `saveTranscript` on; user-deletable | Auto-expire (e.g. 30 days) configurable |
| Answers/notes | Stored with session | Deleted with session |
| Contexts (resume/docs) | Until user deletes | Deletion cascades to chunks/embeddings |
| Prompts/LLM logs | Off by default | Opt-in for debugging, redacted |

Third-party processors (STT, LLM, hosting) must be disclosed in the privacy policy; choose providers with no-training-on-customer-data terms and, where required, zero-retention options. Verify each provider's current terms.

## Compliance checklist (to validate with counsel)
- GDPR: lawful basis, DPA with processors, data-subject deletion/export, DPIA likely warranted (voice data).
- CCPA/CPRA disclosures.
- Chrome Web Store: privacy disclosures, limited-use compliance, single-purpose description. Review current developer program policies before submission.
- Terms of service of call platforms (Meet/Teams/Zoom) regarding automated capture.

## Incident readiness
Secrets rotation runbook, audit log of admin access, breach-notification procedure.
