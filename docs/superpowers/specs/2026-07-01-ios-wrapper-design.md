# iOS wrapper for Forge (Capacitor shell)

## Problem

Forge currently exists only as a single `index.html` opened in a desktop/mobile browser. The goal is to ship Forge as a real iOS app distributed through the App Store (via TestFlight first). This spec covers only getting the existing web app running inside a native iOS shell — not push notifications (separate follow-on spec), not Android, not App Store listing/submission metadata.

## Why Capacitor

Considered three approaches:

1. **Capacitor** (chosen) — actively maintained, generates a real Xcode project the user owns and can inspect/modify directly, handles native permission plumbing (mic access, etc.) through config rather than hand-written native code, and has an official `@capacitor/push-notifications` plugin that the follow-on push-notifications spec will build on.
2. **Cordova** — the older predecessor to Capacitor. Still functional but maintenance and plugin ecosystem have shifted to Capacitor; no advantage over it today.
3. **Raw Xcode project with a hand-built WKWebView** — full control, zero JS tooling dependency, but means manually writing permission handling, asset bundling, and (later) push notification wiring that Capacitor provides out of the box. Rejected: more manual work for no real benefit, since the user is already comfortable with Node/npm (used on the `forge-manager-backend` Railway service).

Node.js/npm is a new tooling dependency for this repo (previously just a raw `index.html` with no build step), but this was explicitly confirmed as acceptable given it's already used on the backend.

## Repo layout

A new `mobile/` directory at the repo root holds the entire Capacitor project:

```
mobile/
  package.json
  capacitor.config.json
  www/              (gitignored - build artifact, not source)
  ios/              (generated Xcode project - committed to git, per Capacitor convention)
```

Root `index.html` remains the single source of truth for the web app — the existing dev/test workflow (open the file, or serve it locally) is unchanged.

`mobile/www/` is a **build artifact**, not a second copy of the source. A `sync` script in `mobile/package.json` does:

```
cp ../index.html www/index.html && npx cap sync ios
```

This copies the current root `index.html` into the Capacitor web root and syncs it into the generated iOS project. `mobile/www/` is gitignored so there's never a stale duplicate checked into version control.

## App identity

- **Name:** Forge
- **Bundle ID:** `com.zoratti.forge`

## Native permissions

- `NSMicrophoneUsageDescription` added to the generated `Info.plist`, required for `getUserMedia`/`MediaRecorder` (voice input) to work inside WKWebView — same requirement as Safari on the web.

## Icon and launch screen

Default/placeholder Capacitor-generated icon and launch screen for this pass. These can be swapped for real assets at any time without touching any of the wrapper logic — not a blocker for getting a build running.

## Backend

No changes. The bundled `index.html` continues to call the same Railway-hosted `forge-manager-backend` API it does today (`getBackend()` returns the same fixed URL regardless of whether the page is loaded in a browser tab or inside the WKWebView shell).

## Testing plan

1. Build and run in the iOS Simulator via Xcode — confirm the UI loads and basic navigation works (mic permission prompts can't be fully tested in Simulator, since it has no real microphone hardware by default on most Mac setups).
2. Build and run on a real device via the existing Apple Developer account — confirm mic permission prompt appears, voice input (`/api/transcribe`) and voice output (`/api/speak`) both work end-to-end over a real network connection.
3. Distribute via TestFlight for further real-device testing before any App Store submission (submission itself is out of scope for this spec).

## Out of scope for this pass

- Push notifications (separate spec, to follow once this shell exists)
- Android
- App Store listing content, screenshots, submission/review process
- Real app icon / launch screen artwork
- Over-the-air update tooling (e.g. Capacitor Live Updates) for shipping UI changes without an App Store review cycle
