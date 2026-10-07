# 03. Extension Audio Capture (MV3)

## Permissions
`tabCapture`, `offscreen`, `sidePanel`, `activeTab`, `storage`. Host permissions only for the gateway origin. Request mic permission via a one-time visible extension page (offscreen documents cannot show permission prompts).

## Flow
1. User gesture (click on action / side panel Start) is required for `tabCapture`.
2. Service worker: `chrome.tabCapture.getMediaStreamId({ targetTabId })`.
3. Service worker ensures an offscreen document exists (`chrome.offscreen.createDocument`, reason `USER_MEDIA`), sends it the `streamId`.
4. Offscreen document:
   - `getUserMedia({ audio: { mandatory: { chromeMediaSource: 'tab', chromeMediaSourceId: streamId } } })` for remote audio.
   - `getUserMedia({ audio: { echoCancellation: true, noiseSuppression: true } })` for mic.
5. Both streams → `AudioContext` → `AudioWorkletNode` → PCM16 frames.

## Important behavior: tab muting
Capturing a tab's audio stops it playing to the user. Route the tab stream back to `audioContext.destination` so the user still hears the call.

## Echo and double-capture
If the user is on speakers, the mic picks up the remote voice, producing duplicate transcript lines. Mitigations: recommend headphones in onboarding; enable `echoCancellation`; drop `local` finals that closely match a `remote` final within ~1.5 s (text-similarity check in the gateway).

## Audio pipeline
- Resample to 16 kHz mono in the AudioWorklet (not on the main thread).
- Encode Int16 little-endian; batch 20-100 ms per WebSocket message.
- Message header: `{ channel: 'remote' | 'local', seq, tsMs }` + binary payload.
- Backpressure: if `ws.bufferedAmount` exceeds a threshold (~256 KB), drop oldest `local` frames first and emit a warning event; never block the audio thread.

## Lifecycle and failure handling
| Event | Behavior |
|-------|----------|
| Tab closed / navigated away | Stop capture, notify panel, offer restart |
| Mic permission denied | Continue remote-only, show persistent banner |
| WebSocket drop | Reconnect with exponential backoff + jitter, resume with `sessionId` + last `seq`; buffer up to ~10 s of audio locally |
| Offscreen doc killed | Service worker detects via heartbeat, recreates, requests user gesture if `streamId` expired |

## Limits to document for users
- Works only for calls running in a Chrome tab (Meet, web Teams, web Zoom). Native desktop call apps are not captured (see ADR-0001).
- Other browser tabs' audio is not captured, but anything playing in the captured tab is.
- Only one active capture per tab.
