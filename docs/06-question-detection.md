# 06. Question Detection

Goal: decide when a remote-speaker utterance warrants an answer suggestion, fast and with few false triggers. This stage largely determines whether the product feels useful or noisy.

## Inputs
Final utterances from the `remote` channel (the `local` channel is context only), plus the last N turns.

## Stage 0: utterance assembly
Speakers often pause mid-question. Merge consecutive remote finals when the gap is under ~1.2 s and the previous text lacks terminal punctuation or ends with a conjunction/filler. Emit a candidate after the merge window closes.

## Stage 1: heuristics (fast path, < 20 ms)
Positive signals:
- Ends with `?`
- Starts with interrogatives: who, what, when, where, why, how, which, can/could/would/do/does/did/is/are
- Imperative prompts: "walk me through", "tell me about", "explain", "describe", "give me an example"
- Long-form prompts: "I'd like you to ...", "what I want to understand is ..."

Negative signals: very short (< 4 words) backchannel ("okay", "right", "mm-hm"), rhetorical or self-talk, pure statements with no ask.

Outcomes: `QUESTION` (high confidence), `NOT_QUESTION`, or `AMBIGUOUS`.

## Stage 2: classifier (ambiguous only)
Small fast LLM call with last 3-5 turns, structured JSON output:
```json
{ "is_question_for_user": true, "confidence": 0.0, "rewritten_question": "..." }
```
Timeout 400 ms; on timeout default to `NOT_QUESTION` in auto mode (user can still click Answer). `rewritten_question` resolves pronouns and context ("and why that one?" → full question) for the answer prompt.

## Dedup and cooldown
- Suppress a candidate whose normalized text has > 0.85 similarity to one fired in the last 20 s.
- If a new question arrives while an answer streams, cancel the old stream (doc 07).
- Optional per-session cooldown (default 2 s) to avoid flicker.

## Manual override
The Answer button always works: uses the last N remote utterances (default 3) or typed text.

## Modes
| Mode | Behavior |
|------|----------|
| Auto | Detect and answer automatically |
| Suggest | Detect and show the question; answer on click (lower cost, less noise) |
| Manual | No detection |

## Evaluation
Build a labeled set of transcript windows (question / not question / ambiguous) from real and synthetic calls. Track precision and recall separately: false positives waste tokens and distract; false negatives are recoverable through the manual button. Weight toward precision. Details in doc 11.
