# iOS development status and contribution plan

**Status date:** 2026-09-05  
**Analyzed revision:** `637fd430a1ca548dfe6c4639ee3afb023289f8cc`  
**Upstream repository:** [MostroP2P/mobile](https://github.com/MostroP2P/mobile)

## Executive summary

Mostro Mobile does not need a separate native iOS rewrite. The application is
already a Flutter codebase with an iOS Xcode project, an iOS bundle identifier,
Firebase configuration, deep-link handling, camera access, biometric
authentication code, and shared Android/iOS application logic.

The missing part is a verified and maintained iOS delivery path. At the analyzed
revision:

- there is no public evidence that `main` is routinely built on macOS or tested
  on an iPhone;
- the current CI runs Flutter analysis and tests on Ubuntu, but does not compile
  the iOS target;
- the release workflow creates Android artifacts only;
- issue [#80](https://github.com/MostroP2P/mobile/issues/80), "Implement iOS
  build", remains open;
- issue [#682](https://github.com/MostroP2P/mobile/issues/682) documents a
  packaging failure with `lucide_icons` on newer Flutter versions;
- Apple signing, App Store Connect, APNs, and TestFlight ownership cannot be
  verified from the public repository.

The recommended contribution is therefore incremental:

1. reproduce the current build on a Mac and physical iPhone;
2. document every real failure against an exact Flutter/Xcode revision;
3. fix platform configuration and dependency blockers in small pull requests;
4. add an unsigned iOS compile check to CI;
5. only then work with a maintainer who controls the Apple Developer account to
   create a signed TestFlight pipeline.

This report is a repository audit and implementation plan, not a formal security
or legal review.

## What already exists

### Shared application and iOS wrapper

The repository contains:

- a Flutter application shared with Android and desktop;
- `ios/Runner.xcworkspace` and the normal Flutter Xcode project structure;
- the bundle identifier `network.mostro.app`;
- an iOS 15 deployment target in the Xcode project;
- application icons and launch assets;
- camera permission text for QR scanning;
- the custom URL scheme `mostro:`;
- background mode declarations for fetch, processing, and remote notifications;
- Firebase and FCM integration in the shared Dart code;
- biometric authentication through `local_auth`;
- secure local storage through `flutter_secure_storage`.

This is enough to treat iOS as a platform enablement and release-engineering
project, not as a new application.

### Historical iOS work

Older releases included an `ios_build.tar.gz` artifact. The historical workflow
used `flutter build ipa --no-codesign`, which showed that the project could
compile at that point but did not create a signed TestFlight-ready release.

In August 2025:

- [PR #274](https://github.com/MostroP2P/mobile/pull/274) updated the minimum
  target and storyboards for newer Xcode versions;
- [PR #275](https://github.com/MostroP2P/mobile/pull/275) removed iOS from the
  release workflow and moved the main build to Ubuntu.

The current Android release process is active, but there is no equivalent public
iOS release process.

## Verified gaps and risks

### 1. No iOS compile gate in CI

`.github/workflows/flutter.yml` uses Ubuntu and Flutter `3.32.5`. It runs
dependency installation, code generation, analysis, and Flutter tests, but it
never invokes Xcode or CocoaPods.

`.github/workflows/release.yml` uses Ubuntu and Flutter `3.32.2`. It produces
signed Android APK and AAB artifacts only.

A green CI run therefore does not prove that the iOS project compiles, archives,
signs, starts, receives notifications, or survives background transitions.

### 2. Flutter version drift

`.fvmrc` declares Flutter `3.35.7`, while the main and release workflows use
`3.32.5` and `3.32.2` respectively.

Before fixing iOS-specific code, the project should define one supported Flutter
version for local development and CI. Otherwise, a failure may be caused by
toolchain drift rather than the iOS implementation itself.

### 3. Open `lucide_icons` compatibility issue

The repository still depends on `lucide_icons: ^0.257.0`. Issue
[#682](https://github.com/MostroP2P/mobile/issues/682) reports that this package
fails on newer Flutter versions because it extends `IconData`, which became
final outside its library.

The first build attempt should confirm whether the failure occurs with the
project-pinned Flutter version. If it does, replace the dependency with the
maintained alternative proposed in the issue, migrate imports, and validate
rendering on Android and iOS in a dedicated pull request.

### 4. Missing Face ID usage description

The shared application uses `local_auth`, but `ios/Runner/Info.plist` does not
define `NSFaceIDUsageDescription`.

Before biometric authentication is exercised on a Face ID device, the project
should add a clear localized explanation and test:

- first authorization;
- successful and failed authentication;
- cancellation;
- device passcode fallback behavior;
- application resume after the system authentication sheet closes.

### 5. Background execution needs platform validation

`Info.plist` declares:

- `fetch`;
- `processing`;
- `remote-notification`.

No `BGTaskSchedulerPermittedIdentifiers` entry is present. The background plugin
and application behavior must be checked against the task type actually
registered at runtime. Unneeded modes should be removed, and required
identifiers should be declared explicitly.

Background delivery must be tested on a physical device. Simulator success is
not sufficient evidence for APNs or iOS background execution.

### 6. Push notification entitlements are not present in the repository

No iOS entitlements file is tracked, and there is no visible `aps-environment`
capability. This does not prove that the maintainers have no private Apple
configuration, but the public project does not provide a reproducible
push-notification setup.

A maintainer with access to the Apple Developer account will need to confirm:

- the App ID for `network.mostro.app`;
- the Push Notifications capability;
- Background Modes configuration;
- the APNs key or certificate used by Firebase;
- the provisioning profile used for development and distribution.

Secrets, certificates, profiles, and private keys must never be committed to
this repository or to a contributor fork.

### 7. CocoaPods resolution is not locked

`ios/Podfile` is tracked, but `ios/Podfile.lock` is not. The project should
decide and document whether application builds are expected to commit
`Podfile.lock`.

For a shipped application, committing the lock file is normally preferable
because it makes CocoaPods dependency resolution reproducible. Any change should
be discussed with maintainers before opening a pull request.

The Dart dependency `nip44` also follows a moving `master` branch. Pinning it to
a tested commit would further improve build reproducibility.

### 8. Native test coverage is effectively absent

`ios/RunnerTests/RunnerTests.swift` is present but still contains the default
template without meaningful assertions. Flutter tests protect the shared code,
but there is no native test that exercises iOS startup, URL handling,
notification registration, or platform channels.

Native tests should be added only where they protect iOS-specific behavior. The
most important initial gate is still a real debug build and smoke test on an
iPhone.

## Unknowns requiring maintainer input

The public repository cannot answer the following questions:

1. Who owns and administers the Apple Developer and App Store Connect accounts?
2. Does an App ID already exist for `network.mostro.app`?
3. Is there a private or internal TestFlight build?
4. Are APNs and Firebase already connected outside the repository?
5. Which countries should an App Store release target?
6. Who will be the legal publisher and respond to App Review?
7. Has the team reviewed Apple's rules for applications facilitating
   cryptocurrency exchange?

These questions should be answered before spending time on the signed release
workflow. They do not block local development and unsigned CI compilation.

## Recommended implementation sequence

### Phase 0: coordinate the work

- Confirm with maintainers that `MostroP2P/mobile` is the canonical client.
- Comment on or reference issue
  [#80](https://github.com/MostroP2P/mobile/issues/80).
- Tell maintainers that the first goal is a reproducible physical-device
  development build, not an immediate App Store submission.
- Agree to keep changes in small reviewable pull requests.

### Phase 1: establish a reproducible baseline on macOS

Use the exact revision and pinned Flutter version first:

```sh
git fetch upstream
git switch main
git merge --ff-only upstream/main
fvm install
fvm flutter doctor -v
fvm flutter pub get
fvm dart run build_runner build -d
fvm flutter gen-l10n
cd ios
pod install --repo-update
cd ..
fvm flutter devices
fvm flutter run -d <iphone-device-id>
```

Record:

- macOS version;
- Xcode version;
- CocoaPods version;
- Flutter and Dart versions;
- exact commit SHA;
- whether the failure happens during Dart compilation, CocoaPods installation,
  Xcode build, signing, installation, or runtime startup;
- complete but sanitized logs.

Do not combine dependency migration, signing changes, notification setup, and
unrelated refactors into the first pull request.

### Phase 2: fix deterministic build blockers

Suggested pull-request order:

1. align the supported Flutter version across `.fvmrc` and CI;
2. resolve `lucide_icons` compatibility if reproduced;
3. add `NSFaceIDUsageDescription` and test local authentication;
4. correct background task declarations and identifiers;
5. agree on and enforce CocoaPods lock-file policy;
6. add an unsigned macOS CI job that compiles the iOS simulator target.

An initial CI command can be as small as:

```sh
flutter build ios --simulator --no-codesign
```

This does not replace physical-device testing or prove that an archive can be
distributed. It only prevents obvious iOS compile regressions from landing
unnoticed.

### Phase 3: physical-device functional validation

Test at least:

- clean install and first launch;
- account/key creation and restoration;
- secure-storage persistence after restart;
- biometric unlock on Face ID and Touch ID devices if supported;
- QR scanning and camera denial/recovery;
- `mostro:` deep links from cold start and warm state;
- relay connection and recovery after network changes;
- maker and taker order flows against the same public network used by Android;
- background/foreground transitions during an active trade;
- local and remote notifications;
- force quit and relaunch during recoverable order states;
- accessibility, text scaling, and small-screen layout;
- release-mode behavior, not debug mode only.

No test should use funds that Pierre or another contributor cannot afford to
lose. Start with the smallest possible amounts and explicitly identify the test
environment and relay set.

### Phase 4: TestFlight

This phase requires a maintainer-controlled Apple setup:

- Apple Developer team membership;
- App ID and capabilities;
- development and distribution signing;
- App Store Connect application record;
- APNs integration;
- version/build-number policy;
- export-compliance answers;
- TestFlight metadata and reviewer instructions;
- a secret-management solution for CI.

Prefer a standard, reviewable workflow using `xcodebuild`, Fastlane, or a
well-maintained GitHub Action. Keep signing assets outside Git and use
short-lived or encrypted CI credentials.

### Phase 5: App Store readiness

Before submission, prepare:

- public privacy policy and support URL;
- App Privacy disclosures matching actual Firebase, notification, and
  device-data behavior;
- screenshots and metadata for supported devices;
- review notes that explain the non-custodial Mostro/Nostr/Lightning
  architecture and provide a test path;
- a country-by-country distribution decision;
- a legal and policy review of Apple's cryptocurrency exchange rules.

The main non-technical risk is App Review. Open source and non-custodial
architecture are important, but they do not automatically guarantee approval for
an application that facilitates cryptocurrency trades.

## Contribution workflow

The repository's own contribution guide asks external contributors to use the
standard fork workflow.

Recommended remote layout:

```text
origin    https://github.com/Pivii/mostro-mobile.git
upstream  https://github.com/MostroP2P/mobile.git
```

Keep `main` clean and synchronized. Create one branch per fix:

```sh
git fetch upstream
git switch main
git merge --ff-only upstream/main
git push origin main

git switch -c fix/ios-face-id-description
# edit and test
git add <specific-files>
git commit -m "fix: configure Face ID usage description"
git push -u origin fix/ios-face-id-description
```

Then open a pull request from:

```text
Pivii/mostro-mobile:fix/ios-face-id-description
```

into:

```text
MostroP2P/mobile:main
```

Before every pull request:

```sh
fvm flutter analyze
fvm flutter test
```

For iOS changes, also include the exact Xcode build command, device/simulator
used, and sanitized result in the pull-request description. Add screenshots only
for visible UI changes.

If upstream moves while a branch is in progress, rebase the branch rather than
merging `main` into it repeatedly:

```sh
git fetch upstream
git rebase upstream/main
git push --force-with-lease
```

Use `--force-with-lease`, never an unrestricted force push.

## Definition of the first milestone

The first useful milestone is not "published on the App Store". It is:

- a documented Flutter/Xcode toolchain;
- `main` compiling for iOS without manual source edits;
- installation and launch on a physical iPhone;
- one complete low-value Mostro flow against the intended network;
- biometric, deep-link, secure-storage, lifecycle, and notification smoke tests
  documented;
- an unsigned iOS compile gate running on every pull request;
- remaining Apple-account and TestFlight work clearly assigned to a maintainer.

Once that milestone is stable, a signed TestFlight pipeline becomes a contained
release-engineering task instead of an open-ended porting effort.

## References

- [Issue #80: Implement iOS build](https://github.com/MostroP2P/mobile/issues/80)
- [Issue #682: lucide_icons compatibility](https://github.com/MostroP2P/mobile/issues/682)
- [PR #274: Xcode and iOS target updates](https://github.com/MostroP2P/mobile/pull/274)
- [PR #275: removal of iOS release build](https://github.com/MostroP2P/mobile/pull/275)
- [Issue #339: notification implementation discussion](https://github.com/MostroP2P/mobile/issues/339)
- [Apple App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
- [TestFlight overview](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview)
