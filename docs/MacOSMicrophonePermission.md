# macOS: Microphone permission (no audio is captured)

## Symptom

On macOS the client shows the list of audio input devices normally, but no
audio is ever captured — the audio bar in the Audio Wizard stays flat and other
players can't hear you. Often the system permission prompt never appears.

## Why this happens

Enumerating audio devices does **not** require any permission on macOS, but
*capturing* from them does. Two independent mechanisms are involved:

1. **TCC (privacy) permission** — the per-app "Microphone" toggle in
   *System Settings → Privacy & Security → Microphone*. The app requests it at
   runtime via `AVCaptureDevice requestAccessForMediaType:` (see
   `src/mumble/CoreAudio.mm`) and must declare `NSMicrophoneUsageDescription`
   in its `Info.plist` (see `src/mumble/mumble.plist.in`).

2. **The code-signing entitlement** `com.apple.security.device.audio-input`
   (see `src/mumble/mumble.entitlements.plist`). When the binary is signed
   with the **hardened runtime** (`codesign --options runtime`, mandatory for
   notarization), macOS *silently* returns no audio from capture devices
   unless this entitlement is part of the signature — no prompt, no error,
   just silence.

The build now embeds the entitlement in every signing path:

- CMake ad-hoc signs `Mumble.app` with the entitlements after building
  (`src/mumble/CMakeLists.txt`), so local and CI builds carry it.
- `macx/scripts/osxdist.py` and `scripts/sign_macOS.py` apply the entitlements
  when producing/signing release DMGs.

## Troubleshooting checklist

1. Check *System Settings → Privacy & Security → Microphone* — the app must be
   listed and enabled. If it's listed and disabled, enable it and restart the
   app.
2. If the app is not listed at all, reset its TCC record and relaunch so the
   prompt is shown again:

   ```sh
   tccutil reset Microphone mumble-client
   ```

   (`mumble-client` is the `CFBundleIdentifier` from `Info.plist`.)
3. Verify the entitlement is in the signature of the installed app:

   ```sh
   codesign -d --entitlements - /Applications/Mumble.app
   ```

   The output must contain `com.apple.security.device.audio-input`.
4. Launch the app from Finder, not from a terminal. When launched from a
   terminal, macOS attributes the microphone request to the terminal app
   instead of the client.
5. Note for self-built/ad-hoc builds: every rebuild changes the ad-hoc code
   signature, so macOS may ask for microphone permission again after each
   rebuild. This is expected; stable Developer ID signing avoids it.
