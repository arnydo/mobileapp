# Phone-only Index assistant — development plan

This branch develops a GrapheneOS-friendly, phone-only assistant from the existing Index voice implementation. The fork remains under upstream GPLv3 licensing; retain attribution and do not use Pebble trademarks as our app's name.

## Required experience

1. On a Pixel 8 Pro, an intentional double-tap on the back of the phone requests an assistant session. Provide a launcher and Quick Settings shortcut fallback.
2. When the device is locked, require **system keyguard authentication** before recording. Fingerprint is preferred if Android offers it, but respect mandatory PIN/password fallback and GrapheneOS restrictions. If already unlocked, do not prompt again.
3. Record through the **phone microphone** (no watch or Index 01 pairing). Show a clear recording indicator and explicit stop/cancel controls.
4. Reuse Index's existing recording, transcription, AI processing, notes, reminders, webhook and MCP pipelines. Never embed service credentials in the APK.
5. No root, Shizuku or Accessibility Service. No audio capture before successful unlock.
6. Validate on GrapheneOS, including app-in-background and locked-device scenarios. Android may suppress background activity launches and sensor events; failure must fall back to a direct user gesture/shortcut rather than weakening system security.

## Confirmed reuse points

- `experimental/src/commonMain/kotlin/coredevices/ring/ui/components/chat/IndexComposeBarHost.kt`: current phone-microphone recording, stop, store and `queueLocalAudioProcessing(fileId)`.
- `experimental/src/androidMain/kotlin/coredevices/ring/util/AudioRecorder.android.kt`: 16 kHz mono PCM voice-recognition recorder.
- `experimental/src/commonMain/kotlin/coredevices/ring/ui/screens/home/IndexFeedScreen.kt`: existing Index feed and voice compose bar.
- `experimental/src/androidMain/kotlin/coredevices/ring/glance/VoiceWidget.kt`: existing microphone widget with `voiceapp://listen` launch intent; confirm actual deep-link consumption before relying on it.
- `composeApp/src/commonMain/kotlin/coredevices/coreapp/ui/screens/OnboardingScreen.kt`: currently offers Watch, Index 01 and Both; add an explicit phone-only choice and ensure navigation never demands pairing.
- `composeApp/src/androidMain/kotlin/coredevices/coreapp/MainActivity.kt`: existing intent dispatch.
- `index-ai` and `mcp`: existing assistant and tool data paths.

## Implementation order

1. **Phone-only onboarding and navigation**: add a no-device option, keep Index enabled, and route the user to the Index feed. Revisit permissions so watch-only privileges are not requested in phone-only mode.
2. **Secure launch path**: add explicit assistant launch intent, authenticate via Android keyguard when locked, then show an assistant session. Test cold start and resume.
3. **Recording activation**: refactor existing `IndexComposeBarHost` capture logic into a reusable controller; prevent recording until authentication succeeds.
4. **Gesture detector**: add Pixel 8 Pro-specific accelerometer/gyro recognizer with optional enablement and a low-battery design; no dangerous privileges. Account for screen-off limitations.
5. **Security and reliability**: remove unneeded BLE/watch permissions and services from the phone-only build flavor, disable cleartext traffic, audit exported activities and logging, verify audio retention/backup and service credentials.
6. **CI, tests, APK**: host tests for lock-state routing and recording states, instrumented GrapheneOS tests for keyguard/background restrictions, then reproducible signed APK.

## Design cautions

- Pebble's README states online transcription and some integrations require configuration tokens unavailable in the public fork; verify usable providers and account provisioning rather than assuming upstream service access.
- The current Android manifest declares `android:usesCleartextTraffic="true"`. Remove this permission for a production phone-only build unless specifically required and justified.
- Existing `IndexComposeBarHost` requests audio permission while the view is visible; secure shortcut must authenticate before mounting/starting recording.
- An unlocked device does not automatically authorize high-impact actions: retain tool-specific user confirmations.
- Never allow lock-screen UI to disclose prior recordings or private notes before authentication.

## Scope of first PR

Start with a narrowly scoped phone-only onboarding change. Do not bundle it with an untested gesture sensor implementation, credential management or unrelated upstream refactors.
