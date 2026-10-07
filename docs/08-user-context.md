# 08. User Context

## Context types
| Type | Source | Use |
|------|--------|-----|
| Resume/CV | PDF/DOCX upload | Grounds first-person answers |
| Documents | PDF/DOCX/TXT/MD (playbooks, briefs, notes) | Reference facts |
| Job description / agenda | Pasted text | Frames what matters |
| Session instructions | Free text field | Tone, constraints, "always mention X" |

## Ingestion
1. Upload to object storage (private bucket, per-user prefix, encrypted at rest).
2. Extract text (PDF text extraction; OCR fallback for scanned files; DOCX parser).
3. Normalize: strip headers/footers, collapse whitespace, keep section headings.
4. Store extracted text in Postgres with status `PROCESSING → READY | FAILED`.
5. Enforce limits: file size (e.g. 10 MB), pages, total tokens per session.

## Strategy: context stuffing first, retrieval later
- v1: if total context fits a token budget (e.g. ~8-20k tokens), include it verbatim in the cached prefix. Simple, deterministic, best latency with prompt caching.
- v2: when context exceeds the budget, chunk by section (400-800 tokens), embed, store in pgvector, retrieve top-k per question. Adds latency and a retrieval-quality failure mode; do this only when needed.

Documented trade-off: RAG adds an embedding call and a vector query to every answer's critical path (tens to hundreds of ms) and can miss relevant chunks; stuffing wastes tokens on large contexts. Start with stuffing.

## Session configuration
```ts
type SessionConfig = {
  mode: 'meeting' | 'sales' | 'interview-practice' | 'practice';
  language: string;
  model: string;
  answerStyle: 'concise' | 'structured' | 'detailed';
  detection: 'auto' | 'suggest' | 'manual';
  contextIds: string[];
  instructions?: string;
  saveTranscript: boolean;
};
```

## Keyword boosting
Extract proper nouns and technical terms from context (names, companies, stack) and pass to STT as boost keywords. Cap list size per provider limits.

## Privacy
Contexts may contain PII (resumes). Per-user isolation, deletion endpoint that removes file, text, chunks, embeddings, and cached derived data. No cross-user reuse.
