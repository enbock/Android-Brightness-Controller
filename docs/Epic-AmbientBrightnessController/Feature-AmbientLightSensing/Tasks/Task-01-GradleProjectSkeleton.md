# T0 — Gradle project skeleton

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AmbientLightSensing.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T0 — Gradle project skeleton**: create the three-module build with Kotlin DSL and a version
  catalog. Root files: `settings.gradle.kts` (`include(":app", ":core", ":infrastructure")`,
  `pluginManagement`/`dependencyResolutionManagement` with `google()` + `mavenCentral()`),
  `build.gradle.kts`, `gradle.properties`, `gradle/libs.versions.toml`, the Gradle wrapper
  (`gradlew`, `gradlew.bat`, `gradle/wrapper/**`), `.gitignore` (build output, `.gradle`,
  `local.properties`) and a machine-local `local.properties`. Per module: `:core` uses the plain
  Kotlin JVM plugin with JUnit 5 and **no Android plugin**; `:infrastructure` (`com.android.library`)
  and `:app` (`com.android.application`) declare `namespace`, `compileSdk = 36`, `minSdk = 36`,
  `targetSdk = 36` and a Java/Kotlin toolchain of 21. `:app` additionally gets a minimal
  `AndroidManifest.xml` and an empty Compose `MainActivity` so the build is verifiable end-to-end.
  No feature code, permissions or services yet. Source layout stays `:core/src/main/kotlin/…`
  (not `src/core/…`) per the recorded decision.
  *Files: `settings.gradle.kts`, `build.gradle.kts`, `gradle.properties`, `gradle/libs.versions.toml`,
  `gradle/wrapper/**`, `gradlew`, `gradlew.bat`, `.gitignore`, `local.properties`,
  `:core/build.gradle.kts`, `:infrastructure/build.gradle.kts`, `:app/build.gradle.kts`,
  `:app/src/main/AndroidManifest.xml`, `:app/src/main/kotlin/com/ambientbrightness/app/MainActivity.kt`.*
  *Verify: `./gradlew :core:test :app:assembleDebug` succeeds; a deliberately failing `:core` test
  is reported as a failure (proves the test task actually runs).*
  *Done: versions ported from the Google-validated `android create empty-activity` template —
  Gradle 9.1.0 (wrapper `-bin`, sha256 pinned), AGP 9.0.1 (`com.android.application` /
  `com.android.library`), Kotlin 2.3.20 (`org.jetbrains.kotlin.jvm` for `:core`,
  `org.jetbrains.kotlin.plugin.compose` for `:app`; AGP 9 has built-in Kotlin support, so no
  `kotlin.android` plugin), Compose BOM 2026.03.01, JUnit Jupiter 5.12.2.
  Verified with `.\gradlew.bat :core:test :app:assembleDebug --no-daemon` → BUILD SUCCESSFUL;
  `:infrastructure:assembleDebug` also BUILD SUCCESSFUL. A temporary failing
  `SkeletonFailingTest` in `:core/src/test/kotlin` made `:core:test` FAIL as expected and was
  removed afterwards (test task proven to execute). APK: `app\build\outputs\apk\debug\app-debug.apk`.
  Commit: `24e583d` (`feat(build): add Gradle skeleton for :app, :core and :infrastructure`).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.
