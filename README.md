# rusty-v8-android

Prebuilt `librusty_v8.a` static libraries for **Android 7.0 (API 24)**.

This fork provides prebuilt `rusty_v8` static libraries compatible with **Android 7.0 (API 24)**, which are not available from existing distributions.

## Targets

| Target | Runner | Use case |
|--------|--------|----------|
| `aarch64-linux-android` | Linux | Android device |
| `x86_64-linux-android` | Linux | Android emulator |

## Scope

- Supported: **Android 7.0 (API 24)**
- Not supported:
  - Android API < 24
  - Android API > 24
  - iOS or other platforms

If you need broader platform support, use the upstream project instead.

## Usage

Set `RUSTY_V8_MIRROR` so rusty_v8's build script fetches the prebuilt archive instead of compiling V8:

```bash
RUSTY_V8_MIRROR=https://github.com/ancientcatz/rusty-v8-android/releases/download \
  cargo build --target aarch64-linux-android
```

Or download the `.a.gz` for your target manually from [Releases](https://github.com/ancientcatz/rusty-v8-android/releases), decompress, then:

```bash
RUSTY_V8_ARCHIVE=/path/to/librusty_v8_release_aarch64-linux-android.a \
  cargo build --target aarch64-linux-android
```

## Building a new version

Trigger the **Build** workflow via `workflow_dispatch`, specifying the rusty_v8 version (e.g. `130.0.7`).

The workflow compiles V8 from source for the supported Android targets (API 24) and uploads the artifacts as a GitHub Release.
