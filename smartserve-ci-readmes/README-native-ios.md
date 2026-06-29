# SmartServe Native iOS CI/CD

This repository contains the native iOS version of SmartServe with Jenkins + Fastlane simulator build automation.

## Current CI/CD Scope

| Target | Status |
|---|---|
| iOS Simulator `.app` | Supported |
| Jenkins artifact `.app.zip` | Supported |
| Signed `.ipa` | Not configured yet |
| Firebase App Distribution | Not configured yet |
| TestFlight/App Store | Not configured yet |

> Native iOS Firebase/TestFlight distribution requires a signed `.ipa`. A simulator `.app` cannot be uploaded to Firebase or TestFlight.

## Project Info

| Item | Value |
|---|---|
| App type | Native iOS |
| UI | SwiftUI |
| Language | Swift |
| Build tool | Xcode / xcodebuild |
| CI/CD | Jenkins + Fastlane |
| Jenkins agent | macOS agent labeled `mac` |
| Example scheme | `SmartServeNativeIOS` |
| Example project | `SmartServeNativeIOS.xcodeproj` |
| Example simulator | `iPhone 17` |

## Requirements

Install these on the Jenkins macOS agent:

- Xcode
- Xcode command line tools
- Ruby + Bundler
- Fastlane
- Jenkins agent labeled `mac`

Check:

```bash
xcodebuild -version
xcode-select -p
xcrun simctl list devices available
bundle -v
```

## Project Structure

Expected structure:

```text
SmartServeNativeIOS/
├── SmartServeNativeIOS.xcodeproj
├── SmartServeNativeIOS/
├── SmartServeNativeIOSTests/
├── SmartServeNativeIOSUITests/
├── fastlane/
├── Gemfile
└── Jenkinsfile
```

Check project/workspace:

```bash
find . -maxdepth 2 -name "*.xcodeproj" -print
find . -maxdepth 2 -name "*.xcworkspace" -print
```

Expected:

```text
./SmartServeNativeIOS.xcodeproj
./SmartServeNativeIOS.xcodeproj/project.xcworkspace
```

## Shared Scheme

Jenkins must be able to find the Xcode scheme.

In Xcode:

```text
Product → Scheme → Manage Schemes → tick Shared
```

Then confirm:

```bash
find . -path "*xcshareddata/xcschemes*" -name "*.xcscheme" -print
```

Expected:

```text
./SmartServeNativeIOS.xcodeproj/xcshareddata/xcschemes/SmartServeNativeIOS.xcscheme
```

## Fastlane Setup

Create `Gemfile`:

```ruby
source "https://rubygems.org"

gem "fastlane"
```

Install:

```bash
bundle install
```

Fastlane lane:

```text
ios build_simulator
```

This lane runs:

```text
xcodebuild
→ build for iphonesimulator
→ copy .app to artifacts/
→ zip .app
```

## Local Build Command

Run from project root:

```bash
BUILD_TYPE=DEBUG \
IOS_SCHEME=SmartServeNativeIOS \
IOS_PROJECT=SmartServeNativeIOS.xcodeproj \
IOS_DESTINATION="platform=iOS Simulator,name=iPhone 17" \
bundle exec fastlane build_simulator
```

Expected output:

```text
artifacts/SmartServeNativeIOS.app
artifacts/SmartServeNativeIOS.app.zip
```

## Jenkins Parameters

| Parameter | Example | Description |
|---|---|---|
| `BUILD_TYPE` | `DEBUG` / `RELEASE` | Simulator build configuration |
| `DISTRIBUTION` | `NONE` | Firebase/Store disabled for now |
| `IOS_SCHEME` | `SmartServeNativeIOS` | Xcode scheme |
| `IOS_PROJECT` | `SmartServeNativeIOS.xcodeproj` | Xcode project file |
| `IOS_DESTINATION` | `platform=iOS Simulator,name=iPhone 17` | Simulator destination |

## Recommended Jenkins Flow

```text
Validate Parameters
Checkout
Check Xcode Environment
Build Native iOS Simulator
Archive artifacts/*.zip
```

## Successful Jenkins Output

Expected success lines:

```text
** BUILD SUCCEEDED **
fastlane.tools finished successfully
Native iOS simulator pipeline completed successfully.
Finished: SUCCESS
```

Expected artifact:

```text
artifacts/SmartServeNativeIOS.app.zip
```

## Install App to iOS Simulator

Boot simulator:

```bash
open -a Simulator
xcrun simctl boot "iPhone 17" || true
xcrun simctl bootstatus booted
```

Install app:

```bash
xcrun simctl install booted "/Users/enz/jenkins-agent/workspace/native ios/artifacts/SmartServeNativeIOS.app"
```

Launch app:

```bash
xcrun simctl launch booted com.smartserve.mobile.SmartServeNativeIOS
```

If launch fails, check the real bundle ID:

```bash
defaults read "/Users/enz/jenkins-agent/workspace/native ios/artifacts/SmartServeNativeIOS.app/Info.plist" CFBundleIdentifier
```

## Important Notes

### 1. Xcode has no built-in terminal

Use macOS Terminal:

```text
Command + Space → Terminal
```

### 2. Simulator build does not need signing

The simulator build uses:

```text
CODE_SIGNING_ALLOWED=NO
```

### 3. Firebase/TestFlight needs signed IPA

To distribute native iOS outside simulator, configure:

```text
Apple Developer Team
Signing certificate
Provisioning profile
Bundle ID
ExportOptions.plist
Fastlane gym/build_app
```

## Next Step

After simulator CI is stable, add signed `.ipa` lane:

```text
build_ipa
```

Then add either:

```text
Firebase App Distribution for iOS
```

or:

```text
TestFlight upload
```
