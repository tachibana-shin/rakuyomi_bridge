# Rakuyomi Bridge — AI Agent Guide

## Project Overview

Kotlin Multiplatform + Android companion app for the RakuYomi KOReader plugin.
Loads `librakuyomi_server.so` (Rust cdylib) via JNI and runs the rakuyomi HTTP
server as a foreground service, exposing it at `http://127.0.0.1:8787`.

Two Android modules ship from this repo:

- **`androidApp/`** — full Compose UI (Material 3, Hilt, DataStore, navigation,
  built-in update flow, browser, logs). minSdk 21.
- **`headless/`** — minimal Android-only app with a single programmatic
  `LinearLayout` screen (no Compose, no Hilt, no DataStore). minSdk 18
  (Android 4.3). Different `applicationId` so it can be installed alongside
  the Compose app; deep links are namespaced
  (`rakuyomi_bridge://start|stop`).

Includes a CLI test harness (`bridge/cli/`) for Linux that starts/stops the
server binary and runs API tests.

## Repository Structure

```
bridge/
├── build.gradle.kts                    # Root: plugin declarations (apply false)
├── settings.gradle.kts                 # Multi-module config
├── gradle/libs.versions.toml           # Version catalog
├── shared/                             # KMP shared module (android + jvm, minSdk 18)
│   ├── build.gradle.kts
│   └── src/
│       ├── commonMain/.../             # ServerConfig, BridgeClient, BridgeResponse
│       ├── androidMain/.../            # JniBridge, RakuyomiServer
│       └── jvmMain/.../                # Platform.actual (HttpURLConnection)
├── androidApp/                         # Compose + Hilt + DataStore (minSdk 21)
│   ├── build.gradle.kts
│   ├── proguard-rules.pro
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── kotlin/.../
│       │   ├── MainActivity.kt
│       │   ├── RakuyomiTheme.kt
│       │   ├── ServerService.kt
│       │   └── BootReceiver.kt
│       └── res/
│           ├── values/strings.xml
│           └── xml/network_security_config.xml
├── androidHeadless/                           # Minimal Android app (minSdk 18, no Compose)
│   ├── build.gradle.kts
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── kotlin/.../headless/
│       │   ├── HeadlessApp.kt
│       │   ├── service/{ServerService,NetworkBridgeWorker,ServiceLauncherActivity}.kt
│       │   ├── receiver/BootReceiver.kt
│       │   ├── settings/SettingsStore.kt
│       │   └── ui/MainActivity.kt      # LinearLayout built in code
│       └── res/
│           ├── values/strings.xml + vi/ja/zh
│           ├── mipmap*/ic_launcher*.xml
│           └── xml/network_security_config.xml
└── cli/                                # JVM CLI test harness (Linux)
    ├── build.gradle.kts
    └── src/main/kotlin/.../BridgeCLI.kt
```

## Build System

- AGP 8.4.0, Kotlin 1.9.22, Gradle 8.6
- JVM target: 1.8, minSdk 18 (shared/headless) or 21 (androidApp), targetSdk 34
- KMP targets: `androidTarget` (shared) + `jvm` (shared + cli)
- UI: Jetpack Compose (Material3) via BOM 2024.02.00 in `:androidApp`;
  programmatic `LinearLayout` in `:androidHeadless` to keep the APK small and
  compatible with API 18
- CLI: `application` plugin, `java.net.HttpURLConnection` (no extra HTTP lib)
- **Build order**: Rust `.so` must be built first (via `scripts/build-rust-android.sh`),
  then the Gradle build creates APKs containing the native libraries.
- The `.so` is built via `scripts/build-rust-android.sh` from the rakuyomi monorepo.
  The headless module re-uses `androidApp/src/main/jniLibs/` via
  `sourceSets["main"].jniLibs` to avoid duplicating the 50MB native payload.
- Rust toolchain: 1.97.1 (pinned) with Android targets (`aarch64-linux-android`,
  `armv7-linux-androideabi`, `x86_64-linux-android`). Requires `cargo-ndk`.

## CI/CD

Two GitHub Actions workflows drive CI:

### `android-ci.yml` (Continuous Integration)

Triggered on push to `main`/`develop` and PR to `main`. Also callable
via `workflow_call` from the release workflow.

Pipeline:
1. **Checkout bridge** (at workspace root)
2. **Clone RakuYomi** (full `tachibana-shin/rakuyomi` repo to `rakuyomi/`)
3. **Install Rust** (dtolnay/rust-toolchain@stable, 1.95.0, Android targets)
4. **Install cargo-ndk** + system deps (fontconfig, freetype)
5. **Build Rust Android libraries** (`rakuyomi/scripts/build-rust-android.sh`,
   với `BRIDGE_DIR=${{ github.workspace }}`, dev mode — only
   `aarch64-linux-android` for speed)
   - Produces `librakuyomi_server.so` into `androidApp/src/main/jniLibs/`
     and `androidHeadless/src/main/jniLibs/`
6. **Set up JDK 21** (Temurin, Gradle cache)
7. **Lint + Test + Assemble debug APKs** for both `:androidApp` and
   `:androidHeadless`, sign and upload as workflow artifacts

Cargo registry/target is cached via `actions/cache@v4`.

### `android-release.yml` (Release)

Triggered on push to `main`.

- **`ci` job**: reuses `android-ci.yml` (validates the push)
- **`release` job** (needs `ci`): builds Rust for all **3 Android targets**
  (arm64-v8a, armeabi-v7a, x86_64) + Gradle release builds (AAB + APK),
  signs, renames headless artifacts, then runs `semantic-release` to publish.

### Directory layout on CI runner

```
$GITHUB_WORKSPACE/      (bridge checked out here)
├── androidApp/
├── androidHeadless/
├── gradlew
├── .releaserc.json
└── rakuyomi/            (cloned from tachibana-shin/rakuyomi)
    └── scripts/build-rust-android.sh
```

Script `build-rust-android.sh` mặc định resolve `BRIDGE_DIR` = `rakuyomi/../bridge`,
nhưng bridge ở ngay `$GITHUB_WORKSPACE` chứ không phải `$GITHUB_WORKSPACE/bridge`.
Do đó CI pass biến môi trường `BRIDGE_DIR: ${{ github.workspace }}` để ghi đè,
giúp script copy `.so` trực tiếp vào đúng `androidApp/src/main/jniLibs/`.

## KMP Targets

| Source Set | Target | Key Files |
|---|---|---|
| `commonMain` | all | ServerConfig, BridgeClient (expect Platform), BridgeResponse |
| `androidMain` | Android | JniBridge, RakuyomiServer |
| `jvmMain` | JVM | Platform.actual (HttpURLConnection) |

The `expect object Platform` / `actual object Platform` pattern provides
HTTP get/post on both JVM and future native targets.

## CLI Usage (Linux)

```sh
./gradlew :cli:run --args="health 8787"
./gradlew :cli:run --args="run ./server /tmp/data"
./gradlew :cli:run --args="exec ./server /tmp/data -- curl http://127.0.0.1:8787/library"
./gradlew :cli:run --args="test ./server /tmp/data"
```

## Key Architecture

```
KOReader (Lua) ──TCP──► Rust server ──► SQLite + WASM sources
                           ▲
Android:  JniBridge ←───╯  (libserver.so via System.loadLibrary)
Linux:    HttpURLConnection  (standalone server binary, systemd)
```

## Permissions (Android)

| API | Permission | UX |
|---|---|---|
| 30+ | `MANAGE_EXTERNAL_STORAGE` | Compose AlertDialog → Settings intent |
| 33+ | `POST_NOTIFICATIONS` | Compose AlertDialog → system dialog |
| 34+ | `FOREGROUND_SERVICE_DATA_SYNC` | Manifest only, maxSdkVersion="34" |

## Important Rules

- **Compose only** (in `:androidApp`) — no XML layouts there. The `:androidHeadless`
  module is exempt: it uses programmatic `LinearLayout` because Compose
  requires API 21+ and the headless build must run on API 18.
- Use `ComponentActivity` + `setContent` for the Compose app; the headless
  app uses plain `Activity` to avoid pulling in `androidx.activity` ≥ 1.7
  (which requires API 19+).
- Use `expect`/`actual` for platform-specific code in shared module
- `jvmMain` must not depend on Android APIs
- `androidMain` must not depend on JVM APIs (java.net, etc.)
- All user-facing strings go into `strings.xml`
- No emojis in code or comments

## RakuYomi Integration Points

- Server port: `8787`
- Server host: `127.0.0.1`
- Data directory: `/storage/emulated/0/koreader/rakuyomi` (Android) or `~/.local/share/rakuyomi` (Linux)
- Health check: `GET /health-check`
- Deep link schemes:
  - Compose app: `rakuyomi_bridge://start` and `...://stop`
  - Headless app: `rakuyomi_bride://start` and `...://stop`
