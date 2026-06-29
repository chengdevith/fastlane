# SmartServe Flutter Mobile CI/CD

This repository contains the Flutter mobile version of SmartServe and a Jenkins + Fastlane setup for Android and iOS simulator builds.

## Current CI/CD Scope

| Platform | Target | Status |
|---|---|---|
| Android | Debug APK | Supported |
| Android | Release APK | Supported |
| Android | Release AAB | Supported for build/archive |
| Android | Firebase App Distribution | Supported with APK |
| iOS | Simulator `.app` | Supported |
| iOS | Signed `.ipa` | Not configured yet |
| iOS | Firebase/TestFlight | Requires signed `.ipa` |

> iOS Firebase distribution cannot use a simulator `.app`. It requires a signed `.ipa`.

## Requirements

Install these on the Jenkins macOS agent:

- Flutter SDK
- Android Studio / Android SDK
- Xcode
- Ruby + Bundler
- Fastlane
- Jenkins agent labeled `mac`

Recommended checks:

```bash
flutter --version
java -version
xcodebuild -version
adb version
bundle -v
```

## Important Android Package Name

For Firebase App Distribution, the Android `applicationId` must match the Firebase Android app package.

Example:

```text
com.smartserve.mobile
```

Check the APK package:

```bash
aapt dump badging build/app/outputs/flutter-apk/app-release.apk | grep package
```

## Fastlane Setup

Fastlane should exist under:

```text
android/fastlane/Fastfile
ios/fastlane/Fastfile
```

For Android Firebase upload, the Firebase plugin is required:

```bash
cd android
bundle exec fastlane add_plugin firebase_app_distribution
bundle install
```

Expected plugin file:

```ruby
gem 'fastlane-plugin-firebase_app_distribution'
```

## Jenkins Parameters

Typical parameters:

| Parameter | Example | Description |
|---|---|---|
| `BUILD_TYPE` | `DEBUG` / `RELEASE` | Build mode |
| `ANDROID_ARTIFACT` | `APK` / `AAB` | Android artifact type |
| `DISTRIBUTION` | `NONE` / `FIREBASE` | Distribution target |
| `VERSION_NAME` | `1.0.0` | App version name |
| `VERSION_CODE` | Jenkins build number | Android version code |
| `RELEASE_NOTES` | `SmartServe Flutter build` | Firebase release notes |

## Jenkins Credentials

For Firebase Android upload:

| Credential ID | Type | Value |
|---|---|---|
| `firebase-android-app-id` | Secret text | Firebase Android App ID |
| `firebase-token` | Secret text | Firebase CLI token |

## Local Flutter Build Commands

### Android Debug APK

```bash
flutter clean
flutter pub get
flutter build apk --debug
```

Output:

```text
build/app/outputs/flutter-apk/app-debug.apk
```

### Android Release APK

```bash
flutter clean
flutter pub get
flutter build apk --release
```

Output:

```text
build/app/outputs/flutter-apk/app-release.apk
```

### Android Release AAB

```bash
flutter clean
flutter pub get
flutter build appbundle --release
```

Output:

```text
build/app/outputs/bundle/release/app-release.aab
```

### iOS Simulator Build

```bash
flutter clean
flutter pub get
flutter build ios --simulator
```

Output:

```text
build/ios/iphonesimulator/Runner.app
```

## Install Android APK to Emulator

Start emulator first, then run:

```bash
adb devices
adb install -r build/app/outputs/flutter-apk/app-debug.apk
adb shell monkey -p com.smartserve.mobile -c android.intent.category.LAUNCHER 1
```

## Install iOS Simulator App

Boot simulator:

```bash
open -a Simulator
xcrun simctl boot "iPhone 17" || true
xcrun simctl bootstatus booted
```

Install:

```bash
xcrun simctl install booted build/ios/iphonesimulator/Runner.app
```

Launch:

```bash
xcrun simctl launch booted com.smartserve.mobile
```

If launch fails, check the real bundle ID:

```bash
defaults read "$(pwd)/build/ios/iphonesimulator/Runner.app/Info.plist" CFBundleIdentifier
```

## Artifact Rules

Jenkins should archive generated files only:

```text
Android APK: build/app/outputs/flutter-apk/*.apk
Android AAB: build/app/outputs/bundle/**/*.aab
iOS Simulator: zip Runner.app first, then archive Runner.app.zip
```

Do not archive the `.app` directory directly without zipping it.

## Current Recommended Flow

For learning CI/CD:

```text
1. Build Android debug APK
2. Build Android release APK
3. Upload Android release APK to Firebase
4. Build iOS simulator .app
5. Later: configure iOS signed IPA
```

## Notes

- Firebase App Distribution for Android should use APK unless the Firebase project is linked to Google Play.
- iOS signed `.ipa` requires Apple Developer Team, certificate, provisioning profile, and correct Bundle ID.
- Use separate Jenkins workspaces for parallel builds to avoid output conflicts.
