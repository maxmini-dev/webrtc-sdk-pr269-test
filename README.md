# webrtc-sdk PR #269 test artifacts

Test artifacts for [webrtc-sdk/webrtc#269](https://github.com/webrtc-sdk/webrtc/pull/269) — Android ADM `setStopRecordingOnMute(boolean)` opt-out.

## What's here

- **APK** (release assets): `sample-app-compose-pr269-debug.apk` — LiveKit's Compose sample app built against a custom WebRTC AAR containing the PR #269 changes, with a runtime **"Stop recording on mute (PR #269)"** switch on the connect screen. Default is OFF, so the app exercises the new opt-out path by default; toggle ON to compare against the old stop-on-mute behavior.
- **Prefixed AAR** (release assets): `libwebrtc-prefixed.aar` — drop-in replacement for `io.github.webrtc-sdk:android-prefixed:144.7559.09`, built from PR #269 commit `2ce4de3c52` with the standard `android_prefixed` pipeline (patches from `webrtc-sdk/webrtc-build` + `com.github.johnrengelman.shadow` relocation `org.*` → `livekit.org.*` + `.so` renamed to `liblkjingle_peerconnection_so.so`).
- **Patch** (in repo): `pr269-test-toggle.patch` — the diff applied to `livekit/client-sdk-android` to consume a local AAR and add the runtime toggle.

## How to test

### Sideload the APK

```bash
adb install sample-app-compose-pr269-debug.apk
```

Launch the app, enter your LiveKit URL and JWT, leave "Stop recording on mute (PR #269)" **OFF** (default), Connect. On connect the app logs:

```
adb logcat | grep -iE "PR269|WebRtcAudioRecord|MicrophoneMute|AudioRecord"
```

Look for `PR269: JavaAudioDeviceModule.setStopRecordingOnMute(false)` at connect time, then toggle mic mute in the call UI. With the opt-out **enabled** (switch OFF), you should observe:
- `AudioRecord` transitions to `RECORDING` state once at connect and **stays** there across mute toggles
- No `WebRtcAudioRecord: stopThread` / restart on each toggle
- WebRTC's `AudioDeviceModule::SetMicrophoneMute(true)` invoked and reflected downstream (audio zeroed inside the engine, capture kept alive)

Toggle the switch ON to reproduce the pre-PR behavior for comparison.

Debug LiveKit server (if you don't have Cloud):
```bash
brew install livekit livekit-cli
livekit-server --dev --bind 0.0.0.0    # api key: devkey, secret: secret
lk token create --api-key devkey --api-secret secret --join --room mute-test --identity user1 --valid-for 24h
```
For emulator use `ws://10.0.2.2:7880`; for a physical device on the same LAN use `ws://<your-mac-lan-ip>:7880`.

### Reproduce the build

Apply the patch to a matching `livekit/client-sdk-android` checkout, drop `libwebrtc-prefixed.aar` at `livekit-android-sdk/libs/libwebrtc.aar`, then build.

```bash
git clone --recursive https://github.com/livekit/client-sdk-android.git
cd client-sdk-android
git am /path/to/pr269-test-toggle.patch
mkdir -p livekit-android-sdk/libs
cp /path/to/libwebrtc-prefixed.aar livekit-android-sdk/libs/libwebrtc.aar
JAVA_HOME=/opt/homebrew/opt/openjdk@17 ./gradlew :sample-app-compose:assembleDebug
```

### Rebuild the AAR from scratch

See `webrtc-sdk/webrtc-build` for the canonical pipeline. Non-obvious gotchas encountered when running arm64 Linux (Colima on Apple Silicon):

- `depot_tools` requires Python 3.11+ (uses `enum.StrEnum`) → Ubuntu 24.04, not 22.04
- Chromium's `third_party/llvm-build/.../clang` is x86_64-only; on arm64 hosts install qemu-user-static and `libc6:amd64` via multiarch so JNI codegen can invoke it. Target compilation uses the Android NDK's arm64-native clang and is fast.
- Skip `install-build-deps.py --android` on arm64 (adds `i386` which 404s on ports.ubuntu.com)
- Pin `httplib2==0.20.4` (last version with `httplib2.socks` submodule that depot_tools' `gerrit_util.py` imports)
- Apply patches `add_license_dav1d`, `android_webrtc_version`, `fix_mocks`, `jni_prefix` from `webrtc-sdk/webrtc-build/build/patches`
- `jni_prefix.patch` needs one hand-fix in `sdk/android/BUILD.gn` for m144_release (add `package_prefix = android_package_prefix` to `generated_peerconnection_jni` and `generated_java_audio_jni` blocks — they moved between `main` and `m144_release`)
- Create `sdk/android/api/org/webrtc/WebrtcBuildVersion.java` (referenced by `android_webrtc_version.patch`)
- After `build_aar.py`, post-process with Gradle Shadow `relocate 'org', 'livekit.org'` on `classes.jar` and rename `libjingle_peerconnection_so.so` → `liblkjingle_peerconnection_so.so` inside `jni/*/`

## What the patch changes

| File | Purpose |
|---|---|
| `settings.gradle` | Add `flatDir { dirs livekit-android-sdk/libs }` (needed because `FAIL_ON_PROJECT_REPOS` is set) |
| `livekit-android-sdk/build.gradle` | Swap `api libs.webrtc` → `api(name: 'libwebrtc', ext: 'aar')` |
| `sample-app-common/build.gradle` | Disable LeakCanary 2.8.1 (calls `ViewModelProvider.Factory.create(KClass, CreationExtras)` — requires androidx.lifecycle 2.9+ that this checkout doesn't pull in, causing `AbstractMethodError` on `CallActivity.onCreate`) |
| `sample-app-common/.../MainViewModel.kt` | Persist the toggle via SharedPreferences (default OFF) |
| `sample-app-common/.../CallViewModel.kt` | Pass `stopRecordingOnMute` into `AudioOptions.javaAudioDeviceModuleCustomizer` |
| `sample-app-compose/.../CallActivity.kt` | Plumb the flag through `BundleArgs` |
| `sample-app-compose/.../MainActivity.kt` | Add the connect-screen `Switch` |

## License

- WebRTC portions: BSD 3-clause (see webrtc-sdk/webrtc)
- LiveKit portions: Apache 2.0 (see livekit/client-sdk-android)
