# 07. Answer Generation

## Pipeline
1. Receive question (auto or manual) with `questionId`.
2. Assemble prompt: system rules + session instructions + user context + recent transcript + question.
3. Call LLM with streaming; forward deltas as `answer_delta`.
4. On completion, persist answer and emit `answer_end` with usage.

## Prompt structure
```
[system]  Role, honesty rules, style rules, output format        (static, cacheable)
[context] Resume / uploaded docs / session instructions          (static per session, cacheable)
[history] Last N transcript turns, labeled Remote / You          (dynamic)
[user]    Question (rewritten) + answer style flag               (dynamic)
```
Put static content first so provider prompt caching can reuse it; this cuts latency and cost on every question after the first. Check each provider's current caching rules and minimum sizes.

## System rules (core)
- Answer in first person as the user, using only facts present in the provided context. If a needed fact is missing, say so briefly and offer a generic framing instead of inventing experience, numbers, or employers.
- Lead with the answer; no preamble. Default 3-6 short sentences or tight bullets; expandable on request.
- For technical questions: state approach first, then detail; include complexity when relevant.
- Match the session language.

## Answer styles
| Style | Output |
|-------|--------|
| Concise | 2-4 sentences / key bullets, glanceable |
| Structured | Short framework (e.g. situation/action/result) for behavioral questions |
| Detailed | Full explanation, code where relevant |

## Model routing
Interface:
```ts
interface LlmProvider { stream(req: LlmRequest, signal: AbortSignal): AsyncIterable<LlmChunk>; }
```
- Default: a fast, low-latency tier model chosen for time-to-first-token.
- Escalate to a larger model for coding/system-design questions or on user toggle.
- Model list per provider is configuration, not code; names and prices change often.
- Fallback: on provider error or > 2.5 s without first token, retry once on the fallback provider.

## Streaming and cancellation
Each question owns an `AbortController`. A new question, `pause`, or `end` aborts the active stream; the UI marks the prior answer as superseded rather than deleting it.

## Guardrails
- Context injection: uploaded docs and transcripts are untrusted text. Wrap in delimiters, state in the system prompt that they are data, never instructions.
- Output length caps (max tokens per style).
- No answers triggered from the `local` channel.
- Log prompts only if the user opts in (privacy, doc 10).

## Coding / screenshot support (P2)
Use `chrome.tabs.captureVisibleTab` on user click, send image to a vision-capable model with the detected question. Show a visible indicator when a screenshot is taken. Treat screenshot content as sensitive (may include other people's data); do not persist by default.

## Quality evaluation
Faithfulness (does it stay within context?), relevance, brevity, latency. See doc 11 for the eval harness and rubric.
