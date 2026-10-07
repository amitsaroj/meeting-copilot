# 04. Streaming STT

## Provider strategy
Primary: Deepgram streaming (low-latency, endpointing). Fallback: AssemblyAI streaming. Behind a `SttProvider` interface so providers can be swapped or A/B tested. Verify current model names, pricing, and parameters against provider docs at implementation time; they change.

```ts
interface SttProvider {
  open(opts: { language: string; sampleRate: 16000; keywords?: string[] }): SttStream;
}
interface SttStream {
  write(pcm16: Buffer): void;
  on(event: 'interim' | 'final' | 'utteranceEnd' | 'error' | 'close', cb: (e: any) => void): void;
  close(): void;
}
```

## Streams
One upstream stream per channel (`remote`, `local`) per session. Keep-alive messages during silence to avoid idle timeouts.

## Settings to tune (Deepgram-style)
- `encoding=linear16`, `sample_rate=16000`, `channels=1`
- `interim_results=true` for live display
- Endpointing / utterance-end thresholds: start around 300-1000 ms, tune using eval set; shorter = faster answers but more split questions
- `smart_format=true`, punctuation on (needed by detector)
- Keyword boosting from user context (company names, technologies, jargon)

## Transcript model
```ts
type Utterance = {
  id: string; sessionId: string;
  channel: 'remote' | 'local';
  text: string; isFinal: boolean;
  startMs: number; endMs: number;
  confidence?: number;
};
```
Interims are ephemeral (UI only). Only finals are stored and fed to the detector.

## Language
Single language per session in v1 (chosen at session creation). Mid-call language switch requires closing and reopening upstream streams with a transcript marker. Multilingual auto-detect is P2.

## Failure handling
- Upstream disconnect: reopen once immediately, then backoff; replay buffered audio if gap < 5 s, otherwise mark a transcript gap.
- Repeated provider errors: fail over to fallback provider for the rest of the session and emit a `provider_changed` event.

## Quality risks
- Domain jargon and accents drive WER; build a small labeled audio set early (see doc 11).
- Cross-talk on the tab channel merges speakers; acceptable in v1, flag in UI as "remote speakers".
