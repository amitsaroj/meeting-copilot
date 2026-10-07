# 09. Side-Panel UI

Built with React in the Chrome Side Panel. The panel is a normal part of the browser window; assume it can be visible if the user shares the whole window or screen. The UI should say so plainly in onboarding rather than imply otherwise.

## Screens
1. **Onboarding**: permissions (tab audio, mic), headphone recommendation, consent notice explanation, supported-call limits.
2. **Session setup**: mode, language, model, answer style, detection mode, context selection, instructions.
3. **Live session**: transcript, answer card, controls.
4. **Post-call**: notes, transcript, delete/export.
5. **Settings**: defaults, data retention, delete-all.

## Live session layout
```
┌ Status bar: ● Listening · 12:03 · Remote/Local levels ┐
│ Answer card (streaming, copy button, regenerate)      │
│ Question chip: "Walk me through your last project"    │
│ ─────────────────────────────────────────────────── │
│ Transcript (auto-scroll, Remote vs You, interim dim)  │
│ ─────────────────────────────────────────────────── │
│ [Answer] [Pause] [Clear] [Style ▾] [End]  [type…][⏎] │
└───────────────────────────────────────────────────────┘
```

## States
| State | UI |
|-------|----|
| Connecting / reconnecting | Banner with retry countdown; audio buffered |
| Mic denied | Persistent banner, remote-only mode |
| STT degraded | Warning chip, provider failover notice |
| Answer streaming | Skeleton → tokens; Stop button |
| Answer superseded | Prior answer collapsed under history |
| Idle > N min | Prompt to end session (cost control) |

## Interaction details
- Keyboard shortcuts: answer, pause, copy answer, toggle style (registered via `chrome.commands`; user-configurable).
- Answers rendered as markdown with code highlighting; copy buttons per block.
- History list of Q/A pairs for the session; click to restore.
- Font size and compact mode; light/dark following system.

## Accessibility
Focus management on new answer, `aria-live="polite"` for streaming text (throttled), sufficient contrast, full keyboard operability.

## Performance
Virtualize the transcript list; batch token deltas per animation frame; avoid re-rendering the whole transcript on each interim.

## State management
Single store (Zustand) fed by one WebSocket event reducer; the offscreen document relays events to the panel via `chrome.runtime` ports (panel does not open its own socket).
