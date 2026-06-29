# Fastlane + Jenkins CI/CD Flow and Usage

This README explains the general **Fastlane + Jenkins** flow used for SmartServe mobile CI/CD across:

- Flutter
- Native Android
- Native iOS

The main goal is to automate:

```text
Source Code → Jenkins → Fastlane → Platform Build Tool → Artifact → Optional Distribution
```

---

## 1. What Fastlane Does

Fastlane is the automation layer for mobile build and release tasks.

In this project, Fastlane handles:

```text
Android:
- Build debug APK
- Build release APK
- Build release AAB
- Upload APK to Firebase App Distribution

iOS:
- Build simulator .app
- Zip .app for Jenkins artifact
- Later: build signed .ipa
- Later: upload .ipa to Firebase/TestFlight
```

Fastlane is called by Jenkins, but it can also be run locally.

---

## 2. What Jenkins Does

Jenkins is the CI/CD orchestrator.

Jenkins handles:

```text
1. Checkout source code from GitHub
2. Validate build parameters
3. Install Ruby dependencies with Bundler
4. Run Fastlane lane
5. Archive build artifacts
6. Inject secrets from Jenkins credentials
```

Jenkins should not contain hard-coded secrets.

Use Jenkins credentials for:

```text
Firebase App ID
Firebase token
Android keystore
Apple signing files
Service account JSON
```

---

## 3. High-Level Flow

```text
Developer pushes code to GitHub
        ↓
Jenkins job starts
        ↓
Jenkins checks out source code
        ↓
Jenkins prepares parameters and release notes
        ↓
Jenkins runs bundle install
        ↓
Jenkins runs Fastlane
        ↓
Fastlane calls Gradle / Flutter / xcodebuild
        ↓
Build artifact is generated
        ↓
Jenkins archives artifact
        ↓
Optional: Fastlane uploads artifact to Firebase / Store
```

---

## 4. Repository Requirements

Each mobile repository should contain:

```text
Repository Root
├── Jenkinsfile
├── Gemfile
├── fastlane/
│   ├── Fastfile
│   ├── Appfile
│   └── Pluginfile
└── project source files
```

Examples:

```text
Flutter:
├── android/fastlane/Fastfile
├── ios/fastlane/Fastfile
└── Jenkinsfile

Native Android:
├── app/
├── fastlane/Fastfile
├── fastlane/Appfile
├── fastlane/Pluginfile
├── Gemfile
└── Jenkinsfile

Native iOS:
├── SmartServeNativeIOS.xcodeproj
├── SmartServeNativeIOS/
├── fastlane/Fastfile
├── Gemfile
└── Jenkinsfile
```

---

## 5. Required Tools on Jenkins Agent

The Jenkins macOS agent should have:

```text
Java 21
Android SDK
Flutter SDK
Xcode
Ruby
Bundler
Fastlane
Git
Firebase CLI or Firebase token
```

Check tools:

```bash
java -version
git --version
bundle -v
fastlane --version
xcodebuild -version
xcrun simctl list devices available
adb version
flutter --version
```

---

## 6. Gemfile

Use a `Gemfile` so Jenkins installs the same Fastlane version consistently.

```ruby
source "https://rubygems.org"

gem "fastlane"

plugins_path = File.join(File.dirname(__FILE__), "fastlane", "Pluginfile")
eval_gemfile(plugins_path) if File.exist?(plugins_path)
```

Install locally:

```bash
bundle install
```

In Jenkins:

```bash
bundle config set path vendor/bundle
bundle install
```

---

## 7. Fastlane Pluginfile

For Android Firebase App Distribution:

```ruby
gem 'fastlane-plugin-firebase_app_distribution'
```

Install plugin:

```bash
bundle exec fastlane add_plugin firebase_app_distribution
bundle install
```

Commit:

```bash
git add Gemfile Gemfile.lock fastlane/Pluginfile
git commit -m "Add Firebase App Distribution plugin"
git push
```

---

## 8. Jenkins Credentials

Recommended Jenkins credentials:

| Credential ID | Type | Used For |
|---|---|---|
| `firebase-token` | Secret text | Firebase CLI token |
| `firebase-android-app-id` | Secret text | Flutter Android Firebase App ID |
| `firebase-native-android-app-id` | Secret text | Native Android Firebase App ID |
| `android-release-keystore` | Secret file | Android release signing |
| `android-keystore-password` | Secret text | Android keystore password |
| `android-key-alias` | Secret text | Android key alias |
| `android-key-password` | Secret text | Android key password |
| `apple-api-key` | Secret file | Later for App Store/TestFlight |
| `ios-signing-certificate` | Secret file | Later for signed IPA |
| `ios-provisioning-profile` | Secret file | Later for signed IPA |

Never commit these secrets to GitHub.

---

## 9. Common Jenkins Parameters

Use parameters so the same Jenkinsfile can build different targets.

| Parameter | Example | Description |
|---|---|---|
| `BUILD_TYPE` | `DEBUG` / `RELEASE` | Build mode |
| `ANDROID_ARTIFACT` | `APK` / `AAB` | Android artifact type |
| `DISTRIBUTION` | `NONE` / `FIREBASE` / `STORE` | Distribution target |
| `VERSION_NAME` | `1.0.0` | App version name |
| `VERSION_CODE` | Jenkins build number | Android version code |
| `RELEASE_NOTES` | `SmartServe build` | Firebase release notes |
| `IOS_SCHEME` | `SmartServeNativeIOS` | iOS scheme |
| `IOS_PROJECT` | `SmartServeNativeIOS.xcodeproj` | iOS project |
| `IOS_DESTINATION` | `platform=iOS Simulator,name=iPhone 17` | iOS simulator |

---

## 10. Parameter Validation Rules

Recommended validation:

```text
DEBUG + AAB = invalid
DEBUG + FIREBASE = invalid
FIREBASE + AAB = invalid unless project is linked with Google Play
STORE + APK = invalid
STORE + DEBUG = invalid
iOS FIREBASE/STORE = invalid until signed IPA is configured
```

Example Jenkins validation:

```groovy
script {
    if (params.BUILD_TYPE == 'DEBUG' && params.ANDROID_ARTIFACT == 'AAB') {
        error("DEBUG build only supports APK.")
    }

    if (params.BUILD_TYPE == 'DEBUG' && params.DISTRIBUTION != 'NONE') {
        error("DEBUG build should use DISTRIBUTION=NONE.")
    }

    if (params.DISTRIBUTION == 'FIREBASE' && params.ANDROID_ARTIFACT != 'APK') {
        error("Firebase App Distribution should use APK.")
    }

    if (params.DISTRIBUTION == 'STORE' && params.ANDROID_ARTIFACT != 'AAB') {
        error("Play Store upload should use AAB.")
    }
}
```

---

## 11. Native Android Fastlane Usage

### Build Debug APK

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

### Build Release APK

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

If signing is not configured:

```text
app/build/outputs/apk/release/app-release-unsigned.apk
```

### Build Release AAB

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

### Upload APK to Firebase

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

---

## 12. Native iOS Fastlane Usage

### Build iOS Simulator App

```bash
BUILD_TYPE=DEBUG \
IOS_SCHEME=SmartServeNativeIOS \
IOS_PROJECT=SmartServeNativeIOS.xcodeproj \
IOS_DESTINATION="platform=iOS Simulator,name=iPhone 17" \
bundle exec fastlane build_simulator
```

Output:

```text
artifacts/SmartServeNativeIOS.app
artifacts/SmartServeNativeIOS.app.zip
```

### Important iOS Limitation

This is simulator output:

```text
.app
```

Firebase/TestFlight requires:

```text
signed .ipa
```

So this is not valid yet:

```text
iOS simulator .app → Firebase
```

Correct future flow:

```text
iOS project → signed .ipa → Firebase App Distribution or TestFlight
```

---

## 13. Flutter Fastlane Usage

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

### iOS Simulator App

```bash
flutter clean
flutter pub get
flutter build ios --simulator
```

Output:

```text
build/ios/iphonesimulator/Runner.app
```

Zip before archiving in Jenkins:

```bash
cd build/ios/iphonesimulator
zip -r Runner.app.zip Runner.app
```

---

## 14. Jenkinsfile Flow

Recommended Jenkins stages:

```text
Validate Parameters
Checkout
Prepare Version and Release Notes
Build
Archive Artifacts
```

For iOS, add environment check:

```text
Check Xcode Environment
```

For Android, add Gradle check:

```text
Check Java and Gradle
```

---

## 15. Example Jenkins Build Step: Android Firebase

```groovy
withCredentials([
    string(credentialsId: 'firebase-native-android-app-id', variable: 'FIREBASE_NATIVE_ANDROID_APP_ID'),
    string(credentialsId: 'firebase-token', variable: 'FIREBASE_TOKEN')
]) {
    sh '''
        BUILD_TYPE=RELEASE \
        ANDROID_ARTIFACT=APK \
        VERSION_NAME=${APP_VERSION_NAME} \
        VERSION_CODE=${APP_VERSION_CODE} \
        RELEASE_NOTES_PATH=release-notes.txt \
        bundle exec fastlane firebase_release
    '''
}
```

---

## 16. Example Jenkins Build Step: iOS Simulator

```groovy
sh '''
    BUILD_TYPE=${BUILD_TYPE} \
    IOS_SCHEME=${IOS_SCHEME} \
    IOS_PROJECT=${IOS_PROJECT} \
    IOS_DESTINATION="${IOS_DESTINATION}" \
    bundle exec fastlane build_simulator
'''
```

Archive:

```groovy
archiveArtifacts artifacts: 'artifacts/*.zip', allowEmptyArchive: false
```

---

## 17. Jenkins Artifact Strategy

Archive files, not directories.

Good:

```text
.apk
.aab
.zip
.ipa
```

Bad:

```text
.app directory directly
```

For iOS simulator:

```bash
zip -r SmartServeNativeIOS.app.zip SmartServeNativeIOS.app
```

Then archive:

```groovy
archiveArtifacts artifacts: 'artifacts/*.zip', allowEmptyArchive: false
```

---

## 18. Parallel Build Usage

Parallel stages are useful when building multiple artifacts at the same time.

Example:

```text
Parallel Build
├── Debug APK
├── Release APK
└── Release AAB
```

Use separate workspaces:

```groovy
ws("${env.WORKSPACE}@debug-apk") {
    checkout scm
}
```

Do not run parallel Android builds in the same workspace because they all write to:

```text
app/build
```

Do not run parallel iOS builds in the same DerivedData path because they can conflict.

Use different paths:

```text
build/DerivedData-debug
build/DerivedData-release
```

---

## 19. Release Notes

Jenkins should create release notes dynamically:

```groovy
writeFile file: 'release-notes.txt', text: params.RELEASE_NOTES + "\n"
stash name: 'release-notes', includes: 'release-notes.txt'
```

Fastlane reads it:

```ruby
release_notes_path = ENV["RELEASE_NOTES_PATH"]
release_notes = File.exist?(release_notes_path) ? File.read(release_notes_path) : "SmartServe build"
```

---

## 20. Android Package Name Rules

Firebase App Distribution requires APK package name to match the Firebase app package.

Check Android package:

```bash
aapt dump badging app/build/outputs/apk/release/app-release.apk | grep package
```

Example mismatch:

```text
APK package: com.smartserve.android
Firebase app package: com.smartserve.mobile
```

Fix by either:

```text
1. Change applicationId to match Firebase
```

or:

```text
2. Use the Firebase App ID that matches the APK package
```

Firebase token is not package-specific.

Firebase App ID is package-specific.

---

## 21. Android Signing

Unsigned release APK:

```text
app-release-unsigned.apk
```

is acceptable only for early CI testing, not for production.

For real Firebase testing and Play Store:

```text
Configure Android release signing
```

Recommended Jenkins credentials:

```text
android-release-keystore
android-keystore-password
android-key-alias
android-key-password
```

---

## 22. iOS Signing

Simulator build does not need signing:

```text
CODE_SIGNING_ALLOWED=NO
```

Signed IPA needs:

```text
Apple Developer Team
Certificate
Provisioning Profile
Bundle ID
ExportOptions.plist
```

Future Fastlane lanes:

```text
build_ipa
firebase_release
testflight_release
```

---

## 23. Common Troubleshooting

### Fastlane cannot find Firebase action

Error:

```text
Could not find action, lane or variable 'firebase_app_distribution'
```

Fix:

```bash
bundle exec fastlane add_plugin firebase_app_distribution
bundle install
git add Gemfile Gemfile.lock fastlane/Pluginfile
git commit -m "Add Firebase App Distribution plugin"
git push
```

### APK package mismatch

Error:

```text
The APK package name does not match your Firebase app's package name
```

Fix:

```text
Use the correct Firebase App ID for that package
```

or change Android `applicationId`.

### Xcode project not found

Error:

```text
xcodebuild: error: 'SmartServeNativeIOS.xcodeproj' does not exist
```

Fix Fastfile to use project root:

```ruby
project_root = File.expand_path("..", __dir__)
project_path = File.join(project_root, ENV["IOS_PROJECT"])
```

### iOS scheme not found

Fix:

```text
Xcode → Product → Scheme → Manage Schemes → tick Shared
```

Commit the shared scheme:

```bash
git add SmartServeNativeIOS.xcodeproj/xcshareddata/xcschemes/*.xcscheme
git commit -m "Share Xcode scheme"
git push
```

### Simulator not found

Check available simulators:

```bash
xcrun simctl list devices available
```

Then update Jenkins parameter:

```text
IOS_DESTINATION=platform=iOS Simulator,name=iPhone 17 Pro
```

### Jenkins cannot archive `.app`

`.app` is a directory. Zip it first:

```bash
zip -r SmartServeNativeIOS.app.zip SmartServeNativeIOS.app
```

---

## 24. Recommended Learning Order

```text
1. Native Android debug APK
2. Native Android release APK
3. Native Android Firebase upload
4. Native iOS simulator build
5. Flutter Android build
6. Flutter iOS simulator build
7. Android release signing
8. iOS signed IPA
9. TestFlight / App Store / Play Store
```

---

## 25. Current SmartServe CI/CD Status

```text
Native Android:
- Jenkins build works
- Fastlane works
- Firebase upload works when Firebase App ID matches package

Native iOS:
- Jenkins build works
- Fastlane simulator lane works
- .app and .app.zip artifacts work

Flutter:
- Android debug/release build supported
- iOS simulator build supported
- Firebase Android supported with APK
```

---

## 26. Best Practices

- Keep Jenkinsfile in repository root.
- Use `bundle exec fastlane`, not global `fastlane`.
- Store secrets in Jenkins credentials.
- Do not commit keystore, tokens, certificates, or provisioning profiles.
- Always archive build artifacts.
- Use parameters for flexible builds.
- Use separate workspaces for parallel builds.
- Keep simulator build and signed release build as separate lanes.
- Use APK for Firebase Android distribution.
- Use AAB for Play Store.
- Use signed IPA for iOS Firebase/TestFlight.
