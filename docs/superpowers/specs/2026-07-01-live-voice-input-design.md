# Live (chunked) voice input for Forge

## Problem

The voice input feature shipped in `2026-07-01-voice-input-output-design.md` is tap-to-start, tap-to-stop, transcribe-after-stop: nothing appears in the field until the user explicitly stops recording. The user wants the field to fill in as they talk, closer to live dictation.

## Why chunked polling over true real-time streaming

Two approaches considered:

1. **True real-time streaming** (word-by-word, near-instant) — requires a persistent streaming connection (e.g. websocket) to a real-time speech-to-text API, plus a relay in the backend. More new infrastructure, more to get wrong, and speech providers' real-time APIs are typically a different (and sometimes costlier/less available) tier than batch transcription.
2. **Chunked polling** (chosen) — re-run the existing, already-deployed `/api/transcribe` endpoint every ~3 seconds against short recorded segments, appending each result as it returns. Noticeably not instant (a few seconds of lag per burst of text), but reuses infrastructure that's already built, tested, and live — no new backend work at all.

Chosen: chunked polling. The lag is an accepted tradeoff for a personal-use app; if it turns out to feel too laggy or the transcription quality at chunk boundaries is bad in practice, that's grounds to revisit toward true streaming later — this spec doesn't need to get that right upfront.

## How segments are captured

A single continuous `MediaRecorder` with `timeslice` doesn't work reliably for this: most recorded formats put header/setup data only in the first emitted chunk, so later chunks in isolation often aren't independently decodable audio files.

Instead: every ~3 seconds, the current `MediaRecorder` instance is stopped and a new one is immediately started on the same already-open microphone `MediaStream` (no repeated permission prompts — the stream itself stays open for the whole recording session). Each stop produces one small, complete, independently-decodable audio clip, which is uploaded to `/api/transcribe` immediately, exactly as a single full recording is today.

## Flow

1. User taps the mic button. Recording starts (same pulsing "recording" visual state as today).
2. Every ~3 seconds, the in-progress segment is closed off and a new one starts recording immediately, with no gap the user would notice.
3. Each closed-off segment uploads to `/api/transcribe` as soon as it's ready. When a result comes back, its text is appended to the target field using the same appending logic already in place (`existing text + ' ' + new text`, or just the new text if the field was empty).
4. Recording ends when either:
   - The user taps the mic button again, or
   - The user presses Enter to send the message while a recording is active.

   In both cases: stop the current `MediaRecorder` segment, wait for its transcription to return and append to the field, *then* (if triggered by Enter) proceed with the send action — using the now-fully-updated field value. This avoids text appearing after a message has already been sent.
5. If any single segment's transcription request fails, that segment's text is simply skipped — the same brief `mic-error` state already used today shows, but the still-running recording is not interrupted.

## Accepted limitations (v1)

- **Boundary accuracy:** words spoken right at a ~3-second cut point may occasionally be mistranscribed, split, or dropped. This is the core tradeoff of chunking vs. true streaming.
- **Ordering:** if one segment's network request happens to resolve slower than a later segment's, its text could append slightly out of order. Not worth building request-sequencing logic for v1 — acceptable for a personal-use app.
- **No new UI:** the mic button's recording/error visual states are unchanged; the only visible difference is that text now appears in a few progressive bursts instead of all at once after stopping.

## What's needed before implementation

Nothing new — this only changes frontend logic (`attachVoiceInput` in `index.html`) and reuses the already-deployed `/api/transcribe` backend endpoint as-is.

## Testing plan

On a real device (the chunk-boundary and timing behavior can't be meaningfully judged any other way):
1. Speak a full sentence continuously and confirm text fills in progressively in a few bursts rather than all at once at the end.
2. Stop mid-sentence via the mic button and confirm the last partial segment still gets transcribed and appended.
3. Start recording, speak, and press Enter to send before manually stopping — confirm the message sends with the fully transcribed text, not a truncated version.
4. Judge in practice how bad chunk-boundary word-splitting is, and decide from there whether this is good enough or worth revisiting toward true streaming.

## Out of scope for this pass

- True real-time/streaming transcription
- Request-sequencing to guarantee strict append ordering
- Any change to voice output (tap-to-play) or to fields where voice input isn't already wired up
