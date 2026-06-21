# Pocket Git - Comprehensive Analysis Report

## PROJECT OVERVIEW

- **App Name**: Pocket Git
- **Package**: `com.aor.pocketgit`
- **Version**: 1.6 (code 160)
- **Type**: Android Git Client
- **Language**: Java (no Kotlin)
- **Min SDK**: 16 | **Target SDK**: 31
- **Build System**: Gradle 7.0.1
- **Git Library**: JGit 6.2.0 + JSch

---

## FOLDER STRUCTURE

```
pocket-git-master/
├── .github/workflows/android.yml       # CI/CD pipeline (Gradle build + APK upload)
├── app/
│   ├── build.gradle                    # App module dependencies & config
│   ├── app_config.json                 # AIDE IDE build configuration
│   ├── libraries.json                  # Dependency manifest
│   ├── repositories.json               # Maven repository URLs
│   ├── proguard-rules.pro              # Empty ProGuard rules
│   └── src/main/
│       ├── AndroidManifest.xml         # Permissions, activities, services
│       ├── java/com/aor/pocketgit/
│       │   ├── activities/             # 14 Activities (UI screens)
│       │   │   ├── MainActivity.java          # Project list + clone management
│       │   │   ├── ProjectActivity.java       # Add/edit project config
│       │   │   ├── FilesActivity.java         # File browser + git operations (1300+ lines)
│       │   │   ├── CommitActivity.java        # Commit details + diff
│       │   │   ├── DiffActivity.java          # Unified diff viewer
│       │   │   ├── BlameActivity.java         # Line-by-line blame view
│       │   │   ├── LogActivity.java           # Commit graph with branch lanes
│       │   │   ├── RemotesActivity.java       # Manage remote repositories
│       │   │   ├── RemoteActivity.java        # Add/edit remote
│       │   │   ├── HelpActivity.java          # Help screen
│       │   │   ├── PreferencesActivity.java   # Settings wrapper
│       │   │   ├── PreferencesFragment.java   # Settings fragment (legacy)
│       │   │   ├── PickerActivity.java        # File/folder chooser
│       │   │   └── UpdatableActivity.java     # Base class with broadcast receiver
│       │   ├── adapters/              # 15 ListView Adapters
│       │   │   ├── ProjectAdapter.java
│       │   │   ├── GitFileAdapter.java
│       │   │   ├── CommitAdapter.java
│       │   │   ├── BranchAdapter.java / BranchArrayAdapter.java
│       │   │   ├── DiffAdapter.java / DiffLineAdapter.java
│       │   │   ├── BlameLineAdapter.java
│       │   │   ├── FolderAdapter.java
│       │   │   ├── RemoteAdapter.java
│       │   │   ├── RefSpecAdapter.java
│       │   │   ├── StashAdapter.java
│       │   │   └── TagAdapter.java
│       │   ├── application/
│       │   │   └── PocketGit.java             # Application class (notification channels)
│       │   ├── data/                  # 4 Data models
│       │   │   ├── Project.java               # Git repository entry
│       │   │   ├── GitFile.java               # File with git states
│       │   │   ├── DiffLine.java              # Diff text line
│       │   │   ├── BlameLine.java             # Blame annotation
│       │   │   └── TypedRefSpec.java          # Remote refspec
│       │   ├── database/              # SQLite persistence
│       │   │   ├── PocketDbHelper.java        # Schema definition (v2)
│       │   │   └── ProjectsDataSource.java    # CRUD operations
│       │   ├── dialogs/               # 5 Custom dialog handlers
│       │   │   ├── CheckoutRemoteDialog.java
│       │   │   ├── MergeDialog.java
│       │   │   ├── SortFilesDialog.java
│       │   │   └── TagListDialog.java
│       │   ├── services/              # 4 Background IntentServices
│       │   │   ├── GitService.java            # Base class (notifications, broadcast)
│       │   │   ├── GitClone.java              # Clone + pull fallback
│       │   │   ├── GitPull.java               # Pull from remote
│       │   │   ├── GitPush.java               # Push to remote
│       │   │   └── GitFetch.java              # Fetch from remote
│       │   ├── tasks/                 # 2 AsyncTasks
│       │   │   ├── CheckoutLocalTask.java
│       │   │   └── CheckoutRemoteTask.java
│       │   ├── utils/                 # 8 Utility classes
│       │   │   ├── GitUtils.java              # Central git operations (442 lines)
│       │   │   ├── CredentialStorage.java      # In-memory credential cache
│       │   │   ├── FileUtils.java              # Recursive file search
│       │   │   ├── FileComparator.java         # Sort comparator
│       │   │   ├── FontUtils.java              # Roboto font helper
│       │   │   ├── MD5Util.java                # MD5 for Gravatar
│       │   │   └── GitIgnore.java
│       │   └── widgets/               # 3 Custom Views
│       │       ├── FloatingActionButton.java
│       │       ├── FlowLayout.java
│       │       └── PlotLaneView.java
│       └── res/
│           ├── drawable/              # 3 XML drawables
│           ├── drawable-anydpi-v21/   # 50+ vector icons
│           ├── drawable-hdpi-mdpi-xhdpi-xxhdpi/  # Raster assets
│           ├── layout/                # 44 layout files
│           ├── layout-v21/            # Material-themed layouts
│           ├── menu/                  # 8 menu XML files
│           ├── values/
│           │   ├── arrays.xml          # Auth type options
│           │   ├── colors.xml          # Material color palette
│           │   ├── strings.xml         # 135 string resources
│           │   └── themes.xml          # Material3 light theme
│           ├── values-night/           # Dark theme variant
│           └── xml/
│               └── preferences.xml     # Settings screen
├── build.gradle                        # Root Gradle config
├── settings.gradle                     # Project name + repositories
├── gradle.properties                   # JVM args + AndroidX flags
├── settings.json                       # IDE state
└── gradlew / gradlew.bat               # Gradle wrapper
```

---

## MODULE DEPENDENCIES

### Data Layer
```
Project.java ─────────────────────────────────────────────┐
  ├── used by: MainActivity, ProjectActivity, FilesActivity, │
  │   LogActivity, CommitActivity, GitService, GitUtils     │
  ├── Project.NONE=0, PASSWORD=1, PRIVATE_KEY=2            │
  ├── STATE_CLONED, STATE_CLONING, STATE_FAILED, STATE_UNKNOWN│
  └── Stores in plaintext: username, password, privateKeyPath│

GitFile.java ─────────────────────────────────────────────┐
  ├── states: ADDED, MODIFIED, COMMITED, REMOVED, CONFLICTING,│
  │   IGNORED, MISSING, UNTRACKED, UNCOMMITED, CHANGED,    │
  │   UNTRACKED_NOT_REMOVED                                │
  └── Static methods: isStageable(), isUnstageable(), ...  │
```

### Database Layer
```
PocketDbHelper (SQLiteOpenHelper v2)
  └── TABLE projects: id, name, url, local_path, authentication,
                      username, password, privatekey, state

ProjectsDataSource ────────────────────────────────┐
  ├── CRUD: createProject, updateProject, deleteProject│
  ├── Queries: getProject, getProjectFromURL, getAllProjects│
  ├── updatePassword, updateState                     │
  └── ⚠ Plaintext credential storage (no encryption)  │
```

### Services Layer
```
GitService (abstract IntentService)
  ├── onHandleIntent() -> loads Project by ID
  ├── startNotification() -> foreground service notification
  ├── updateNotification() -> progress updates
  ├── finishNotification*() -> completion/error notifications
  └── broadcastMessage() -> UPDATE_BROADCAST intent

GitClone ────────────────────────────────────┐
  ├── Normal flow: GitUtils.cloneRepository()│
  ├── Certificate error: use_pull=true retry │
  └── Updates state: CLONING -> CLONED/FAILED│

GitPull   ── GitUtils.pullRepository()
GitPush   ── GitUtils.pushRepository()
GitFetch  ── GitUtils.fetchRepository()
```

### Git Operations Hub (GitUtils.java)
```
cloneRepository(project, service)
pullRepository(project, service, remote)
fetchRepository(project, service, remote)
pushRepository(project, service, remote, pushTags)
stageFiles(project, files)
unstageFiles(project, files)
removeFiles(project, files)
revertFiles(project, files)
revert(project)                     # Hard reset HEAD
setCredentials(project, command)    # Auth setup (SSH/PW)
getRepository(project)              # Repository object
getDiffList(repo, commit, walk)     # Diff entries
getDiffText(repo, entry)            # Unified diff
prepareTreeParser(repo, objectId)   # Tree iterator
formatDate(time)                    # Timestamp formatter
```

### UI Navigation Flow
```
MainActivity (Project List)
  ├── [+] FAB → ProjectActivity (Create)
  ├── Long-press → CAB: Edit ProjectActivity / Delete
  └── Project Click → check state
       ├── CLONING → clone dialog
       ├── CLONED → FilesActivity
       ├── FAILED → clone dialog
       └── no URL → toast

FilesActivity (File Browser + Git Ops)
  ├── Drawer: Branch list (local/remote toggle)
  ├── Menu: branch ops, remote ops, stash, tag, log, author
  ├── CAB (file selection): stage, unstage, diff, blame, revert, delete, remove
  ├── FAB: commit dialog
  ├── Pull/Push/Fetch → IntentService → Notification
  ├── Log → LogActivity → CommitDetail → DiffActivity
  ├── New File/Folder → dialog
  └── Sort → SortFilesDialog
```

---

## PERMISSION AUDIT

| Permission | Declaration | Runtime Check | Required | Risk |
|---|---|---|---|---|
| `READ_EXTERNAL_STORAGE` | `AndroidManifest.xml:9` | `MainActivity.java:295` | Yes | Low |
| `WRITE_EXTERNAL_STORAGE` | `AndroidManifest.xml:10` | `MainActivity.java:296` | Yes | Medium |
| `MANAGE_EXTERNAL_STORAGE` | `AndroidManifest.xml:11` | **NONE** | Yes | **CRITICAL** |
| `FOREGROUND_SERVICE` | `AndroidManifest.xml:12` | N/A | Yes | Low |
| `INTERNET` | `AndroidManifest.xml:13` | N/A | Yes | Medium |

### Issues
1. `MANAGE_EXTERNAL_STORAGE` declared but **never requested at runtime** → app will crash on Android 11+ when user selects custom directory
2. No SAF (Storage Access Framework) integration; uses raw file paths
3. `android:requestLegacyExternalStorage="true"` - temporary Android 10 workaround, ignored on Android 11+

---

## SECURITY AUDIT

### Critical Severity

| Issue | Location | Root Cause | Fix |
|---|---|---|---|
| Plaintext credentials in SQLite | `ProjectsDataSource.java:51-56`, `PocketDbHelper.java:16` | No encryption for password/privatekey columns | Use EncryptedSharedPreferences or SQLCipher |
| SSH StrictHostKeyChecking disabled | `GitUtils.java:86,124,143` | `session.setConfig("StrictHostKeyChecking", "no")` on ALL connections | Enable host key verification + known_hosts |
| No HTTPS certificate validation | `GitClone.java:41` | `sslVerify=false` in config | Remove sslVerify override; implement proper SSL |
| Private keys stored in DB path | `PocketDbHelper.java:16` | `privatekey VARCHAR` column stores file path | Store in Android Keystore |
| Password visible in activity transitions | `MainActivity.java:91-92` | Passwords passed as Intent extras | Use encrypted tokens or secure channel |

### Medium Severity

| Issue | Location | Detail |
|---|---|---|
| Deprecated JCenter | `settings.gradle:6` | Repository shut down since Feb 2022 |
| Gravatar over HTTP | `CommitActivity.java:75` | `http://www.gravatar.com/avatar/` - should be HTTPS |
| No ProGuard/R8 | `build.gradle:18` | `minifyEnabled false`, empty proguard rules |
| `abortOnError false` | `build.gradle:25` | Lint warnings suppressed |
| Static mutable state | `GitClone.java:19-20` | `sRunning`, `sCurrent` static variables in service |
| In-memory credential cache | `CredentialStorage.java:25-26` | Static HashMap never cleared; persists forever |

---

## PERFORMANCE ANALYSIS

### Issues

1. **Blocking I/O on UI Thread**: `FilesActivity.getRepository()`, `GitUtils.getRepository()` called on main thread via `getFiles()`, `initializeBranches()`, `refreshFiles()`
2. **Unbounded Commit Loading**: `LogActivity.java:77` - `plotCommitList.fillTo(Integer.MAX_VALUE)` loads ALL commits into memory
3. **AsyncTask Deprecated**: `FilesActivity.refreshStatus()` and `LogActivity.initializeList()` use deprecated Android API
4. **No RecyclerView**: Uses `ListView` (inefficient for large lists, no view holder pattern enforced)
5. **Status Recalculation**: `calculateFileStates()` loops through all file sets on every refresh
6. **Repository Reopening**: `getRepository()` creates new `RepositoryBuilder()` on every call - no caching
7. **Notification Throttling**: `GitService.updateNotification()` properly throttles to 1s intervals ✓

### Potential Leaks
- `BroadcastReceiver` in `UpdatableActivity`: properly registered in `onResume()`, unregistered in `onPause()` ✓
- `GitService` static fields persist across service restarts
- `FileTreeIterator`, `RevWalk`, `DiffFormatter` - used in try blocks but some lack `close()` in catch blocks

---

## BUG HUNT

### Logic Errors

| File | Line | Issue |
|---|---|---|
| `FilesActivity.java` | 723 | `UNCOMMITED` → should be `UNCOMMITTED` (typo) |
| `GitClone.java` | 53 | `BranchConfig.LOCAL_REPOSITORY` appended as literal (`"..." + LOCAL_REPOSITORY`) - display glitch |
| `GitFetch.java` | 14 | Copy-paste: constructor says `"Git Pull Service"` instead of `"Git Fetch Service"` |
| `Project.java` | 31-35 | URL parsing fragile; assumes `@` is credential separator |
| `FilesActivity.java` | 741 | `contains(this.mStatus.getUntracked(), file) && !contains(this.mStatus.getRemoved(), file)` - always false for single file |
| `FilesActivity.java` | 344-368 | `actionDiffFile()` opens repository but only closes on success path; leaks on exception |

### Edge Cases

- **Empty repository**: `initializeBranches()` calls `optionCreateBranch()` when `currentBranch == null`, but no initial commit flow
- **`.gitignore` parse errors**: Caught as `PatternSyntaxException` in `refreshStatus()` but only shows snackbar
- **Concurrent service access**: No synchronization on `ProjectsDataSource.database`
- **Deep file paths**: `getFilePattern()` may throw `StringIndexOutOfBoundsException` if path manipulation fails
- **No submodule support**: JGit submodule commands not utilized
- **No conflict resolution**: Merge conflicts detected but no resolution UI (only toast)
- **SSH key without passphrase**: `JSch.addIdentity(privateKey, passphrase)` called even when passphrase is empty

---

## DEPENDENCY ANALYSIS

### Current Dependencies

| Artifact | Version | Status | Recommendation |
|---|---|---|---|
| `com.android.tools.build:gradle` | 7.0.1 | EOL (2022) | Upgrade to 8.x |
| `androidx.appcompat:appcompat` | 1.4.2 | Old | Upgrade to 1.7.x |
| `androidx.preference:preference` | 1.1.1 | Old (pre-Kotlin) | Upgrade to 1.2.1 |
| `com.google.android.material:material` | 1.6.1 | Old | Upgrade to 1.12.x |
| `org.eclipse.jgit:org.eclipse.jgit.ssh.jsch` | 6.2.0 | Current (2022) | Consider 6.8+ |
| `com.afollestad.material-dialogs:core` | 0.9.6.0 | **ABANDONED** (2019) | Migrate to Material3 dialogs |
| `com.github.castorflex.smoothprogressbar` | 1.1.0 | **ABANDONED** (2017) | Use Material3 progress |
| `com.github.bumptech.glide:glide` | 4.13.2 | Old | Upgrade to 4.16+ |

### Build Configuration Issues

- **JCenter** (`settings.gradle:6`): Shut down February 2022 - builds will fail
- **No version catalog**: Use `libs.versions.toml` for centralized management
- **compileSdk 31**: Should be 34 (current)
- **Java 8 only**: Consider Java 11/17 for modern features
- **No testing dependencies**: Zero test coverage

---

## ARCHITECTURAL OBSERVATIONS

### Strengths
- IntentService pattern correctly separates background git operations from UI
- Broadcast-based UI updates decouple services from activities
- Git operations centralized in `GitUtils` (single responsibility)
- Consistent adapter pattern for ListViews
- Material Design theme with vector icons

### Weaknesses
- **FilesActivity is a god class** (1300+ lines): mixes file browsing, git operations, branch management, stash/tag handling, authentication
- **No dependency injection**: Manual `new` calls everywhere; untestable
- **No MVVM/MVP**: Activities handle all business logic, data access, and UI updates
- **No Repository pattern**: `ProjectsDataSource` called directly from each activity
- **Static utility methods**: `GitUtils` static methods are untestable, cannot be mocked
- **Legacy patterns**: IntentService deprecated in API 30, AsyncTask deprecated in API 30
- **No navigation component**: Manual Intent extras with string keys (no type safety)

---

## CHANGELOG / DIFF

- Repository has **no git history** (not a git repo)
- `versionCode=160`, `versionName=1.6` suggests iterative releases
- Database v2 migration added `state` column (for clone tracking)

---

## RECOMMENDED ACTIONS

### P0 - Security & Compatibility (Do First)
1. **Encrypt stored credentials** using EncryptedSharedPreferences + Android Keystore
2. **Migrate to Scoped Storage** (SAF / MediaStore), remove `MANAGE_EXTERNAL_STORAGE`
3. **Enable SSH host key verification** with configurable known_hosts
4. **Remove JCenter** dependency (will break build)

### P1 - Build & Maintainability
1. **Upgrade Gradle** to 8.x + AGP 8.x
2. **Update all dependencies** to latest stable versions
3. **Replace abandoned libraries**: material-dialogs, smoothprogressbar
4. **Add ProGuard/R8** rules for release builds

### P2 - Modernization
1. **Refactor `FilesActivity`** into smaller ViewModel + Repository + Fragments
2. **Replace IntentService** with WorkManager or coroutines
3. **Replace AsyncTask** with coroutines / Flow
4. **Migrate ListView** to RecyclerView
5. **Add Navigation Component** for type-safe routing

### P3 - Quality
1. **Write unit tests** for GitUtils logic
2. **Add UI tests** (Espresso) for critical paths
3. **Set up CI linting** (detekt, ktlint, checkstyle)
4. **Add crash reporting** (Firebase Crashlytics)

---

## SUMMARY

Pocket Git is a functional Android Git client built in 2019-2022 era. It provides essential Git operations (clone, pull, push, fetch, commit, branch, stash, tag, diff, blame, log) through a clean but dated codebase. The architecture follows straightforward Activity + Service + SQLite pattern without modern Android architecture components.

**Critical issues** center on plaintext credential storage, disabled SSL/SSH security checks, and MANAGE_EXTERNAL_STORAGE dependency. The build system uses deprecated JCenter and old Gradle/AGP versions. The codebase has zero test coverage and several bugs (typos, copy-paste errors, resource leak paths).

**Recommendation**: Address P0 security issues immediately, then progressively modernize the build system and architecture to ensure long-term maintainability and compatibility with evolving Android platforms.
