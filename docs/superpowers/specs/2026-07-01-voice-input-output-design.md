# Voice input/output for Forge

## Problem

Typing into Forge — chat messages to Rex/Marcus, food logs, reminders, ideas — is friction, especially on mobile. The user wants to speak instead of type (voice input), and wants Rex/Marcus replies to be readable *and* speakable back (voice output), using ElevenLabs voices.

## Scope

Two independent sub-features, sharing one vendor (ElevenLabs) and one backend (`forge-manager-backend`):

1. **Voice input** — a reusable mic button on every compose-style text field in the app.
2. **Voice output** — a tap-to-play button on Rex/Marcus chat messages that speaks the reply aloud in an agent-specific voice.

They ship as two separate implementation slices (input first is a natural default given dependency order, but either can go first) — this doc covers both since they were designed together and share infrastructure decisions.

## Why ElevenLabs for both directions

Originally considered the browser's built-in Web Speech API for voice input (free, no backend work) vs. a server-side transcription API. Rejected the browser API because the user's stated goal is eventually shipping Forge as an iOS App Store app (PWA or WebView-wrapped) — Web Speech API support in that context (home-screen PWA / WKWebView) is unreliable in Safari/iOS, while `MediaRecorder` (audio capture) is solid there. Since audio must be captured and sent to a server either way, and ElevenLabs offers both a transcription API (Scribe) and a TTS API, using ElevenLabs for both means one vendor, one API key, instead of mixing providers.

## Voice input

**Where:** Rex chat input, Marcus chat input, food logging box, reminders (todo/want add inputs), ideas add input, budget transaction note field. Any other free-text field can get the same treatment later using the same reusable piece.

**Flow:**
1. User taps the mic button next to a field.
2. Browser requests mic permission (once; browser remembers after that).
3. Recording starts via `MediaRecorder`; the mic button shows a recording state (e.g. pulsing/red).
4. User taps again to stop (no silence-based auto-stop for v1 — keeps behavior predictable and simple).
5. The recorded audio blob uploads to a new backend endpoint, `POST /api/transcribe`, as multipart form data.
6. The backend forwards the audio directly to ElevenLabs' Scribe (speech-to-text) API and returns `{ text }`.
7. The frontend inserts the returned text into the field that was recording (appending if the field already had text, matching how a user would expect dictation to add to what's there).
8. If permission is denied, recording fails, or the transcription call errors, the mic button shows a brief inline error state and the field is left untouched — no silent failures.

**Data handling (privacy requirement):** The audio blob exists only in browser memory during recording and upload — never written to `localStorage`, IndexedDB, or any persistent client storage. The backend passes the audio straight through to ElevenLabs in the request and does not write it to disk, a database, or any storage bucket; once the transcript is returned to the frontend, the audio is discarded. Only the resulting text persists (as whatever message/log entry it becomes — same as if the user had typed it). Caveat: ElevenLabs' own handling of audio on their end is governed by their API terms, not something this app controls.

## Voice output (Rex & Marcus only)

**Where:** Each AI message bubble in the Coach (Rex) and Manager (Marcus) tabs gets a small play icon.

**Flow:**
1. User taps play on a specific AI reply.
2. Frontend calls `POST /api/speak` with `{ text, agent: 'rex' | 'marcus' }`.
3. Backend calls ElevenLabs' TTS API using the voice ID configured for that agent, returns the generated audio.
4. Frontend plays the audio via a standard `<audio>` element; the play icon shows a loading/playing state.
5. The generated audio is cached client-side per message (in memory, for the current session) so replaying the same message doesn't re-call the API and re-incur cost.

**Trigger:** Tap-to-play only — no auto-play on new replies. This avoids surprise audio and avoids generating (and paying for) speech for replies the user has no interest in hearing.

**Voices:** Rex and Marcus each get their own distinct ElevenLabs voice ID, matching their separate personalities.

## What's needed before implementation

- An ElevenLabs account and API key (added to Railway env vars on `forge-manager-backend`, same pattern as the existing `ANTHROPIC_API_KEY`).
- Two ElevenLabs voice IDs, picked by the user from ElevenLabs' voice library — one for Rex, one for Marcus.

## Out of scope for this pass

- Auto-play of AI replies.
- Silence-based auto-stop for recording.
- Voice input on read-only/search-only fields (book search, exercise search) — can be added later using the same reusable mic component if wanted.
- Any transcript/audio history or logging beyond the resulting text.
