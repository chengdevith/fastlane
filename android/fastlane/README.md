fastlane documentation
----

# Installation

Make sure you have the latest version of the Xcode command line tools installed:

```sh
xcode-select --install
```

For _fastlane_ installation instructions, see [Installing _fastlane_](https://docs.fastlane.tools/#installing-fastlane)

# Available Actions

## Android

### android clean

```sh
[bundle exec] fastlane android clean
```

Safe clean Android project

### android build_debug

```sh
[bundle exec] fastlane android build_debug
```

Build Android debug APK

### android install_debug

```sh
[bundle exec] fastlane android install_debug
```

Install Android debug APK to connected device/emulator

### android build_install_debug

```sh
[bundle exec] fastlane android build_install_debug
```

Build and install Android debug APK

### android clean_build_debug

```sh
[bundle exec] fastlane android clean_build_debug
```

Clean and build Android debug APK

### android build_release

```sh
[bundle exec] fastlane android build_release
```

Build Android release APK

### android launch_debug

```sh
[bundle exec] fastlane android launch_debug
```

Launch Android app

----

This README.md is auto-generated and will be re-generated every time [_fastlane_](https://fastlane.tools) is run.

More information about _fastlane_ can be found on [fastlane.tools](https://fastlane.tools).

The documentation of _fastlane_ can be found on [docs.fastlane.tools](https://docs.fastlane.tools).
