# Voice Input/Output Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add voice-to-text input (mic button on every compose field) and voice-to-speech output (tap-to-play on Rex/Marcus replies), both backed by ElevenLabs, to the Forge app.

**Architecture:** Two new proxy endpoints on the existing `forge-manager-backend` Express server (`/api/transcribe`, `/api/speak`) forward audio/text to ElevenLabs and back, mirroring the existing proxy pattern used for exercise/book search. The frontend (`The-Forge/index.html`, a single-file vanilla-JS app) gets a reusable mic-button helper wired onto every compose field, and a play-button addition to the existing `addAgentMsg` chat-rendering helper.

**Tech Stack:** Node/Express backend (CommonJS, global `fetch`/`FormData`/`Blob`, Node 18+), `multer` (new dependency, memory storage only) for multipart upload parsing. Frontend: vanilla JS, `MediaRecorder` Web API, no new frontend dependencies.

## Global Constraints

- No audio is ever written to disk or a database anywhere in this app's own stack (multer must use `memoryStorage()`, never `diskStorage()`) — spec requirement, non-negotiable.
- `ELEVENLABS_API_KEY`, `ELEVENLABS_VOICE_REX`, `ELEVENLABS_VOICE_MARCUS` are already set as Railway env vars on `forge-manager-backend` (user confirmed) — code must read them via `process.env`, never hardcode a key or voice ID.
- Voice output is tap-to-play only — no auto-play on new AI replies.
- This repo (`The-Forge`) has no test runner; verification is done via `node -e` syntax checks (parsing script blocks with `new Function()`) and live browser testing via the `claude-in-chrome` tool, matching how every other feature in this codebase has been verified. The backend repo (`forge-manager-backend`) also has no test runner; verification is via direct `curl` calls to the deployed Railway URL after push.
- Follow existing code conventions: backend routes use try/catch with `res.json({error...})` or safe fallbacks (see existing `/api/exercises/search`, `/api/books/search`); frontend uses vanilla DOM APIs, no framework.

---

### Task 1: Backend — `/api/transcribe` endpoint (speech-to-text)

**Files:**
- Modify: `forge-manager-backend/package.json` (add `multer` dependency)
- Modify: `forge-manager-backend/server.js` (add route, add `multer` require + upload middleware)

**Interfaces:**
- Produces: `POST /api/transcribe` — accepts `multipart/form-data` with a single field `audio` (a Blob, any browser-recorded audio mimetype e.g. `audio/webm`). Returns `200 { text: string }` on success, or `200 { text: '', error: string }` on failure (never a hard 500, so the frontend can show a clean inline error instead of a thrown exception).

- [ ] **Step 1: Add `multer` dependency**

Run in `forge-manager-backend/`:
```bash
npm install multer
```

Verify `package.json` now has `"multer"` under `dependencies`.

- [ ] **Step 2: Add the require and upload middleware near the top of `server.js`**

Locate this existing block near the top of `forge-manager-backend/server.js`:
```js
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const { createClient } = require('@supabase/supabase-js');
const { runCoach } = require('./agent-coach');
const { runManager } = require('./agent-manager');
const { callClaude } = require('./claude');

const app = express();
app.use(cors());
app.use(express.json({ limit: '50mb' }));
```

Replace it with:
```js
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const multer = require('multer');
const { createClient } = require('@supabase/supabase-js');
const { runCoach } = require('./agent-coach');
const { runManager } = require('./agent-manager');
const { callClaude } = require('./claude');

const app = express();
app.use(cors());
app.use(express.json({ limit: '50mb' }));

// memoryStorage only — audio must never touch disk (privacy requirement)
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 15 * 1024 * 1024 } });
```

- [ ] **Step 3: Add the `/api/transcribe` route**

Add this route anywhere after the `app.get('/health', ...)` line and before the final `const PORT = ...` block at the end of `server.js`:

```js
app.post('/api/transcribe', upload.single('audio'), async (req, res) => {
  try {
    if (!req.file) return res.json({ text: '', error: 'No audio received' });
    const form = new FormData();
    form.append('model_id', 'scribe_v1');
    form.append('file', new Blob([req.file.buffer], { type: req.file.mimetype || 'audio/webm' }), 'audio.webm');
    const r = await fetch('https://api.elevenlabs.io/v1/speech-to-text', {
      method: 'POST',
      headers: { 'xi-api-key': process.env.ELEVENLABS_API_KEY },
      body: form
    });
    if (!r.ok) {
      const errBody = await r.text();
      console.error('transcribe error:', r.status, errBody);
      return res.json({ text: '', error: 'Transcription failed' });
    }
    const data = await r.json();
    res.json({ text: data.text || '' });
  } catch (e) {
    console.error('transcribe error:', e.message);
    res.json({ text: '', error: 'Transcription failed' });
  }
});
```

- [ ] **Step 4: Commit and push**

```bash
cd forge-manager-backend
git add package.json package-lock.json server.js
git commit -m "Add /api/transcribe endpoint for voice input (ElevenLabs Scribe)"
git push
```

- [ ] **Step 5: Verify live on Railway**

Wait ~30s for Railway to redeploy, then record a short voice memo on your phone/computer (any short audio file works, e.g. `.m4a`/`.webm`/`.mp3`), and run:
```bash
curl -X POST https://forge-manager-backend-production.up.railway.app/api/transcribe \
  -F "audio=@/path/to/your/test-audio-file"
```
Expected: `{"text":"<whatever you said in the recording>"}`. If you get `{"text":"","error":"Transcription failed"}`, check the Railway deploy logs for the `transcribe error:` line to see the ElevenLabs error detail (most likely cause: `ELEVENLABS_API_KEY` env var missing/typo'd, or an unsupported audio format).

---

### Task 2: Backend — `/api/speak` endpoint (text-to-speech)

**Files:**
- Modify: `forge-manager-backend/server.js`

**Interfaces:**
- Consumes: `process.env.ELEVENLABS_API_KEY`, `process.env.ELEVENLABS_VOICE_REX`, `process.env.ELEVENLABS_VOICE_MARCUS` (already set on Railway).
- Produces: `POST /api/speak` — accepts JSON `{ text: string, agent: 'rex' | 'marcus' }`. Returns raw `audio/mpeg` bytes with `200` status on success, or JSON `{ error: string }` with `200` status on failure (so the frontend can distinguish "got audio" from "got an error" purely by `Content-Type`).

- [ ] **Step 1: Add the `/api/speak` route**

Add this route right after the `/api/transcribe` route added in Task 1:

```js
app.post('/api/speak', async (req, res) => {
  try {
    const { text, agent } = req.body;
    if (!text) return res.json({ error: 'No text provided' });
    const voiceId = agent === 'marcus' ? process.env.ELEVENLABS_VOICE_MARCUS : process.env.ELEVENLABS_VOICE_REX;
    if (!voiceId) return res.json({ error: 'No voice configured for agent: ' + agent });
    const r = await fetch('https://api.elevenlabs.io/v1/text-to-speech/' + voiceId, {
      method: 'POST',
      headers: {
        'xi-api-key': process.env.ELEVENLABS_API_KEY,
        'content-type': 'application/json',
        'accept': 'audio/mpeg'
      },
      body: JSON.stringify({ text, model_id: 'eleven_monolingual_v1' })
    });
    if (!r.ok) {
      const errBody = await r.text();
      console.error('speak error:', r.status, errBody);
      return res.json({ error: 'Speech generation failed' });
    }
    const audioBuffer = Buffer.from(await r.arrayBuffer());
    res.set('Content-Type', 'audio/mpeg');
    res.send(audioBuffer);
  } catch (e) {
    console.error('speak error:', e.message);
    res.json({ error: 'Speech generation failed' });
  }
});
```

- [ ] **Step 2: Commit and push**

```bash
cd forge-manager-backend
git add server.js
git commit -m "Add /api/speak endpoint for voice output (ElevenLabs TTS)"
git push
```

- [ ] **Step 3: Verify live on Railway**

Wait ~30s for redeploy, then run:
```bash
curl -X POST https://forge-manager-backend-production.up.railway.app/api/speak \
  -H "content-type: application/json" \
  -d '{"text":"Hey, this is a test.","agent":"rex"}' \
  -o /tmp/test-rex-voice.mp3 -D -
```
Expected: response headers show `Content-Type: audio/mpeg`, and `/tmp/test-rex-voice.mp3` plays as speech in the Rex voice when opened. Repeat with `"agent":"marcus"` and confirm it's a different-sounding voice and saves to a separate file for comparison.

---

### Task 3: Frontend — reusable mic-button component, wired onto every compose field

**Files:**
- Modify: `The-Forge/index.html` (CSS block, new JS functions, new markup on 6 fields, wiring at bottom of script)

**Interfaces:**
- Produces: `function attachVoiceInput(micBtnEl, targetInputEl)` — call once per field to wire up recording/transcription/insertion. No return value; all state is scoped inside the function via closures per call.
- Consumes: `getBackend()` (existing helper returning the Railway backend base URL).

- [ ] **Step 1: Add CSS for the mic button**

Add this near the other small-button styles (e.g. right after the `.counter-btn` rule around line 159 in the current file — exact placement doesn't matter, any top-level `<style>` location works):

```css
.mic-btn { flex-shrink: 0; width: 32px; height: 32px; border-radius: 50%; border: 1px solid var(--border); background: var(--surface-2); color: var(--text); cursor: pointer; font-size: 14px; display: flex; align-items: center; justify-content: center; }
.mic-btn.recording { background: var(--bad); border-color: var(--bad); color: #fff; animation: mic-pulse 1s infinite; }
.mic-btn.mic-error { border-color: var(--bad); }
@keyframes mic-pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }
```

- [ ] **Step 2: Add the reusable `attachVoiceInput` function**

Add this function near `doExerciseSearch` (any top-level function position in the main `<script>` block works — this task adds it right before `function wireCustomAddHandlers(dd) {`):

```js
function attachVoiceInput(micBtnEl, targetInputEl) {
  let mediaRecorder = null;
  let chunks = [];
  micBtnEl.addEventListener('click', async () => {
    micBtnEl.classList.remove('mic-error');
    if (mediaRecorder && mediaRecorder.state === 'recording') {
      mediaRecorder.stop();
      return;
    }
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      chunks = [];
      mediaRecorder = new MediaRecorder(stream);
      mediaRecorder.addEventListener('dataavailable', e => { if (e.data.size > 0) chunks.push(e.data); });
      mediaRecorder.addEventListener('stop', async () => {
        stream.getTracks().forEach(t => t.stop());
        micBtnEl.classList.remove('recording');
        const blob = new Blob(chunks, { type: 'audio/webm' });
        const formData = new FormData();
        formData.append('audio', blob, 'recording.webm');
        micBtnEl.disabled = true;
        try {
          const res = await fetch(getBackend() + '/api/transcribe', { method: 'POST', body: formData });
          const data = await res.json();
          if (data.text) {
            targetInputEl.value = targetInputEl.value ? targetInputEl.value + ' ' + data.text : data.text;
            targetInputEl.dispatchEvent(new Event('input', { bubbles: true }));
          } else {
            micBtnEl.classList.add('mic-error');
          }
        } catch (e) {
          micBtnEl.classList.add('mic-error');
        }
        micBtnEl.disabled = false;
      });
      mediaRecorder.start();
      micBtnEl.classList.add('recording');
    } catch (e) {
      micBtnEl.classList.add('mic-error');
    }
  });
}
```

- [ ] **Step 3: Add a mic button next to the meal-logging field and wire it**

Find the meal input markup (search for `id="meal-input"` in `index.html`). Wrap it so a mic button sits alongside it — match whatever flex/layout wrapper is already there, adding:
```html
<button class="mic-btn" id="mic-meal" type="button">🎤</button>
```
as a sibling of the `#meal-input` element (inside its existing container div).

Then, near the bottom of the script where other one-off wiring happens (next to `document.getElementById('cal-prev-btn').addEventListener(...)`), add:
```js
attachVoiceInput(document.getElementById('mic-meal'), document.getElementById('meal-input'));
```

- [ ] **Step 4: Verify meal-input voice capture end-to-end in the browser**

Start a local server and open the app (same pattern used throughout this project):
```bash
cd /Users/cadenzoratti/The-Forge
python3 -m http.server 8934
```
Using the `claude-in-chrome` browser tool: navigate to `http://localhost:8934/index.html`, click the new mic button next to the meal input, grant mic permission when prompted, say a short test phrase, click the mic button again to stop. Confirm the meal input field fills with a reasonable transcription of what was said, and check console messages for errors (`read_console_messages` with `onlyErrors: true`). Stop the local server afterward (`pkill -f "http.server 8934"`).

- [ ] **Step 5: Wire the mic button onto the remaining 5 fields**

Repeat the same pattern (button markup + `attachVoiceInput` call) for:
- Rex chat input: element `id="coach-input"` (line ~1489 references it via `document.getElementById('coach-input')`) — add `<button class="mic-btn" id="mic-coach" type="button">🎤</button>` beside it, wire with `attachVoiceInput(document.getElementById('mic-coach'), document.getElementById('coach-input'));`
- Marcus chat input: element `id="manager-input"` (line ~1512) — button `id="mic-manager"`, wire `attachVoiceInput(document.getElementById('mic-manager'), document.getElementById('manager-input'));`
- Reminders "todo" add input: `<textarea id="todo-input" placeholder="e.g. dispute the card charge, order the book"></textarea>` followed by `<button class="add-meal-btn" id="add-todo-btn">Add</button>` (around line 469-470) — add `<button class="mic-btn" id="mic-todo" type="button">🎤</button>` beside the textarea, wire with `attachVoiceInput(document.getElementById('mic-todo'), document.getElementById('todo-input'));`
- Reminders "want" add input: `<textarea id="want-input" placeholder="e.g. try a new gym, read another book after this one"></textarea>` followed by `<button class="add-meal-btn" id="add-want-btn">Add</button>` (around line 475-476) — button `id="mic-want"`, wire `attachVoiceInput(document.getElementById('mic-want'), document.getElementById('want-input'));`
- Ideas add input: `<textarea id="idea-input" placeholder="Add something for later, e.g. yearly budget view"></textarea>` followed by `<button class="add-meal-btn" id="add-idea-btn">Add to backlog</button>` (around line 488-489) — button `id="mic-idea"`, wire `attachVoiceInput(document.getElementById('mic-idea'), document.getElementById('idea-input'));`
- Budget transaction note field: `<input type="text" id="tx-note" placeholder="Note (optional)" style="grid-column: span 2;">` (line 438) — button `id="mic-tx-note"`, wire `attachVoiceInput(document.getElementById('mic-tx-note'), document.getElementById('tx-note'));`

For each, follow the exact same two-part pattern as Step 3: add the `<button class="mic-btn" ...>🎤</button>` markup beside the field, then add one `attachVoiceInput(...)` call in the wiring section at the bottom of the script.

- [ ] **Step 6: Verify each of the 5 fields in the browser**

Using the same local-server + `claude-in-chrome` approach as Step 4, visit each tab (Coach, Manager, Reminders, Ideas, Budget) and confirm each new mic button appears next to its field, requests permission, records, and inserts a transcript on stop. Check console for errors after each. Clean up test data/localStorage afterward and stop the local server.

- [ ] **Step 7: Commit and push**

```bash
cd /Users/cadenzoratti/The-Forge
git add index.html
git commit -m "Add voice-to-text mic button to every compose field"
git push
```

---

### Task 4: Frontend — tap-to-play voice output on Rex/Marcus replies

**Files:**
- Modify: `The-Forge/index.html` (CSS, `addAgentMsg` function, `coach-send`/`manager-send` click handlers)

**Interfaces:**
- Consumes: `POST /api/speak` (Task 2), `getBackend()` (existing helper).
- Produces: modifies `addAgentMsg(container, role, text, icon)` to accept an optional 5th parameter `agent` (`'rex' | 'marcus' | null`); when provided and `role === 'assistant'`, a play button is rendered and wired.

- [ ] **Step 1: Add CSS for the play button**

Add next to the `.mic-btn` rules added in Task 3:
```css
.speak-btn { flex-shrink: 0; width: 22px; height: 22px; border-radius: 50%; border: 1px solid var(--border); background: none; color: var(--text-dim); cursor: pointer; font-size: 11px; display: flex; align-items: center; justify-content: center; margin-left: 6px; }
.speak-btn:hover { border-color: var(--ember); color: var(--ember); }
.speak-btn.loading { opacity: 0.5; cursor: default; }
```

- [ ] **Step 2: Extend `addAgentMsg` to accept an `agent` param and render a play button**

Locate the current `addAgentMsg` function:
```js
function addAgentMsg(container, role, text, icon) {
  const empty = container.querySelector('.empty');
  if (empty) empty.remove();
  const div = document.createElement('div');
  div.className = 'agent-msg ' + (role === 'user' ? 'user' : 'assistant');
  if (role === 'assistant' && icon) {
    const iconEl = document.createElement('span');
    iconEl.className = 'agent-icon';
    iconEl.textContent = icon;
    div.appendChild(iconEl);
  }
  const textEl = document.createElement('span');
  textEl.className = 'agent-text';
  textEl.textContent = text;
  div.appendChild(textEl);
  container.appendChild(div);
  container.scrollTop = container.scrollHeight;
  return div;
}
```

Replace it with:
```js
function addAgentMsg(container, role, text, icon, agent) {
  const empty = container.querySelector('.empty');
  if (empty) empty.remove();
  const div = document.createElement('div');
  div.className = 'agent-msg ' + (role === 'user' ? 'user' : 'assistant');
  if (role === 'assistant' && icon) {
    const iconEl = document.createElement('span');
    iconEl.className = 'agent-icon';
    iconEl.textContent = icon;
    div.appendChild(iconEl);
  }
  const textEl = document.createElement('span');
  textEl.className = 'agent-text';
  textEl.textContent = text;
  div.appendChild(textEl);
  if (role === 'assistant' && agent) {
    const speakBtn = document.createElement('button');
    speakBtn.className = 'speak-btn';
    speakBtn.type = 'button';
    speakBtn.textContent = '\u{1F50A}';
    let cachedAudioUrl = null;
    speakBtn.addEventListener('click', async () => {
      if (cachedAudioUrl) { new Audio(cachedAudioUrl).play(); return; }
      speakBtn.classList.add('loading');
      try {
        const res = await fetch(getBackend() + '/api/speak', {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify({ text: textEl.textContent, agent })
        });
        const contentType = res.headers.get('content-type') || '';
        if (contentType.includes('audio')) {
          const blob = await res.blob();
          cachedAudioUrl = URL.createObjectURL(blob);
          new Audio(cachedAudioUrl).play();
        }
      } catch (e) {}
      speakBtn.classList.remove('loading');
    });
    div.appendChild(speakBtn);
  }
  container.appendChild(div);
  container.scrollTop = container.scrollHeight;
  return div;
}
```

Note: the play button reads `textEl.textContent` at click time (not the `text` parameter captured at creation), so it correctly speaks the final reply even though the div is first created with placeholder text ("Thinking...") and updated later — see Step 3.

- [ ] **Step 3: Pass `agent` when creating the Rex and Marcus placeholder messages**

In the `coach-send` click handler, locate:
```js
  const thinking = addAgentMsg(container, 'assistant', 'Thinking...', 'Rex');
```
Change to:
```js
  const thinking = addAgentMsg(container, 'assistant', 'Thinking...', 'Rex', 'rex');
```

In the `manager-send` click handler, locate:
```js
  const thinking = addAgentMsg(container, 'assistant', 'Delegating to the team...', '💼');
```
Change to:
```js
  const thinking = addAgentMsg(container, 'assistant', 'Delegating to the team...', '💼', 'marcus');
```

Also, in `loadManagerHistory()` (which re-renders past Marcus messages when the Manager tab is reopened), locate:
```js
      } else if (m.role === 'manager') {
        addAgentMsg(container, 'assistant', m.content, '💼');
```
Change to:
```js
      } else if (m.role === 'manager') {
        addAgentMsg(container, 'assistant', m.content, '💼', 'marcus');
```
This gives past Marcus messages a play button too, not just the one just received in the current session. (There's no equivalent history-reload function on the Rex/Coach side in the current codebase, so no analogous change is needed there — Rex's play button only needed on the `coach-send` path from Step 3 above.)

- [ ] **Step 4: Verify in the browser**

Using the local-server + `claude-in-chrome` approach from Task 3, open the Coach (Rex) tab, send a short message, wait for the real reply to replace "Thinking...", then click the new speaker icon on that reply — confirm audio plays (check for a network request to `/api/speak` via `read_network_requests`, and check console for errors). Repeat on the Manager (Marcus) tab. Click the same speaker icon a second time on the same message and confirm no second network request fires (the cached-audio-URL path), just replays.

- [ ] **Step 5: Commit and push**

```bash
cd /Users/cadenzoratti/The-Forge
git add index.html
git commit -m "Add tap-to-play voice output on Rex/Marcus replies"
git push
```
