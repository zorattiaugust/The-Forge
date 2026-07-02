# Live (Chunked) Voice Input Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Forge's voice-input mic button append transcribed text progressively (in ~3-second bursts) while recording, instead of only after the user taps the mic a second time to stop.

**Architecture:** Rewrite `attachVoiceInput` in `index.html` to keep the microphone stream open for the whole recording session, but rotate through short-lived `MediaRecorder` instances every ~3 seconds — each one produces a small, independently-decodable audio clip that's uploaded to the already-deployed `POST /api/transcribe` endpoint as soon as it's ready, with the returned text appended to the target field. Recording stops either via a second mic tap, or (for the Rex/Marcus chat fields only) via Enter-to-send, which now waits for the final segment's transcript before sending.

**Tech Stack:** Vanilla JS in `index.html` (no build step, no framework, no test runner) — `MediaRecorder`, `getUserMedia`, `fetch`. No backend changes.

## Global Constraints

- No new dependencies, no build tooling introduced — this is a single-file vanilla JS change, matching the existing codebase.
- `/api/transcribe` (already deployed) is used exactly as-is: `POST` with `FormData` field `audio` (a blob), JSON response `{ text }` on success.
- No automated test framework exists in this repo and none should be introduced for this change — verification is manual, in a real browser, exactly as the design spec's testing plan calls for (chunk-boundary/timing behavior can only be meaningfully judged on a real device anyway).
- Every step that touches `index.html` must be manually verified in a browser before moving to the next step — there is no automated test suite to fall back on.

---

### Task 1: Rewrite `attachVoiceInput` for chunked/segmented recording

**Files:**
- Modify: `index.html:1857-1900` (replace the entire `attachVoiceInput` function)

**Interfaces:**
- Consumes: nothing new — same `getBackend()` helper and `/api/transcribe` endpoint already used today.
- Produces: `micBtnEl._forgeStopRecording` — a zero-argument function, attached to the mic button element passed into `attachVoiceInput`, callable by other code (Task 2). Calling it: if not currently recording, returns `Promise.resolve()` immediately. If recording, stops the current segment, waits for its transcription to be appended to the target field, stops the microphone stream, and returns a `Promise<void>` that resolves once all of that is done.

- [x] **Step 1: Read the current function to confirm exact current line range**

Run: `grep -n "^function attachVoiceInput" -A 45 index.html | head -50`

Expected: shows the existing function starting at line 1857 and ending with a closing `}` before line 1901 (a blank line). Confirm the exact end line before editing — if the codebase has changed since this plan was written, adjust the line range in the next step accordingly.

- [x] **Step 2: Replace the function body**

Replace the entire existing `attachVoiceInput` function (from `function attachVoiceInput(micBtnEl, targetInputEl) {` through its closing `}`) with:

```javascript
function attachVoiceInput(micBtnEl, targetInputEl) {
  if (micBtnEl.dataset.voiceWired) return;
  micBtnEl.dataset.voiceWired = '1';
  const SEGMENT_MS = 3000;
  let stream = null;
  let currentSegment = null;
  let rotateTimer = null;
  let isRecording = false;

  function appendText(text) {
    targetInputEl.value = targetInputEl.value ? targetInputEl.value + ' ' + text : text;
    targetInputEl.dispatchEvent(new Event('input', { bubbles: true }));
  }

  function uploadSegment(blob) {
    if (blob.size === 0) return Promise.resolve();
    const formData = new FormData();
    formData.append('audio', blob, 'recording.webm');
    return fetch(getBackend() + '/api/transcribe', { method: 'POST', body: formData })
      .then(res => res.json())
      .then(data => {
        if (data.text) appendText(data.text);
        else micBtnEl.classList.add('mic-error');
      })
      .catch(() => { micBtnEl.classList.add('mic-error'); });
  }

  function beginSegment() {
    const chunks = [];
    const recorder = new MediaRecorder(stream);
    const completed = new Promise(resolve => {
      recorder.addEventListener('stop', () => {
        const blob = new Blob(chunks, { type: 'audio/webm' });
        uploadSegment(blob).then(resolve);
      });
    });
    recorder.addEventListener('dataavailable', e => { if (e.data.size > 0) chunks.push(e.data); });
    recorder.start();
    return { recorder, completed };
  }

  function rotateSegment() {
    const previous = currentSegment;
    currentSegment = beginSegment();
    previous.recorder.stop();
  }

  function stopRecording() {
    if (!isRecording) return Promise.resolve();
    isRecording = false;
    micBtnEl.classList.remove('recording');
    clearInterval(rotateTimer);
    const finalSegment = currentSegment;
    finalSegment.recorder.stop();
    return finalSegment.completed.then(() => {
      stream.getTracks().forEach(t => t.stop());
      stream = null;
    });
  }

  micBtnEl._forgeStopRecording = stopRecording;

  micBtnEl.addEventListener('click', async () => {
    micBtnEl.classList.remove('mic-error');
    if (isRecording) {
      stopRecording();
      return;
    }
    try {
      stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      isRecording = true;
      micBtnEl.classList.add('recording');
      currentSegment = beginSegment();
      rotateTimer = setInterval(rotateSegment, SEGMENT_MS);
    } catch (e) {
      micBtnEl.classList.add('mic-error');
    }
  });
}
```

Note: `rotateSegment` starts the next `MediaRecorder` on the same open `stream` *before* stopping the previous one, so there is no gap where audio isn't being captured — a `MediaStream`'s tracks can be read by more than one `MediaRecorder` at a time, so this overlap is safe.

- [x] **Step 3: Manual verification in a desktop browser**

Open `index.html` in Chrome (serve it with `python3 -m http.server` from the repo root and open `http://localhost:<port>/index.html` — opening via `file://` can block microphone access in some browsers). Go to a tab with a mic button (e.g. Reminders → todo input). Click the mic button, grant permission, and speak a full sentence continuously for at least 10 seconds without stopping.

Expected: the input field fills in with text in a few separate bursts roughly every 3 seconds, not all at once at the end. Click the mic button again to stop; expected: one final burst of text appears for whatever was said in the last partial segment, and the mic button's `recording` (pulsing) visual state clears.

- [x] **Step 4: Commit**

```bash
git add index.html
git commit -m "Rewrite voice input as chunked/segmented recording for live transcription"
```

---

### Task 2: Flush the final segment before Enter-to-send on Rex/Marcus chat inputs

**Files:**
- Modify: `index.html:1609-1617` (the `coach-input`/`manager-input` Enter-keydown handler)

**Interfaces:**
- Consumes: `micBtnEl._forgeStopRecording` from Task 1 (a zero-arg function returning `Promise<void>`), looked up via `document.getElementById('mic-coach')` / `document.getElementById('mic-manager')`.
- Produces: nothing new for other tasks — this is a leaf change.

- [x] **Step 1: Replace the existing Enter-keydown handler**

Replace:

```javascript
[document.getElementById('coach-input'), document.getElementById('manager-input')].forEach(el => {
  el.addEventListener('keydown', e => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      const btnId = el.id === 'coach-input' ? 'coach-send' : 'manager-send';
      document.getElementById(btnId).click();
    }
  });
});
```

with:

```javascript
[document.getElementById('coach-input'), document.getElementById('manager-input')].forEach(el => {
  el.addEventListener('keydown', async e => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      const micId = el.id === 'coach-input' ? 'mic-coach' : 'mic-manager';
      const micBtn = document.getElementById(micId);
      if (micBtn && micBtn._forgeStopRecording) {
        await micBtn._forgeStopRecording();
      }
      const btnId = el.id === 'coach-input' ? 'coach-send' : 'manager-send';
      document.getElementById(btnId).click();
    }
  });
});
```

- [x] **Step 2: Manual verification in a desktop browser**

With the same local server running from Task 1, go to the Rex (Coach) tab. Click the mic button next to the chat input, speak a short sentence, and — while still recording (before tapping the mic to stop) — press Enter.

Expected: recording stops (mic button's pulsing state clears), the last bit of speech appears in the input field, and then the message sends with the complete transcribed text (not missing the final words). Repeat once for the Marcus (Manager) tab.

- [x] **Step 3: Commit**

```bash
git add index.html
git commit -m "Flush final voice segment before Enter-to-send on Rex/Marcus chat inputs"
```

---

### Task 3: Deploy and verify on a real device

**Files:** none (deployment + verification only)

**Interfaces:** none — this task only exercises what Tasks 1 and 2 built.

- [x] **Step 1: Push to origin/main**

```bash
git push origin main
```

Expected output includes a line like `<old-sha>..<new-sha>  main -> main`. This triggers GitHub Pages to redeploy the updated `index.html` automatically (same as the last few voice-feature deploys) — no further action needed for the frontend to go live.

- [x] **Step 2: Confirm the live page has the update**

Run: `curl -s https://zorattiaugust.github.io/The-Forge/ | grep -c "_forgeStopRecording"`

Expected: a number greater than `0` (confirms the new code is present in the deployed page). GitHub Pages deploys can take a minute or two — if the count is `0`, wait ~30 seconds and retry rather than assuming failure.

- [ ] **Step 3: Test on the real iPhone**

On the phone, fully close and reopen Forge (to avoid any stale Safari page cache), then:
1. Tap a mic button and speak a full sentence continuously — confirm text appears in a few progressive bursts rather than all at once.
2. Tap the mic again mid-recording to stop — confirm the last partial segment's text still appears.
3. On the Rex or Marcus tab, tap the mic, speak, and press the on-screen return/send key before manually stopping — confirm the message sends with the complete transcribed text.
4. Judge how noticeable/annoying the ~3-second chunk boundaries are in real use (per the design spec's accepted limitation) — this is the real-world signal for whether this is good enough or worth revisiting toward true streaming later. No code change required based on this judgment as part of this plan; report back what was observed.
