# ADR-0001: Chrome extension first, desktop app later

Status: Proposed

## Context
Two ways to capture call audio: (a) a Chrome extension using `tabCapture` for browser-based calls; (b) a desktop app (Electron or native) using OS system-audio loopback, which also covers native Zoom/Teams clients.

## Decision
Ship the extension first; keep the protocol and gateway client-agnostic so a desktop client can reuse them.

## Consequences
Positive: fast to build and distribute, no install friction, easy side-panel UI, shared TypeScript codebase.
Negative: no native-app call capture; dependent on Chrome extension policies and MV3 constraints; desktop users on native clients are excluded until the desktop client exists.

## Alternatives considered
| Option | Why not first |
|--------|---------------|
| Electron/desktop first | Larger build (per-OS audio loopback, signing, updates) before validating demand |
| Native mobile | Audio capture of other apps is heavily restricted |
| Bot joins the meeting | Visible participant, platform-specific SDKs and approvals; viable later for a transparent enterprise product |

## Revisit when
Browser-call coverage proves insufficient in user research, or Web Store policy risk materializes.
