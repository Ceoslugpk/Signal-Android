# Signal Android - API 18 Port Analysis

## Feasibility Verdict

**Likely Impossible.**

While it might theoretically be possible to aggressively strip features and mock out massive portions of the modern Android SDK, achieving a *functional client on the live Signal network* with this fork on API 18 is practically impossible.

**Reasoning:**
1. **Native Protocol Dependencies:** `libsignal-client` is a Rust-based library precompiled for Android. The prebuilt AAR (`0.91.0`) targets a specific NDK and Android API level (usually much higher than 18, often 21 or 23 minimum for modern Rust/NDK). Since we cannot downgrade `libsignal` because it would break compatibility with the live network, compiling or running this required cryptography library on API 18 would fail due to missing native libc APIs (`__getrandom`, `getauxval`, modern `std::atomic` implementations) and NDK toolchain limitations.
2. **Dependency Graph Collapse:** Signal relies entirely on modern AndroidX libraries (CameraX, Compose, Navigation, WorkManager, Media3) and Kotlin Coroutines. These libraries have minimum API requirements of 19 (for legacy versions) or, more commonly, 21-24+ (Compose requires API 21, Media3 requires API 21). There is no cohesive dependency graph that resolves to API 18 while supporting the core features (messaging/media) using the current architectural patterns.
3. **Core Subsystem Rewrites:** Almost every major subsystem (background execution, crypto storage, camera, UI) relies on APIs introduced well after API 18 (e.g., Keystore API 23+, WorkManager API 19+, Camera2 API 21+). Porting would require writing parallel implementations using long-deprecated APIs (like `AlarmManager` with manual wake locks, legacy `Camera` API, software crypto providers) and ripping out Jetpack Compose entirely.

## 1. minSdk Definitions

The primary source of truth for the minimum SDK version is in the Gradle version catalog, with references spread across various module build files.

*   `gradle/libs.versions.toml`: `minSdk = "23"` (Line 9)
*   `app/build.gradle.kts`: `minSdk = libs.versions.minSdk.get().toInt()` (Line 209)
*   `microbenchmark/build.gradle.kts`: `minSdk = libs.versions.minSdk.get().toInt()`
*   `benchmark/build.gradle.kts`: `minSdk = 23`
*   `build-logic/plugins/src/main/java/signal-sample-app.gradle.kts`: `minSdk = libs.versions.minSdk.get().toInt()`
*   `build-logic/plugins/src/main/java/signal-library.gradle.kts`: `minSdk = libs.versions.minSdk.get().toInt()`
*   `demo/camera/build.gradle.kts`: `minSdk = 23`
*   `demo/video/build.gradle.kts`: `minSdk = 26`
*   `demo/registration/build.gradle.kts`: `minSdk = 26`
*   `baseline-profile/build.gradle.kts`: `minSdk = 28`
*   Multiple instances in `app/lint-baseline.xml` indicating previous configurations expecting API 21+.

## 2. Dependency Graph & Conflicts

Attempting to resolve a dependency graph for API 18 presents insurmountable conflicts.

**Native Blockers:**
*   **`libsignal-client` (`org.signal:libsignal-client:0.91.0`)**:
    *   *Conflict:* Prebuilt native libraries (Rust/NDK) generally assume at least Android 5.0 (API 21) or 6.0 (API 23) in modern toolchains to utilize features like 64-bit support (arm64-v8a requires API 21+) and modern libc functions.
    *   *Resolution:* None without potentially forking and heavily patching the Rust codebase and an ancient NDK, which is highly unstable and risky for cryptographic code.
*   **`signal-ringrtc` (`org.signal:ringrtc-android:2.67.2`)**:
    *   *Conflict:* Requires modern WebRTC implementations and hardware codecs. We will strip this feature as per the requirements.

**AndroidX & Kotlin Conflicts:**
*   **Jetpack Compose (`androidx.compose.*`)**:
    *   *Conflict:* Compose requires a minimum of API 21.
    *   *Resolution:* Compose cannot be used on API 18. The entire UI layer must be rewritten using standard Android Views (XML).
*   **AndroidX Media3 (`androidx.media3.*`)**:
    *   *Conflict:* Requires API 21+.
    *   *Resolution:* Must revert to legacy `MediaPlayer` or an ancient version of ExoPlayer (v2.x).
*   **CameraX (`androidx.camera.*`)**:
    *   *Conflict:* Requires API 21+.
    *   *Resolution:* Must use the deprecated `android.hardware.Camera` API.
*   **WorkManager (`androidx.work.*` - implicitly used/required by architecture)**:
    *   *Conflict:* Requires API 19+ for modern implementations, older versions existed but are highly incompatible with modern Kotlin/Coroutines.
*   **Kotlin & Coroutines**:
    *   *Conflict:* Modern Kotlin standard libraries and coroutines heavily optimize for newer Dalvik/ART runtimes. While theoretically compilable, running complex coroutine flows on Dalvik (API 18) often hits multidex limits, method count issues, and severe performance bottlenecks.

*Conclusion:* A cohesive, resolvable dependency graph for API 18 does not exist for this codebase without replacing roughly 80% of the libraries.


## 3. Subsystem Analysis & API Usage

Because the app is fundamentally built around modern Android architectures, API 18 violations are systemic rather than localized.

| Subsystem | Severity | Scope | Key Files/APIs | Analysis |
| :--- | :--- | :--- | :--- | :--- |
| **Keystore / Crypto Storage** | **Blocker** | Pervasive | `KeyStoreHelper.java`, `AndroidKeyStoreProvider` | Android Keystore backed by hardware requires API 23+. API 18 Keystore is highly vulnerable and lacks required ciphers (AES/GCM). Requires a complete fallback to a custom password-based key derivation (PBKDF2) and software storage (SQLCipher), weakening security drastically. |
| **UI / Jetpack Compose** | **Needs Rewrite** | Pervasive | `app/src/main/java/**/*.kt` | Compose requires API 21. Every Compose screen (which is most of the modern app) must be rewritten in XML/Legacy Views. |
| **Camera & Media Capture** | **Needs Rewrite** | Large | `demo/camera/*`, `WebRtcVideoUtil.java` | CameraX requires API 21+. Must write a custom camera controller using the deprecated `android.hardware.Camera`. |
| **Background Jobs** | **Needs Rewrite** | Large | `jobs/*` (WorkManager) | Doze mode (API 23+) and JobScheduler (API 21+) are absent. Must rely on `AlarmManager` and explicit `WakeLock` management, leading to severe battery drain. |
| **Notifications** | **Needs Compat** | Medium | `NotificationChannels.java`, `MessageNotifier.java` | Notification Channels require API 26. Must use `NotificationCompat` but gracefully handle the lack of channels, Direct Reply (API 24), and modern styling. |
| **Permissions** | **Needs Compat** | Small | `Permissions.java` | Runtime permissions were introduced in API 23. On API 18, permissions are granted at install time. Code must be bypassed or stubbed to assume granted for API 18. |

### Feature Matrix (Keep / Strip / Uncertain)

| Feature | Action | Notes |
| :--- | :--- | :--- |
| 1:1 Text Messaging | **Keep** | Core requirement. |
| Group Text Messaging | **Keep** | Core requirement. |
| Attachments (Image/Video) | **Keep** | Requires rewriting camera/media pickers for API 18. |
| Registration / Linking | **Keep** | Requires rewriting Keystore logic for API 18. |
| RingRTC (Voice/Video) | **Strip** | WebRTC/Hardware codecs are incompatible or highly unstable on API 18. |
| Payments / MobileCoin | **Strip** | Likely relies on modern crypto/Keystore APIs. |
| Stories | **Strip** | Heavy media/Compose reliance; not a core requirement. |
| Calling Links / Meet | **Strip** | Tied to RingRTC. |


## 4. Proposed Order of Work

Given the immense scope, if this port were to be attempted, the following order of work (from easiest to hardest) is proposed:

1. **Gradle and Build Script Adjustments (Easiest)**
   - Modify `gradle/libs.versions.toml` and all `build.gradle.kts` files to set `minSdk = 18`.
   - Remove/disable all modules and plugins that strictly require API 21+ and are not core (e.g., `benchmark`, `macrobenchmark`, `baseline-profile`).

2. **Strip Non-Core Features (Easy to Medium)**
   - Delete all code related to RingRTC (Voice/Video calling), Payments (MobileCoin), and Stories.
   - Remove these dependencies from `libs.versions.toml`.
   - Stub out UI entry points for these features.

3. **Downgrade AndroidX Dependencies (Medium)**
   - Roll back AndroidX libraries to their lowest possible versions that support API 18 (e.g., `androidx.appcompat:appcompat:1.3.1`).
   - Note: This will likely break compatibility with Kotlin 2.x and Coroutines 1.9.0, requiring a massive downgrade of the entire Kotlin toolchain.

4. **Rewrite Camera and Media Layers (Hard)**
   - Remove `androidx.camera` (CameraX) and `androidx.media3` (Media3/ExoPlayer).
   - Write a custom `Camera1Controller` using `android.hardware.Camera` for taking pictures and recording video.
   - Revert media playback to the legacy Android `MediaPlayer` API.

5. **Rewrite Background Execution (Hard)**
   - Strip `androidx.work` (WorkManager).
   - Implement a custom job scheduling system using `AlarmManager` and explicit `WakeLock` handling for background sync and message delivery.

6. **Rewrite Jetpack Compose UI (Very Hard)**
   - Jetpack Compose does not support API 18.
   - Every single screen written in Compose (`@Composable` functions) must be completely rewritten using traditional Android XML layouts, Activities, and Fragments. This represents thousands of hours of UI work.

7. **Rewrite Cryptographic Storage / Keystore (Extremely Hard/Blocker)**
   - API 18 lacks secure hardware-backed Keystore capabilities (introduced in API 23+).
   - The entire `KeyStoreHelper` and identity key management system must be rewritten to use an alternate, less secure method (like a user-provided passphrase to derive an AES key via PBKDF2 to unlock SQLCipher).

8. **Solve `libsignal-client` Native Compilation (Near Impossible)**
   - The prebuilt AAR for `libsignal-client` is compiled for modern NDKs (usually API 21+).
   - Must fork the `libsignal` Rust repository.
   - Attempt to configure an ancient Android NDK (e.g., NDK r10e) to compile the modern Rust codebase for 32-bit ARM (armeabi-v7a) targeting API 18.
   - This will likely fail due to modern Rust standard library requirements (e.g., missing `getrandom` syscalls in old Android libc).

---
*Appendix: Lint Analysis*
A full Lint report could not be generated because the Gradle build fails during the configuration phase when `minSdk` is lowered to 18. The build system rejects the modern dependency graph (specifically Compose and KSP plugins) before static analysis can even begin. See `docs/lint-api18.txt` for the raw configuration failure log.
