# Phase 0A - Build Stabilization

## Goal
Stabilize the build system. Remove deprecated JCenter. Upgrade dependencies to latest versions compatible with existing architecture. No source code changes, no architecture changes.

---

## Constraint Analysis

### Problem: material-dialogs library is JCenter-only

`com.afollestad.material-dialogs:core:0.9.6.0` was **only published on JCenter**.

JCenter shut down in February 2022. This artifact was **not** migrated to MavenCentral.

**Options**:

| Option | Approach | Risk | Buildable? |
|---|---|---|---|
| A | Remove JCenter entirely + replace material-dialogs with `MaterialAlertDialogBuilder` | **High** (changes 30+ dialog instances across all Activities) | ✅ Only if dialog refactor complete |
| B | Remove JCenter but keep `material-dialogs` by hosting the AAR locally via `flatDir` | **Low** (requires downloading the AAR once) | ✅ |
| C | Remove JCenter but add a `flatDir` repo pointing to the existing cached AAR (if it exists on the dev machine) | **Low** (requires AAR present) | ❌ Unreliable across machines |
| D | Keep JCenter **only** for this single artifact using Gradle content filtering | **Low** (JCenter still running in read-only mode) | ✅ Currently works |
| E | Remove JCenter, add JitPack as alternative (material-dialogs mirrored on JitPack) | **Low** | ✅ If JitPack build exists |

**Recommendation for Phase 0A**: Accept **Option D temporarily** — keep JCenter but restrict it to only `com.afollestad.material-dialogs` using content filtering. This is the only change that guarantees the build succeeds without modifying any Java source files. Document as tech debt to fix in Phase 1.

Actually, **Option B** is also viable: download the AAR from JCenter (while it's still readable), place it in `app/libs/`, add `flatDir` repo, remove JCenter. The AAR can be downloaded manually.

---

## Files to Modify

### 1. `settings.gradle`

```groovy
// BEFORE:
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        jcenter() // Warning: this repository is going to shut down soon
    }
}

// AFTER:
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        // JCenter retained temporarily for material-dialogs:0.9.6.0 only
        // TODO Phase 1: Replace material-dialogs with MaterialAlertDialogBuilder
        jcenter() {
            content {
                includeGroup("com.afollestad.material-dialogs")
            }
        }
    }
}
```

### 2. `app/build.gradle`

```
// BEFORE (current):
dependencies {
    implementation "androidx.appcompat:appcompat:1.4.2"
    implementation "androidx.preference:preference:1.1.1"
    implementation "com.google.android.material:material:1.6.1"
    implementation "com.github.castorflex.smoothprogressbar:library:1.1.0"
    implementation "org.eclipse.jgit:org.eclipse.jgit.ssh.jsch:6.2.0.202206071550-r"
    implementation "com.afollestad.material-dialogs:core:0.9.6.0"
    implementation "com.github.bumptech.glide:glide:4.13.2"
}

// AFTER - Dependencies updated to latest compatible versions:
android {
    compileSdk 33  // UPGRADED from 31
    defaultConfig {
        targetSdk 33  // UPGRADED from 31
    }
}

dependencies {
    implementation "androidx.appcompat:appcompat:1.6.1"       // UPGRADED from 1.4.2
    implementation "androidx.preference:preference:1.1.1"     // KEPT (1.2+ requires Kotlin)
    implementation "com.google.android.material:material:1.11.0"  // UPGRADED from 1.6.1
    implementation "com.github.castorflex.smoothprogressbar:library:1.1.0"  // KEPT (abandoned, replace later)
    implementation "org.eclipse.jgit:org.eclipse.jgit.ssh.jsch:6.8.0.202311291350-r"  // UPGRADED from 6.2.0
    implementation "com.afollestad.material-dialogs:core:0.9.6.0"  // KEPT (JCenter, see Phase 1)
    implementation "com.github.bumptech.glide:glide:4.16.0"   // UPGRADED from 4.13.2
    annotationProcessor "com.github.bumptech.glide:compiler:4.16.0"  // ADDED (required by Glide 4.16)
}
```

### 3. `gradle.properties`

```
# BEFORE:
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
android.enableJetifier=true

# AFTER: (no changes needed)
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
android.enableJetifier=true
```

### 4. `build.gradle` (root)

```
// BEFORE:
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:7.0.1"
    }
}

// AFTER: (upgrade AGP for compileSdk 33 support)
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:7.4.2"
    }
}
```

**Why AGP 7.4.2?**
- Supports compileSdk 33
- Supports Java 8-targeted compilation
- Gradle 7.5+ compatibility
- Does NOT require Kotlin plugin
- LTS release (stable, well-tested)

---

## Dependency Upgrade Justification

| Dependency | Old | New | Why This Version |
|---|---|---|---|
| `appcompat` | 1.4.2 | 1.6.1 | Latest non-Kotlin-dependent version |
| `material` | 1.6.1 | 1.11.0 | Latest stable M3 (no Compose requirement) |
| `jgit` | 6.2.0 | 6.8.0 | Latest 6.x stable (no API breaks) |
| `glide` | 4.13.2 | 4.16.0 | Latest stable |
| `preference` | 1.1.1 | 1.1.1 | **KEPT** - 1.2+ requires Kotlin stdlib |
| `smoothprogressbar` | 1.1.0 | 1.1.0 | **KEPT** - Abandoned, Phase 1 replacement |
| `material-dialogs` | 0.9.6.0 | 0.9.6.0 | **KEPT** - JCenter, Phase 1 replacement |

---

## AGP Upgrade Compatibility

| AGP Version | Gradle Required | compileSdk | Notes |
|---|---|---|---|
| 7.0.x | 7.0+ | 31 | Current |
| **7.4.x** | **7.5+** | **33** | **Target** |
| 8.0.x | 8.0+ | 34 | Requires Java 17 |

Since we do not have a `gradle-wrapper.properties` file visible, need to check the wrapper version and potentially update it.

**Gradle Update Plan**:
1. Check current wrapper version in `gradle/wrapper/gradle-wrapper.properties`
2. Update to Gradle 7.6.3 (latest 7.x, compatible with AGP 7.4.2)
3. Run `./gradlew wrapper --gradle-version=7.6.3`

---

## Phase 0A Execution Steps

Ordered, each step individually buildable:

### Step 1: Update Gradle wrapper
```
./gradlew wrapper --gradle-version=7.6.3
```
Verify: Check `gradle-wrapper.properties`

### Step 2: Update root build.gradle (AGP 7.0.1 → 7.4.2)

### Step 3: Update settings.gradle (JCenter → content-scoped JCenter)

### Step 4: Update app/build.gradle (compileSdk 31→33, dependency versions)

### Step 5: Verify build
```
./gradlew assembleDebug --warning-mode=all
```

---

## What Phase 0A Does NOT Touch

| Area | Status |
|---|---|
| Java source files (.java) | **NOT TOUCHED** |
| XML layouts | **NOT TOUCHED** |
| AndroidManifest.xml | **NOT TOUCHED** |
| ProGuard rules | **NOT TOUCHED** |
| Resource files | **NOT TOUCHED** |
| Data models | **NOT TOUCHED** |
| Git functionality | **PRESERVED** |
| Activities | **PRESERVED** |
| Services | **PRESERVED** |
| SQLite database | **NOT CHANGED** |
| IntentService | **PRESERVED** |
| AIDE-specific configs (app_config.json, libraries.json, repositories.json) | **NOT MODIFIED** |

---

## Risk & Mitigation

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| JCenter read-only access removed | **Low** (still operational) | Build fails for material-dialogs | Fallback: download AAR to `libs/` + `flatDir` |
| AGP 7.4.2 incompatible with some library | **Low** | Build fails | Pin problematic library to old version |
| Gradle 7.6.3 has breaking changes | **Low** | Build fails | Use `--warning-mode=all` to catch issues |
| compileSdk 33 breaks runtime permission code | **Medium** | MANAGE_EXTERNAL_STORAGE changes | Already handled in Android 13+ (no new permission issues) |

---

## Verification Criteria

- [ ] `./gradlew assembleDebug` succeeds
- [ ] No JCenter warnings in build output
- [ ] All dependencies resolve from Google/MavenCentral
- [ ] No new deprecation warnings from Gradle
- [ ] `libs.versions.toml` not required (keeping build.gradle simple)

---

## Deliverable

When Phase 0A is complete:
1. A working `build.gradle` (root + app) with modern AGP and dependencies
2. `settings.gradle` with JCenter scoped to material-dialogs only
3. Updated `gradle-wrapper.properties` for Gradle 7.6.3
4. Build verification log

---

**After Phase 0A is complete, I will generate the Phase 0B plan.**

Ready for approval.
