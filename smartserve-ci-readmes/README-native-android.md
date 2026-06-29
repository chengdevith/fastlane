# SmartServe Native Android CI/CD

This repository contains the native Android version of SmartServe with Jenkins + Fastlane CI/CD.

## Current CI/CD Scope

| Target | Status |
|---|---|
| Debug APK | Supported |
| Release APK | Supported |
| Release AAB | Supported |
| Firebase App Distribution | Supported with APK |
| Play Store upload | Not configured yet |

## Project Info

| Item | Value |
|---|---|
| App type | Native Android |
| Language | Kotlin |
| CI/CD | Jenkins + Fastlane |
| Jenkins agent | macOS agent labeled `mac` |
| Firebase package example | `com.smartserve.android` or `com.smartserve.mobile` |

## Requirements

Install these on the Jenkins macOS agent:

- Android Studio
- Android SDK
- Java 21
- Ruby + Bundler
- Fastlane
- Jenkins agent labeled `mac`

Check:

```bash
java -version
./gradlew --version
bundle -v
adb version
```

## Important: Package Name and Firebase App ID

Firebase App Distribution validates that the APK package name matches the Firebase Android app package.

Example error:

```text
The APK package name 'com.smartserve.android' does not match your Firebase app's package name 'com.smartserve.mobile'
```

You have two valid choices:

### Option 1: Keep native Android package

Use:

```text
com.smartserve.android
```

Then Jenkins credential `firebase-native-android-app-id` must contain the Firebase App ID for the Firebase app with package:

```text
com.smartserve.android
```

### Option 2: Reuse mobile Firebase app

Change Android `applicationId` to:

```text
com.smartserve.mobile
```

Then use the Firebase App ID for package:

```text
com.smartserve.mobile
```

Check current package:

```bash
grep -R "applicationId\|namespace" app/build.gradle.kts app/build.gradle 2>/dev/null
```

Check APK package:

```bash
aapt dump badging app/build/outputs/apk/release/app-release.apk | grep package
```

If the release output is unsigned:

```bash
aapt dump badging app/build/outputs/apk/release/app-release-unsigned.apk | grep package
```

## Fastlane Plugin

Firebase App Distribution requires this plugin.

Install:

```bash
bundle exec fastlane add_plugin firebase_app_distribution
bundle install
```

Expected `fastlane/Pluginfile`:

```ruby
gem 'fastlane-plugin-firebase_app_distribution'
```

## Fastlane Lanes

Recommended lanes:

| Lane | Purpose |
|---|---|
| `build_android` | Build APK or AAB |
| `firebase_release` | Build release APK and upload to Firebase |

## Environment Variables

| Variable | Example | Description |
|---|---|---|
| `BUILD_TYPE` | `DEBUG` / `RELEASE` | Build type |
| `ANDROID_ARTIFACT` | `APK` / `AAB` | Artifact type |
| `VERSION_NAME` | `1.0.0` | Version name |
| `VERSION_CODE` | `1` | Version code |
| `RELEASE_NOTES_PATH` | `release-notes.txt` | Firebase release notes file |
| `FIREBASE_NATIVE_ANDROID_APP_ID` | `1:xxx:android:xxx` | Firebase App ID |
| `FIREBASE_TOKEN` | secret | Firebase CLI token |

## Jenkins Credentials

| Credential ID | Type | Description |
|---|---|---|
| `firebase-native-android-app-id` | Secret text | Firebase Android App ID for this native app |
| `firebase-token` | Secret text | Firebase CLI token |

## Local Build Commands

### Debug APK

```bash
BUILD_TYPE=DEBUG \
ANDROID_ARTIFACT=APK \
VERSION_NAME=1.0.0 \
VERSION_CODE=1 \
bundle exec fastlane build_android
```

Output:

```text
app/build/outputs/apk/debug/app-debug.apk
```

### Release APK

```bash
BUILD_TYPE=RELEASE \
ANDROID_ARTIFACT=APK \
VERSION_NAME=1.0.0 \
VERSION_CODE=1 \
bundle exec fastlane build_android
```

Output:

```text
app/build/outputs/apk/release/app-release.apk
```

If signing is not configured, Gradle may output:

```text
app/build/outputs/apk/release/app-release-unsigned.apk
```

### Release AAB

```bash
BUILD_TYPE=RELEASE \
ANDROID_ARTIFACT=AAB \
VERSION_NAME=1.0.0 \
VERSION_CODE=1 \
bundle exec fastlane build_android
```

Output:

```text
app/build/outputs/bundle/release/app-release.aab
```

### Firebase Upload

```bash
BUILD_TYPE=RELEASE \
ANDROID_ARTIFACT=APK \
VERSION_NAME=1.0.0 \
VERSION_CODE=1 \
RELEASE_NOTES_PATH=release-notes.txt \
FIREBASE_NATIVE_ANDROID_APP_ID="1:xxx:android:xxx" \
FIREBASE_TOKEN="xxx" \
bundle exec fastlane firebase_release
```

## Jenkins Parameters

| Parameter | Example | Description |
|---|---|---|
| `BUILD_TYPE` | `DEBUG` / `RELEASE` | Build type |
| `ANDROID_ARTIFACT` | `APK` / `AAB` | Artifact type |
| `DISTRIBUTION` | `NONE` / `FIREBASE` / `STORE` | Distribution target |
| `VERSION_NAME` | `1.0.0` | Version name |
| `VERSION_CODE` | empty = Jenkins build number | Version code |
| `RELEASE_NOTES` | `SmartServe Native Android build` | Firebase notes |

## Recommended Jenkins Flow

```text
Validate Parameters
Checkout
Prepare Version and Release Notes
Build Native Android
Archive Artifacts
```

## Parallel Build Option

Parallel is useful for full CI:

```text
Parallel Native Android Build
├── Debug APK
├── Release APK
└── Release AAB
```

Use separate workspaces for each branch:

```groovy
ws("${env.WORKSPACE}@debug-apk") {
    checkout scm
}
```

Do not run parallel Gradle builds in the same workspace because all branches write to `app/build`.

## Install APK to Emulator

Start emulator first:

```bash
adb devices
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb shell monkey -p com.smartserve.android -c android.intent.category.LAUNCHER 1
```

Change package name if your `applicationId` is different.

## Release Signing

For real Firebase testing and Play Store, configure release signing.

Unsigned release APK:

```text
app-release-unsigned.apk
```

is not suitable for proper tester installation or Play Store release.

Recommended next step:

```text
Configure Android release keystore in Jenkins credentials
```

## Notes

- Firebase token is not package-specific. You do not need a new Firebase token for every app.
- Firebase App ID is app-specific. Use the correct Firebase App ID for the APK package.
- Play Store upload should use AAB, not APK.
