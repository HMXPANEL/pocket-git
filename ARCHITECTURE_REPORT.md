# Pocket Git - Complete Architecture Report & Transformation Plan

> Generated: 2026-06-21
> Analysis Scope: 100% of 55 Java source files, 129 XML files, all build/config files

---

## TABLE OF CONTENTS

1. [Complete Dependency Graph](#1-complete-dependency-graph)
2. [Class-Level Dependency Map](#2-class-level-dependency-map)
3. [Navigation & Execution Flow](#3-navigation--execution-flow)
4. [Database Architecture](#4-database-architecture)
5. [JGit Integration Map](#5-jgit-integration-map)
6. [Authentication System](#6-authentication-system)
7. [Repository Management Flow](#7-repository-management-flow)
8. [File Browser Architecture](#8-file-browser-architecture)
9. [Security Vulnerability Matrix](#9-security-vulnerability-matrix)
10. [Dead & Duplicate Code](#10-dead--duplicate-code)
11. [Performance Bottlenecks](#11-performance-bottlenecks)
12. [Deprecated Android APIs](#12-deprecated-android-apis)
13. [Modernization Opportunities](#13-modernization-opportunities)
14. [Pocket Git X Transformation Strategy](#14-pocket-git-x-transformation-strategy)
15. [Implementation Phases](#15-implementation-phases)

---

## 1. COMPLETE DEPENDENCY GRAPH

### Layer Diagram (Top to Bottom)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          UI LAYER (Activities)                           │
│  MainActivity  ProjectActivity  FilesActivity  LogActivity              │
│  CommitActivity  DiffActivity  BlameActivity  HelpActivity               │
│  RemotesActivity  RemoteActivity  PickerActivity                         │
│  PreferencesActivity  PreferencesFragment                                │
│  UpdatableActivity (base)                                                │
├──────────────────────────────────────────────────────────────────────────┤
│                        ADAPTERS (13 files)                               │
│  ProjectAdapter  GitFileAdapter  CommitAdapter  BranchAdapter            │
│  BranchArrayAdapter  DiffAdapter  DiffLineAdapter                       │
│  BlameLineAdapter  FolderAdapter  RemoteAdapter  RefSpecAdapter          │
│  StashAdapter  TagAdapter                                               │
├──────────────────────────────────────────────────────────────────────────┤
│                   CUSTOM WIDGETS (3 files)                               │
│  FloatingActionButton  FlowLayout  PlotLaneView                          │
├──────────────────────────────────────────────────────────────────────────┤
│                      DIALOGS (4 files)                                   │
│  CheckoutRemoteDialog  MergeDialog  SortFilesDialog  TagListDialog       │
├──────────────────────────────────────────────────────────────────────────┤
│                         TASKS (2 files)                                  │
│  CheckoutLocalTask (AsyncTask)  CheckoutRemoteTask (AsyncTask)           │
├──────────────────────────────────────────────────────────────────────────┤
│                     BACKGROUND SERVICES (5 files)                        │
│  GitService (abstract IntentService)                                     │
│  GitClone : GitService  GitPull : GitService                             │
│  GitPush : GitService  GitFetch : GitService                            │
├──────────────────────────────────────────────────────────────────────────┤
│                    DATA LAYER (5 files)                                  │
│  Project  GitFile  DiffLine  BlameLine  TypedRefSpec                     │
├──────────────────────────────────────────────────────────────────────────┤
│                      DATABASE (2 files)                                  │
│  PocketDbHelper (SQLiteOpenHelper)                                       │
│  ProjectsDataSource (CRUD)                                               │
├──────────────────────────────────────────────────────────────────────────┤
│                  UTILITY LAYER (6 files)                                 │
│  GitUtils  CredentialStorage  FileUtils  FileComparator                  │
│  FontUtils  MD5Util                                                     │
├──────────────────────────────────────────────────────────────────────────┤
│              APPLICATION LIFECYCLE (1 file)                              │
│  PocketGit (Application)                                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                     EXTERNAL DEPENDENCIES                                │
│  JGit 6.2.0  JSch  AndroidX  Material Components                        │
│  Material Dialogs  SmoothProgressBar  Glide                              │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 2. CLASS-LEVEL DEPENDENCY MAP

### 2.1 Activity → Service Dependencies

```
MainActivity
  └──→ GitClone (IntentService)
  └──→ ProjectsDataSource
  └──→ CredentialStorage

ProjectActivity
  └──→ PickerActivity (via startActivityForResult)
  └──→ FontUtils

FilesActivity (1889 lines - GOD CLASS)
  ├──→ GitPull, GitPush, GitFetch (IntentServices)
  ├──→ GitUtils (static)
  ├──→ GitFileAdapter
  ├──→ BranchAdapter
  ├──→ RemoteAdapter
  ├──→ StashAdapter
  ├──→ CheckoutLocalTask, CheckoutRemoteTask
  ├──→ CheckoutRemoteDialog, MergeDialog, SortFilesDialog, TagListDialog
  ├──→ CredentialStorage
  ├──→ FileComparator
  ├──→ FileUtils
  ├──→ FontUtils
  └──→ ProjectsDataSource

RemotesActivity
  ├──→ GitUtils
  ├──→ RemoteAdapter
  └──→ FloatingActionButton

RemoteActivity
  ├──→ GitUtils
  ├──→ RefSpecAdapter
  └──→ FloatingActionButton

CommitActivity
  ├──→ GitUtils
  ├──→ DiffAdapter
  └──→ MD5Util

LogActivity
  ├──→ CommitAdapter
  └──→ ProjectsDataSource

DiffActivity
  └──→ DiffLineAdapter

BlameActivity
  ├──→ GitUtils
  ├──→ BlameLineAdapter
  └──→ ProjectsDataSource
```

### 2.2 GitService Inheritance

```
GitService (abstract IntentService)
  ├── Fields: mId, mProject, mBuilder, mNotifyManager, lastTime
  ├── Methods: onHandleIntent(), startNotification(), updateNotification(),
  │           finishNotification(), broadcastMessage(), createPendingIntent()
  │
  ├── GitClone (110 lines)
  │   ├── override onHandleIntent()
  │   ├── sRunning (static), sCurrent (static)
  │   ├── cloneRepository() or pullRepository() fallback
  │   └── containsException() recursive search
  │
  ├── GitPull (43 lines)
  │   └── override onHandleIntent() → pullRepository()
  │
  ├── GitPush (50 lines)
  │   └── override onHandleIntent() → pushRepository()
  │
  └── GitFetch (43 lines)
      └── override onHandleIntent() → fetchRepository()
```

### 2.3 GitUtils Static Methods Called By

| Method | Called From |
|--------|------------|
| `repositoryCloned()` | MainActivity |
| `cloneRepository()` | GitClone |
| `pullRepository()` | GitClone, GitPull |
| `fetchRepository()` | GitFetch |
| `pushRepository()` | GitPush |
| `stageFiles()` | FilesActivity |
| `unstageFiles()` | FilesActivity |
| `removeFiles()` | FilesActivity |
| `revertFiles()` | FilesActivity |
| `revert()` | FilesActivity |
| `getRepository()` | FilesActivity, GitUtils (internal), RemoteActivity, RemotesActivity, BlameActivity, CommitActivity, LogActivity |
| `setCredentials()` | GitUtils (internal: clone, pull, fetch, push) |
| `getDiffList()` | CommitActivity |
| `getDiffText()` | CommitActivity |
| `prepareTreeParser()` | FilesActivity (actionDiffFile) |
| `formatDate()` | CommitActivity, StashAdapter |
| `formatDateShort()` | CommitAdapter |
| `getTags()` | TagListDialog |

---

## 3. NAVIGATION & EXECUTION FLOW

### 3.1 App Startup Flow

```
Application.onCreate()
  └── StrictMode.setVmPolicy() - disables all VM policy violations
  └── Create NotificationChannels (START, FINISHED, ERROR)

MainActivity.onCreate()
  ├── setupFAB() → create FloatingActionButton
  ├── setupProjectList() → ListView with MultiChoiceModeListener
  ├── setupProjectClick() → openProject() or clone
  ├── checkPermissions() → REQUEST_EXTERNAL_STORAGE (READ+WRITE)
  ├── handleIntent() → SEND/VIEW intent parsing → URL parsing
  └── Intent parsing supports: github.com, bitbucket.org

MainActivity.onStart()
  ├── refreshProjects() → datasource.getAllProjects()
  └── checkPermissions() again
```

### 3.2 Repository Lifecycle

```
CREATE (ProjectActivity)
  │
  ├── User enters: name, URL, localPath, auth type, username, password, privateKey
  │
  ▼
Save → MainActivity.onActivityResult() → datasource.createProject(...)
  │
  ├── State = "UNKNOWN"
  │
  ▼
Project Click → openProject()
  │
  ├── If CLONING + isCloning() → toast "Project is cloning"
  │
  ├── If CLONING + !isCloning() → prepareCloneRepository() → confirm → clone
  │
  ├── If repositoryCloned() + not FAILED → FilesActivity
  │
  ├── If URL empty → toast "Project does not have an URL"
  │
  └── Else → prepareCloneRepository() → CredentialStorage.checkCredentials()
        │
        ▼
      GitClone IntentService
        ├── [state = CLONING]
        ├── GitUtils.cloneRepository() (or pull fallback)
        │   ├── setCredentials(project, cloneCommand)
        │   │   ├── AuthNONE → setUnknownHost()
        │   │   ├── AuthPASSWORD → setSSHPassword() + UsernamePasswordCredentialsProvider
        │   │   └── AuthPRIVATEKEY → setPrivateKey()
        │   └── clone.call()
        ├── [state = CLONED or FAILED]
        ├── Broadcast: CLONE_UPDATE_BROADCAST
        └── Notification: finished/failed

EDIT (ProjectActivity in edit mode)
  │
  └── MainActivity.onActivityResult() → datasource.updateProject()

DELETE (Contextual Action Bar)
  │
  └── datasource.deleteProject() + optionally deleteAllFilesIn(localPath)
```

### 3.3 FilesActivity Navigation Map

```
FilesActivity.onCreate()
  ├── setupFAB() → commit FAB (hidden until staged changes exist)
  ├── onNewIntent() → load project, set current folder, refresh files
  ├── setupFileList() → ListView with MultiChoiceModeListener
  ├── setupParentFolderClick()
  └── setupDrawer() → ActionBarDrawerToggle with branch list

FILES OPERATIONS (from CAB):
  Stage     → GitUtils.stageFiles() → refresh
  Unstage   → GitUtils.unstageFiles() → refresh
  Diff      → GitUtils.prepareTreeParser() + DiffFormatter → DiffActivity
  Blame     → Intent → BlameActivity
  Revert    → GitUtils.revertFiles() → refresh
  Delete    → File.delete() recursively → refresh
  Remove    → GitUtils.removeFiles() (git rm --cached) → refresh

REMOTE OPERATIONS (from menu):
  Pull      → CredentialStorage.check() → GitPull IntentService
  Push      → CredentialStorage.check() → GitPush IntentService
  Fetch     → CredentialStorage.check() → GitFetch IntentService
  Manage    → RemotesActivity

BRANCH OPERATIONS (from menu):
  Create    → dialog → CreateBranchCommand → checkout
  Delete    → dialog → DeleteBranchCommand.setForce(true)
  Merge     → MergeDialog → MergeCommand
  Revert    → ResetCommand.ResetType.HARD (twice!)

STASH OPERATIONS (from menu):
  Save      → StashCreateCommand (optional includeUntracked)
  List      → StashListCommand → dialog
  Apply     → StashApplyCommand → optionally StashDropCommand
  Drop      → StashDropCommand

TAG OPERATIONS (from menu):
  Create    → TagCommand (optional message)
  List      → TagListDialog → checkout or delete
  Push      → PushCommand.setPushTags()

OTHER:
  Log       → LogActivity
  Author    → dialog → StoredConfig.setString()
  New File  → File.createNewFile()
  New Folder→ File.mkdir()
  Sort      → SortFilesDialog → FileComparator
  Search    → FileUtils.searchFiles() AsyncTask → filter adapter
  Commit    → commit() → if no author/email → author dialog → Git.commit()
```

---

## 4. DATABASE ARCHITECTURE

### 4.1 Schema (PocketGit.db, version 2)

```sql
CREATE TABLE projects (
    id              INTEGER PRIMARY KEY,
    name            VARCHAR,
    url             VARCHAR,
    local_path      VARCHAR,
    authentication  INTEGER,    -- 0=NONE, 1=PASSWORD, 2=PRIVATE_KEY
    username        VARCHAR,    -- plaintext
    password        VARCHAR,    -- ⚠ PLAINTEXT
    privatekey      VARCHAR,    -- ⚠ PLAINTEXT file path
    state           VARCHAR     -- UNKNOWN, CLONING, CLONED, FAILED
);
```

### 4.2 Migration History

```
Version 1 → 2: added "state" column, set all existing rows to "UNKNOWN"
```

### 4.3 DataSource Methods

| Method | SQL | Purpose |
|--------|-----|---------|
| `getAllProjects()` | SELECT | Sorted by name COLLATE NOCASE |
| `getProject(id)` | SELECT WHERE id=? | Single project lookup |
| `getProjectFromURL(url)` | SELECT WHERE url=? | Find by URL (dedup check) |
| `createProject(...)` | INSERT | New project |
| `updateProject(...)` | UPDATE WHERE id=? | Edit project |
| `deleteProject(id)` | DELETE WHERE id=? | Remove project |
| `updatePassword(id, pw)` | UPDATE password | Save password from dialog |
| `updateState(id, state)` | UPDATE state | Cloning/failed/cloned |

### 4.4 Security Issues

1. **Passwords stored in SQLite plaintext** - no encryption
2. **Private key file paths stored** - not the key itself, but path reveals user structure
3. **No ContentProvider** - direct file access only
4. **No database encryption** (no SQLCipher)
5. **No database locking** - `ProjectsDataSource` is not thread-safe

---

## 5. JGIT INTEGRATION MAP

### 5.1 JGit API Usage

| JGit Class | Usage Location | Purpose |
|---|---|---|
| `Git.cloneRepository()` | GitUtils.cloneRepository() | Clone remote repo |
| `Git.pull()` | GitUtils.pullRepository() | Pull from remote |
| `Git.fetch()` | GitUtils.fetchRepository() | Fetch from remote |
| `Git.push()` | GitUtils.pushRepository() | Push to remote |
| `Git.add()` | GitUtils.stageFiles() | Stage files |
| `Git.reset()` | GitUtils.unstageFiles(), GitUtils.revert() | Unstage / hard reset |
| `Git.rm()` | GitUtils.removeFiles() | Remove from index |
| `Git.checkout()` | GitUtils.revertFiles(), CheckoutLocalTask, CheckoutRemoteTask | Checkout / revert |
| `Git.branchCreate()` | FilesActivity.optionCreateBranch() | Create branch |
| `Git.branchDelete()` | FilesActivity.actionDeleteBranch() | Delete branch |
| `Git.branchList()` | FilesActivity.initializeBranches(), MergeDialog | List branches |
| `Git.commit()` | FilesActivity.commit() | Create commit |
| `Git.merge()` | MergeDialog | Merge branches |
| `Git.stashCreate()` | FilesActivity.optionStashCreate() | Create stash |
| `Git.stashApply()` | FilesActivity.optionStashApply() | Apply stash |
| `Git.stashDrop()` | FilesActivity.actionStashDelete() | Drop stash |
| `Git.stashList()` | FilesActivity.optionStashList() | List stashes |
| `Git.tag()` | FilesActivity.optionTagCreate() | Create tag |
| `Git.tagList()` | GitUtils.getTags() | List tags |
| `Git.status()` | FilesActivity.refreshStatus() | Working tree status |
| `BlameCommand` | BlameActivity | Blame file |
| `DiffFormatter` | FilesActivity, GitUtils, CommitActivity | Generate diffs |
| `RevWalk` | GitUtils, CommitActivity, LogActivity | Walk commits |
| `PlotWalk` | LogActivity | Graph with branch lanes |
| `RemoteConfig` | RemotesActivity, RemoteActivity, GitUtils | Remote management |
| `StoredConfig` | Many locations | Git config operations |
| `BranchTrackingStatus` | FilesActivity | Ahead/behind counts |
| `RepositoryBuilder` | FilesActivity.getRepository(), GitUtils.getRepository(), LogActivity | Open repository |
| `UsernamePasswordCredentialsProvider` | GitUtils.setCredentials() | HTTP auth |

### 5.2 SSH Authentication Flow

```
GitUtils.setCredentials(project, command)
  │
  ├── authentication == 1 (PASSWORD)
  │   ├── command.setCredentialsProvider(UsernamePasswordCredentialsProvider)
  │   └── setSSHPassword() → JschConfigSessionFactory.setPassword()
  │       └── StrictHostKeyChecking="no"
  │
  ├── authentication == 2 (PRIVATE KEY)
  │   └── setPrivateKey() → JSch.addIdentity(path, passphrase)
  │       └── StrictHostKeyChecking="no"
  │       ├── If passphrase available → addIdentity(path, passphrase)
  │       └── Then ALWAYS → addIdentity(path)   [BUG - double add]
  │
  └── authentication == 0 (NONE)
      └── setUnknownHost() → StrictHostKeyChecking="no"
```

---

## 6. AUTHENTICATION SYSTEM

### 6.1 Three Modes

| Mode | Code | UI Fields Shown | Backend |
|---|---|---|---|
| None | 0 | - | No credentials |
| Password | 1 | username, password | HTTP Basic / SSH password |
| Private Key | 2 | private key file path | SSH key (JSch) |

### 6.2 CredentialStorage Flow

```
checkCredentials(context, project, listener)
  │
  ├── Mode 0 (NONE)
  │   └── listener.credentialsSet()
  │
  ├── Mode 1 (PASSWORD)
  │   ├── password = getPassword(project)
  │   │   ├── project.getPassword() (from DB)
  │   │   └── mPasswords cache (in-memory HashMap)
  │   ├── If password found → listener.credentialsSet()
  │   └── If password missing → Dialog: enter password
  │       ├── Optional "Save Password" → updatePassword() to DB
  │       └── setPassword() → mPasswords cache + listener.credentialsSet()
  │
  └── Mode 2 (PRIVATE KEY)
      ├── Check privateKey file exists
      ├── KeyPair.load(JSch, path)
      ├── If encrypted → passphrase = getPassphrase(path) (in-memory)
      │   ├── If passphrase missing → Dialog: enter passphrase
      │   │   ├── keyPair.decrypt(passphrase)
      │   │   ├── If still encrypted → RECURSE (re-show dialog)
      │   │   └── setPassphrase() + listener.credentialsSet()
      │   ├── If passphrase found:
      │   │   ├── keyPair.decrypt(passphrase)
      │   │   ├── If still encrypted → removePassphrase() + RECURSE
      │   │   └── setPassphrase() + listener.credentialsSet()
      └── If not encrypted → listener.credentialsSet()
```

---

## 7. REPOSITORY MANAGEMENT FLOW

### 7.1 Remote Management

```
RemotesActivity
  ├── listView populated from RemoteConfig.getAllRemoteConfigs(config)
  ├── FAB → dialog → new RemoteConfig(config, name) → addURI + addFetchRefSpec → update
  └── CAB delete → config.unsetSection("remote", name) → save

RemoteActivity (per-remote detail)
  ├── Edit URL → removeURI() + addURI()
  ├── List RefSpecs (fetch + push)
  ├── FAB → dialog → addFetchRefSpec() or addPushRefSpec()
  └── CAB delete → removeFetchRefSpec() or removePushRefSpec()
```

### 7.2 Pull/Push/Fetch Flow (same pattern, 3x duplicated code)

```
optionPull() / optionPush() / optionFetch()
  ├── Get repository → getStoredConfig()
  ├── Get remotes → RemoteConfig.getAllRemoteConfigs()
  ├── If 0 remotes → toast "No remotes configured"
  ├── If 1 remote → execute directly
  └── If 2+ → Spinner dialog → execute selected
      └── executePull/Push/Fetch()
          └── CredentialStorage.checkCredentials()
              └── IntentService startService()
                  └── GitUtils.pull/push/fetchRepository()
                      └── setCredentials(project, transportCommand)
```

Bug: optionPull/Fetch/Push code blocks are **nearly identical** (20+ line duplicated blocks)

---

## 8. FILE BROWSER ARCHITECTURE

### 8.1 File Status Determination

```
FilesActivity.refreshStatus() [AsyncTask]
  └── Git.status().call() → Status object
      └── FilesActivity.refreshFiles(useCache)

FilesActivity.refreshFiles(useCache)
  └── getFiles(currentFolder) → List<GitFile>
      └── For each file in folder.listFiles():
          └── calculateFileStates(file) → Set<String>
              ├── contains(Status.getAdded(), file)       → "added"
              ├── contains(Status.getModified(), file)    → "modified"
              ├── contains(Status.getChanged(), file)     → "changed"
              ├── contains(Status.getRemoved(), file)     → "removed"
              ├── contains(Status.getConflicting(), file) → "conflicting"
              ├── containsParent(Status.getIgnoredNotInIndex(), file) → "ignored"
              ├── contains(Status.getUntracked(), file)   → "untracked"
              ├── if untracked AND NOT removed            → "untracked not removed"
              └── contains(Status.getUncommittedChanges(), file) → "uncommited"
      ├── Also add MISSING files from Status.getMissing()
      └── Sort via FileComparator.getPreferedComparator()
```

### 8.2 Status Icon Mapping (GitFileAdapter)

| State | Icon |
|-------|------|
| UNTRACKED | `ic_untracked` (question mark) |
| UNCOMMITED | `ic_uncommited` (orange dot) |
| IGNORED | `ic_ignored` (grey slash) |
| ADDED | `ic_added` (green plus) |
| CHANGED | `ic_changed` (blue arrow) |
| MODIFIED | `ic_modified` (orange pencil) |
| CHANGED+MODIFIED | `ic_changed_modified` (combined) |
| REMOVED | `ic_removed` (red minus) |
| CONFLICTING | `ic_conflicting` (red exclamation) |
| MISSING | `ic_missing` (red X) |

---

## 9. SECURITY VULNERABILITY MATRIX

| # | Severity | Finding | Location | Impact | Fix |
|---|---|---|---|---|---|
| S1 | **CRITICAL** | Plaintext passwords in SQLite | `PocketDbHelper.java:16` `ProjectsDataSource.java:52-53` | Any app with root or ADB backup can read all stored passwords | Use EncryptedSharedPreferences or SQLCipher |
| S2 | **CRITICAL** | SSH StrictHostKeyChecking disabled | `GitUtils.java:86,124,143` | MITM attacks on all SSH connections | Enable host key verification; store known_hosts |
| S3 | **CRITICAL** | SSL certificate verification disabled | `GitClone.java:41` | MITM attacks on HTTPS clone | Remove sslVerify=false override |
| S4 | **CRITICAL** | No Server Certificate Validation | `GitUtils.java:86` | `StrictHostKeyChecking=no` for all SSH auth types | Implement proper SSH host key validation |
| S5 | **HIGH** | Passwords passed in Intent extras | `MainActivity.java:91-93` | Any app reading logs/intents can intercept | Use secure channel or tokens |
| S6 | **HIGH** | `MANAGE_EXTERNAL_STORAGE` without runtime request | `AndroidManifest.xml:11` | Crashes on Android 11+ when accessing storage | Implement SAF; request permission properly |
| S7 | **HIGH** | MD5 used for Gravatar (deprecated) | `MD5Util.java` | MD5 is cryptographically broken | Use SHA-256 |
| S8 | **MEDIUM** | Hardcoded CP1252 charset | `MD5Util.java:17` | Inconsistent encoding across platforms | Use UTF-8 |
| S9 | **MEDIUM** | Gravatar over HTTP | `CommitActivity.java:75` | `http://www.gravatar.com` not HTTPS | Use `https://` |
| S10 | **MEDIUM** | No ProGuard/R8 obfuscation | `build.gradle:18` | APK easily decompilable | Enable minification |
| S11 | **MEDIUM** | JCenter deprecated | `settings.gradle:6` | Build will fail | Remove, use MavenCentral |
| S12 | **LOW** | `abortOnError false` | `build.gradle:25` | Lint issues hidden | Enable strict lint |
| S13 | **LOW** | Logging with `Log.e` may leak info | Throughout | Sensitive data in logcat | Strip logs in release |
| S14 | **LOW** | No ContentProvider URI permissions | `AndroidManifest.xml` | Direct file access without SAF | Use FileProvider |

---

## 10. DEAD & DUPLICATE CODE

### 10.1 Dead Code

| File | Code | Reason |
|---|---|---|
| `MergeDialog.java` | `BranchArrayAdapter` import | Imported but commented out (line 33, 71) |
| `GitUtils.java` | `prepareTreeParser()` parameter `objectId` | Parameter accepted but always resolves `"HEAD"` |
| `GitUtils.java` | `hasAtLeastOneReference()` | Never called anywhere |
| `GitUtils.java` | `containsException()` (private) | Written but never called; GitClone has its own copy |
| `FilesActivity.java:1472` | `// final ArrayAdapter<Ref> adapter = new BranchArrayAdapter(...)` | Commented out |
| `BranchArrayAdapter.java` | Entire class | Only used by the commented-out code in MergeDialog and the commented-out line in FilesActivity |
| `SettingsFragment.onCreatePreferences()` | Empty override | Required by PreferenceFragmentCompat but unused |
| `PocketDbHelper.onDowngrade()` | Empty method | No downgrade path needed |

### 10.2 Duplicate Code Blocks

| # | Code Pattern | Files | Lines Each |
|---|---|---|---|
| D1 | Pull/Push/Fetch remote selection dialog | `FilesActivity.java:1142-1381` | ~80 lines x 3 = 240 duplicated |
| D2 | `FontUtils.setRobotoFont(this, new MaterialDialog.Builder(this)...)` pattern | 14+ dialogs across all activities | ~10 lines each |
| D3 | `Repository repository = getRepository(project); ... repository.close();` | ~20 locations | 3-5 lines each |
| D4 | `ProjectsDataSource ds = new ProjectsDataSource(this); ds.open(); ... ds.close();` | ~15 locations | 4 lines each |
| D5 | ProgressMonitor anonymous inner class | `GitUtils.java:149-173, 229-253, 259-288, 291-315` | 12 lines x 4 |
| D6 | `containsException()` utility method | `GitClone.java:90-98`, `GitUtils.java:178-186` | 9 lines x 2 |
| D7 | `Credentials.checkCredentials()` + startService pattern | `FilesActivity.java:1186-1198, 1246-1258, 1306-1319, 1367-1380` | 12 lines x 4 |
| D8 | `Spinner remote selection` dialog building | `FilesActivity.java:1142-1183, 1202-1243, 1262-1303, 1323-1364` | ~40 lines x 4 |

---

## 11. PERFORMANCE BOTTLENECKS

| # | Issue | Location | Severity | Impact |
|---|---|---|---|---|
| P1 | `LogActivity.fillTo(Integer.MAX_VALUE)` | `LogActivity.java:77` | HIGH | Loads ALL commits into memory; OOM on large repos |
| P2 | `AsyncTask` on UI for git status | `FilesActivity.java:633-678` | HIGH | Blocks UI refresh; not cancellable properly |
| P3 | Repository opened/closed repeatedly | Every file operation | MEDIUM | `RepositoryBuilder` created fresh each time (no pool) |
| P4 | ListView instead of RecyclerView | All list screens | MEDIUM | Inefficient view recycling; no diffing |
| P5 | Files list recalculated from scratch | `FilesActivity.refreshFiles()` | MEDIUM | Full status re-read on every refresh |
| P6 | `calculateFileStates()` loops through all sets | `FilesActivity.java:714-775` | LOW | O(n*m) per file where n=status sets, m=files |
| P7 | `containsChild()` compares canonical paths | `FilesActivity.java:777-784` | LOW | String concatenation in loop; expensive I/O |
| P8 | No caching of Repository objects | Throughout | MEDIUM | Every operation re-opens .git directory |
| P9 | `Glide.with(this).load(gravatarURL)` in adapter | `CommitAdapter.java` | LOW | Network in getView() (though Glide async) |
| P10 | `FontUtils.setRobotoFont()` walks entire view tree | Every `onPostCreate()` | LOW | Recursive tree walk on every activity create |

---

## 12. DEPRECATED ANDROID APIS

| API | Status | Used In | Replacement |
|---|---|---|---|
| `AsyncTask` | Deprecated API 30 | `FilesActivity`, `LogActivity`, `BlameActivity`, `CheckoutLocalTask`, `CheckoutRemoteTask`, `SearchTask` | Coroutines + Flow |
| `IntentService` | Deprecated API 30 | `GitService`, `GitClone`, `GitPull`, `GitPush`, `GitFetch` | WorkManager or CoroutineWorker |
| `ListView` | Not deprecated but legacy | All activity layouts + adapters | RecyclerView |
| `ArrayAdapter` | Not deprecated but legacy | All 13 adapters | ListAdapter + DiffUtil |
| `ActionMode` (CAB) | Supported but legacy | `MainActivity`, `FilesActivity`, `RemotesActivity`, `RemoteActivity` | Contextual action with Material3 |
| `android.R.id.home` | Old pattern | All activities | Navigation Component |
| `startActivityForResult` | Deprecated API 30 | All activity results | Activity Result API |
| `onActivityResult` | Deprecated API 30 | `MainActivity`, `ProjectActivity`, `LogActivity`, `FilesActivity`, `PreferencesFragment` | Activity Result API |
| `onNewIntent(Bundle)` | Deprecated API 30 | `LogActivity` | Activity Result API (shadow) |
| SharedPreferences `.commit()` | Not deprecated but sync | `PreferencesFragment.java:80` | `.apply()` for async writes |
| `ActivityCompat.requestPermissions()` | Not deprecated but legacy | `MainActivity.java:298` | `registerForActivityResult` |

---

## 13. MODERNIZATION OPPORTUNITIES

### 13.1 Architecture

| Current | Target | Benefit |
|---------|--------|---------|
| Activity-based | Single Activity + Compose/Navigation | Better lifecycle, shared state |
| No DI | Hilt / Koin | Testability, decoupling |
| Manual DB access | Room + Repository pattern | Type safety, LiveData/Flow |
| Static utils | Injectible services | Testability |
| God class FilesActivity | ViewModel + UseCases + Repository | Maintainability |
| No tests | Unit tests + UI tests | Quality assurance |

### 13.2 UI

| Current | Target | Benefit |
|---------|--------|---------|
| ListView + ArrayAdapter | RecyclerView + ListAdapter | Performance, diffing |
| Material Dialogs (abandoned lib) | Material3 Dialogs | Modern look, maintained |
| Legacy themes | Material3 + Dynamic Color | Modern appearance |
| FloatingActionButton (custom) | Material FAB | Standard, accessible |
| XML layouts | Jetpack Compose (optional) | Declarative, less code |
| DrawerLayout | Navigation Rail + Bottom Nav | Modern navigation |
| Legacy icons | Material Icons | Consistent design |

### 13.3 Background Work

| Current | Target | Benefit |
|---------|--------|---------|
| IntentService | WorkManager | Survives process death, modern API |
| AsyncTask | Coroutines + Flow | Structured concurrency, lifecycle-aware |
| BroadcastReceiver for updates | StateFlow / SharedFlow | Better state management |

### 13.4 Data

| Current | Target | Benefit |
|---------|--------|---------|
| SQLiteOpenHelper | Room | Type safety, migration, Flow |
| Plaintext passwords | EncryptedSharedPreferences | Security |
| No caching | Repository + cache layer | Performance |
| Manual Cursor parsing | Room DAO | Safety, compile-time checks |

### 13.5 Security

| Current | Target | Benefit |
|---------|--------|---------|
| SSH StrictHostKeyChecking=no | Proper host key verification | MITM protection |
| sslVerify=false | Proper SSL validation | MITM protection |
| MD5 for Gravatar | SHA-256 or disable | Cryptographic safety |
| CP1252 charset | UTF-8 | Cross-platform consistency |

---

## 14. POCKET GIT X - TRANSFORMATION STRATEGY

### 14.1 Preserved Functionality (DO NOT TOUCH)

These Git operations work correctly and must be preserved identically:

- `GitUtils.cloneRepository()` - Clone with progress
- `GitUtils.pullRepository()` - Pull with progress  
- `GitUtils.pushRepository()` - Push with progress + result parsing
- `GitUtils.fetchRepository()` - Fetch with progress
- `GitUtils.stageFiles()` - git add
- `GitUtils.unstageFiles()` - git reset HEAD
- `GitUtils.removeFiles()` - git rm --cached
- `GitUtils.revertFiles()` - git checkout (file revert)
- `GitUtils.revert()` - git reset --hard HEAD
- `GitUtils.getDiffList()` - diff between commits
- `GitUtils.getDiffText()` - format diff entry
- `GitUtils.getTags()` - list tags
- `GitUtils.formatDate()` - date formatting
- `GitUtils.repositoryCloned()` - check if cloned
- `GitUtils.getRepository()` - open repository
- `GitUtils.setCredentials()` - authentication setup
- `GitUtils.prepareTreeParser()` - tree parsing for diff
- `CredentialStorage.checkCredentials()` - credential flow
- `CredentialStorage.getPassword()` - password retrieval
- `PocketDbHelper` - database schema (can be replaced with Room but must preserve data)
- `ProjectsDataSource` - CRUD operations (can migrate to Room)

### 14.2 Preserved Git Operation Flows

- GitClone.onHandleIntent() - Clone service logic
- GitPull.onHandleIntent() - Pull service logic
- GitPush.onHandleIntent() - Push service logic
- GitFetch.onHandleIntent() - Fetch service logic
- CheckoutLocalTask - Local branch checkout
- CheckoutRemoteTask - Remote branch checkout with tracking
- BlameActivity.update() - Blame computation
- LogActivity.initializeList() - Commit graph generation
- FilesActivity action methods (stage, unstage, diff, blame, etc.)
- FilesActivity.calculateFileStates()
- All dialog business logic (merge, sort, tag list, checkout remote)

### 14.3 Complete Replacements

**Replace 1: All UI**
- All 14 Activities → Single Activity + Fragments/Compose
- All 44 layout XMLs → New Material3 layouts
- All 8 menu XMLs → New menu structure
- All 13 adapters → RecyclerView adapters
- Custom FAB → Material FAB
- All legacy dialogs → Modern Material3 dialogs
- Help WebView → Help content

**Replace 2: Navigation**
- Intent-based nav → Navigation Component
- DrawerLayout → Bottom Navigation + Navigation Rail
- startActivityForResult → Activity Result API

**Replace 3: Background Execution**
- IntentService → WorkManager
- AsyncTask → Coroutines

**Replace 4: Database**
- SQLiteOpenHelper + DataSource → Room + Repository

**Replace 5: Architecture**
- Static utils → Dependency injection
- God classes → MVVM with ViewModels
- No tests → Unit + UI tests

### 14.4 Package Structure for Pocket Git X

```
com.aor.pocketgitx/
├── PocketGitXApp.kt              # Application class
├── MainActivity.kt                # Single Activity host
│
├── navigation/
│   ├── Destinations.kt            # Route definitions
│   ├── NavGraph.kt                # Navigation graph
│   └── BottomNavItem.kt           # Bottom nav items
│
├── ui/
│   ├── home/                      # Home Screen
│   │   ├── HomeScreen.kt
│   │   ├── HomeViewModel.kt
│   │   ├── RecentRepositoriesCard.kt
│   │   └── QuickActionsGrid.kt
│   ├── repositories/              # Repository Dashboard
│   │   ├── detail/                # Single repo view
│   │   │   ├── RepoScreen.kt
│   │   │   ├── RepoViewModel.kt
│   │   │   ├── StatsCards.kt
│   │   │   ├── QuickActions.kt
│   │   │   └── ToolSection.kt
│   │   ├── files/                 # File Explorer
│   │   │   ├── FileExplorerScreen.kt
│   │   │   ├── FileExplorerViewModel.kt
│   │   │   ├── FileListAdapter.kt
│   │   │   └── BreadcrumbNav.kt
│   │   ├── history/               # Commit History
│   │   │   ├── CommitHistoryScreen.kt
│   │   │   ├── CommitHistoryViewModel.kt
│   │   │   └── CommitAdapter.kt
│   │   ├── commit/                # Commit Detail
│   │   │   ├── CommitDetailScreen.kt
│   │   │   ├── CommitDetailViewModel.kt
│   │   │   └── DiffViewer.kt
│   │   └── branches/              # Branches & Tags
│   │       ├── BranchListScreen.kt
│   │       ├── TagListScreen.kt
│   │       └── BranchViewModel.kt
│   ├── ai/                        # AI Assistant
│   │   ├── AIScreen.kt
│   │   ├── AIViewModel.kt
│   │   ├── ChatBubble.kt
│   │   └── ChatAdapter.kt
│   ├── activity/                  # Activity Feed
│   │   ├── ActivityScreen.kt
│   │   └── ActivityViewModel.kt
│   ├── profile/                   # Profile / Settings
│   │   ├── ProfileScreen.kt
│   │   └── SettingsScreen.kt
│   └── components/                # Shared Components
│       ├── GitStatusBadge.kt
│       ├── DiffLineView.kt
│       ├── BranchLaneView.kt
│       ├── GravatarImage.kt
│       ├── ProgressOverlay.kt
│       ├── ErrorDialog.kt
│       └── ConfirmDialog.kt
│
├── data/
│   ├── local/
│   │   ├── AppDatabase.kt          # Room database
│   │   ├── ProjectEntity.kt        # Room entity
│   │   ├── ProjectDao.kt           # Room DAO
│   │   └── Converters.kt           # Type converters
│   ├── repository/
│   │   ├── ProjectRepository.kt
│   │   ├── GitRepository.kt
│   │   └── UserPreferences.kt
│   └── model/
│       ├── GitFileState.kt
│       ├── CommitInfo.kt
│       ├── BranchInfo.kt
│       └── RemoteInfo.kt
│
├── domain/
│   ├── git/
│   │   ├── GitOperations.kt        # Wraps GitUtils
│   │   ├── CloneManager.kt
│   │   ├── RemoteManager.kt
│   │   ├── StashManager.kt
│   │   └── TagManager.kt
│   ├── security/
│   │   ├── CredentialManager.kt
│   │   ├── KeyStoreProvider.kt
│   │   └── SecureStorage.kt
│   └── ai/
│       ├── AIService.kt
│       ├── AIPromptBuilder.kt
│       └── GitAnalyzer.kt
│
└── worker/
    ├── CloneWorker.kt
    ├── PullWorker.kt
    ├── PushWorker.kt
    └── FetchWorker.kt
```

---

## 15. IMPLEMENTATION PHASES

### Phase 0: Foundation (Week 1-2)

**Goal**: Set up modern build system + dependency injection + database

Changes:
1. Upgrade Gradle → 8.7, AGP → 8.5
2. Remove JCenter → MavenCentral only
3. Add Kotlin support (start with Kotlin 2.0)
4. Add Hilt for DI
5. Add Room database (migrate from PocketDbHelper)
6. Add EncryptedSharedPreferences (migrate plaintext passwords)
7. Add Coroutines + Flow dependencies
8. Set up testing framework (JUnit 5, Mockk, Turbine)
9. Configure R8/ProGuard properly
10. Add Material3 Compose (optional) or Material3 views

**Risk**: Build may break; need careful Gradle upgrade path

### Phase 1: Security Hardening (Week 2-3)

**Goal**: Fix all critical security vulnerabilities

Changes:
1. Migrate passwords from SQLite to EncryptedSharedPreferences
2. Implement SSH host key verification
3. Enable SSL certificate validation
4. Replace MD5 with SHA-256 for Gravatar
5. Switch Gravatar URL to HTTPS
6. Implement SAF for file access (remove MANAGE_EXTERNAL_STORAGE)
7. Add runtime permission handling for MANAGE_EXTERNAL_STORAGE

**Risk**: SSH/SSL changes may break existing connections; test with multiple providers

### Phase 2: Architecture Refactor (Week 3-6)

**Goal**: Replace legacy architecture without breaking Git ops

Changes:
1. Extract FilesActivity into ViewModel + UseCases
2. Replace IntentService with WorkManager workers
3. Replace AsyncTask with Coroutines
4. Replace all ListViews with RecyclerViews
5. Replace ArrayAdapters with ListAdapters + DiffUtil
6. Replace startActivityForResult with Activity Result API
7. Add Repository pattern for data access
8. Extract duplicate Pull/Push/Fetch dialog code

**Risk**: Large refactor; must preserve all Git operation logic

### Phase 3: UI Redesign - Pocket Git X (Week 6-10)

**Goal**: Complete UI transformation to Material3

Changes:
1. Create single Activity with Navigation Component
2. Implement Bottom Navigation (Home, Repositories, AI, Activity, Profile)
3. Build Home Screen (Recent repos, Quick Actions, Activity feed)
4. Build Repository Dashboard (stats, actions, tools)
5. Build File Explorer (breadcrumb, git badges, search, sort)
6. Build Commit History (timeline UI, filters, search)
7. Build Commit Details (metadata, files changed, diff viewer)
8. Build Branch/Tag management screens
9. Build Remote management screens
10. Build AI Assistant screen
11. Update all themes to Material3 with Dynamic Color

**Risk**: Large UI surface; test all Git operations after redesign

### Phase 4: AI Integration (Week 10-12)

**Goal**: Add AI-powered developer assistant

Changes:
1. Build AI Service abstraction (supports multiple providers)
2. Implement project analysis prompts
3. Implement file explanation prompts
4. Implement commit message generation
5. Implement bug detection prompts
6. Implement code review prompts
7. Add chat UI with conversation history
8. Add settings for AI provider configuration

**Risk**: API-dependent feature; need graceful degradation when offline

### Phase 5: Polish & Release (Week 12-14)

**Goal**: Production-ready release

Changes:
1. Performance optimization (caching, lazy loading, pagination)
2. Accessibility improvements
3. Dark mode / Dynamic color refinement
4. Edge-to-edge display
5. Add unit tests (critical paths)
6. Add UI tests (critical screens)
7. Set up CI/CD pipeline
8. Crash reporting (Firebase or Sentry)
9. Analytics (optional)
10. Play Store listing preparation

**Risk**: Testing may reveal regressions from earlier phases

---

## APPENDIX: FILE-LEVEL SUMMARY

| Package | File | Lines | Status | Key Responsibility |
|---|---|---|---|---|
| **activities** | MainActivity.java | 385 | **REFACTOR** | Project list, clone management |
| | ProjectActivity.java | 334 | **REFACTOR** | Add/edit project |
| | FilesActivity.java | 1889 | **REWRITE** | File browser + all Git ops (GOD CLASS) |
| | UpdatableActivity.java | 74 | **REWRITE** | Base with broadcast receiver |
| | CommitActivity.java | 97 | **REWRITE** | Commit details |
| | DiffActivity.java | 48 | **REWRITE** | Diff viewer |
| | BlameActivity.java | 113 | **REWRITE** | Blame view |
| | LogActivity.java | 103 | **REWRITE** | Commit graph |
| | RemotesActivity.java | 165 | **REWRITE** | Remote management |
| | RemoteActivity.java | 203 | **REWRITE** | Remote detail + refspecs |
| | PickerActivity.java | 165 | **REWRITE** | File/folder chooser |
| | PreferencesActivity.java | 33 | **REWRITE** | Settings |
| | PreferencesFragment.java | 93 | **REWRITE** | Settings fragment |
| | HelpActivity.java | 25 | **REWRITE** | Help |
| **data** | Project.java | 73 | **KEEP** | Data model |
| | GitFile.java | 100 | **KEEP** | File with states |
| | DiffLine.java | 27 | **KEEP** | Diff line model |
| | BlameLine.java | 33 | **KEEP** | Blame line model |
| | TypedRefSpec.java | 26 | **KEEP** | RefSpec wrapper |
| **database** | PocketDbHelper.java | 29 | **REPLACE** | SQLite schema |
| | ProjectsDataSource.java | 108 | **REPLACE** | CRUD operations |
| **services** | GitService.java | 138 | **REPLACE** | Base IntentService |
| | GitClone.java | 110 | **REWRITE** | Clone logic (KEEP core, REWRITE service wrapper) |
| | GitPull.java | 43 | **REWRITE** | Pull logic |
| | GitPush.java | 50 | **REWRITE** | Push logic |
| | GitFetch.java | 43 | **REWRITE** | Fetch logic |
| **tasks** | CheckoutLocalTask.java | ~40 | **REPLACE** | AsyncTask → Coroutine |
| | CheckoutRemoteTask.java | ~50 | **REPLACE** | AsyncTask → Coroutine |
| **dialogs** | CheckoutRemoteDialog.java | ~80 | **REWRITE** | Dialog logic → Material3 |
| | MergeDialog.java | ~70 | **REWRITE** | Dialog logic → Material3 |
| | SortFilesDialog.java | ~70 | **REWRITE** | Dialog logic → Material3 |
| | TagListDialog.java | ~70 | **REWRITE** | Dialog logic → Material3 |
| **utils** | GitUtils.java | 442 | **ADD MIGRATION LAYER** | Core Git ops (KEEP as-is, wrap) |
| | CredentialStorage.java | 162 | **REWRITE** | Use Keystore |
| | FileUtils.java | 31 | **KEEP** | Search utility |
| | FileComparator.java | 101 | **KEEP** | Sort utility |
| | FontUtils.java | 71 | **REWRITE** | Use system fonts |
| | MD5Util.java | 23 | **REPLACE** | SHA-256 |
| **widgets** | FloatingActionButton.java | ~120 | **REPLACE** | Material FAB |
| | FlowLayout.java | ~120 | **KEEP** | Custom layout (still useful) |
| | PlotLaneView.java | ~100 | **KEEP** | Commit graph rendering |
| **application** | PocketGit.java | 27 | **KEEP** | Notification channel setup |
| **adapters** | 13 adapters | ~30-120 each | **REPLACE** | RecyclerView adapters |
