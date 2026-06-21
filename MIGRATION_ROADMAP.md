# Pocket Git X - Migration Roadmap

> **Status**: Pre-Implementation Planning  
> **Target**: Pocket Git X v1.0  
> **Strategy**: Incremental, independently buildable phases  

---

## TABLE OF CONTENTS

1. [Core Principles](#1-core-principles)
2. [Current Architecture Diagram](#2-current-architecture-diagram)
3. [Target Architecture Diagram](#3-target-architecture-diagram)
4. [UI Screen Map](#4-ui-screen-map)
5. [Navigation Flow](#5-navigation-flow)
6. [Database Migration Plan](#6-database-migration-plan)
7. [Security Migration Plan](#7-security-migration-plan)
8. [RecyclerView Migration Plan](#8-recyclerview-migration-plan)
9. [Material 3 Migration Plan](#9-material-3-migration-plan)
10. [AI Assistant Integration Plan](#10-ai-assistant-integration-plan)
11. [Risk Assessment Matrix](#11-risk-assessment-matrix)
12. [File Modification Strategy](#12-file-modification-strategy)
13. [Phase Recommendations](#13-phase-recommendations)
14. [Approval Request](#14-approval-request)

---

## 1. CORE PRINCIPLES

### Sacred Files (DO NOT MODIFY INITIALLY)

These files contain working Git logic that must be preserved as-is:

| File | Reason | When to Touch |
|---|---|---|
| `GitUtils.java` | All core JGit operations; every Git feature depends on it | Phase 4+ (add wrapper layer only) |
| `CredentialStorage.java` | Credential flow logic used by all remote operations | Phase 2 (migrate storage backend only) |
| `GitClone.java` | Clone with certificate fallback retry | Phase 3 (replace IntentService base only) |
| `GitPull.java` | Pull with progress reporting | Phase 3 (replace IntentService base only) |
| `GitPush.java` | Push with result parsing | Phase 3 (replace IntentService base only) |
| `GitFetch.java` | Fetch with progress reporting | Phase 3 (replace IntentService base only) |
| `PocketDbHelper.java` | Database schema must be migrated, not dropped | Phase 2 only |
| `ProjectsDataSource.java` | All project CRUD must be preserved | Phase 2 only |
| `Project.java`, `GitFile.java`, `DiffLine.java`, `BlameLine.java`, `TypedRefSpec.java` | Data models used everywhere | Phase 3+ (can be converted to Kotlin data classes) |

### Replace-First Files (HIGHEST IMPACT, LOWEST RISK)

These files have no business logic and can be replaced immediately:

| File | Replacement | Effort | Impact |
|---|---|---|---|
| `DiffActivity.java` | New Material3 Diff Screen | Low | High (visible UI) |
| `BlameActivity.java` | New Material3 Blame Screen | Low | High (visible UI) |
| `HelpActivity.java` | New Help Screen | Very Low | Low |
| `FloatingActionButton.java` | Material FAB | Low | Medium |
| `MD5Util.java` | SHA-256 / remove | Very Low | Low |

### God Class Strategy for FilesActivity (1889 lines)

**DO NOT** attempt to rewrite in one phase. Extract in order:

1. Extract `SearchTask` → standalone file
2. Extract `actionCheckoutRemote/actionCheckoutLocal` → BranchManager
3. Extract `optionPull/optionFetch/optionPush` + variants → RemoteActionManager
4. Extract stash operations → StashManager
5. Extract tag operations → TagManager
6. Extract file operations → FileActionManager
7. Extract commit flow → CommitManager
8. Extract `calculateFileStates`, `contains`, `containsChild`, `containsParent` → FileStateCalculator
9. Remaining UI shell → FilesViewModel + FilesFragment

---

## 2. CURRENT ARCHITECTURE DIAGRAM

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PRESENTATION LAYER                              │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │                   ACTIVITIES (14 files)                      │       │
│  │                                                               │       │
│  │  MainActivity   FilesActivity(1889)  ProjectActivity         │       │
│  │  LogActivity    CommitActivity      DiffActivity             │       │
│  │  BlameActivity  HelpActivity        PickerActivity           │       │
│  │  RemotesActivity RemoteActivity                              │       │
│  │  PreferencesActivity  PreferencesFragment                    │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │ extends                                     │
│  ┌──────────────────────▼──────────────────────────────────────┐       │
│  │                  UpdatableActivity                            │       │
│  │  ┌─ BroadcastReceiver (UPDATE_BROADCAST)                     │       │
│  │  └─ Snackbar helpers, FontUtils.apply()                      │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │ extends                                     │
│  ┌──────────────────────▼──────────────────────────────────────┐       │
│  │              AppCompatActivity (AndroidX)                     │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  DIALOGS (4)     │  TASKS (2)      │  ADAPTERS (13)          │       │
│  │  CheckoutRemote  │  CheckoutLocal  │  ProjectAdapter         │       │
│  │  MergeDialog     │  CheckoutRemote │  GitFileAdapter         │       │
│  │  SortFilesDialog │                 │  CommitAdapter          │       │
│  │  TagListDialog   │                 │  BranchAdapter          │       │
│  │                  │                 │  BlameLineAdapter       │       │
│  │  WIDGETS (3)     │                 │  DiffAdapter            │       │
│  │  FloatingAction  │                 │  DiffLineAdapter        │       │
│  │  FlowLayout      │                 │  FolderAdapter          │       │
│  │  PlotLaneView    │                 │  RemoteAdapter          │       │
│  │                  │                 │  RefSpecAdapter         │       │
│  │                  │                 │  StashAdapter           │       │
│  │                  │                 │  TagAdapter             │       │
│  │                  │                 │  BranchArrayAdapter     │       │
│  └──────────────────┴─────────────────┴────────────────────────┘       │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │ depends on
┌──────────────────────────▼──────────────────────────────────────────────┐
│                       BUSINESS LOGIC LAYER                              │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  GitUtils.java (442 lines - STATIC METHODS)                  │       │
│  │  ├── cloneRepository()   ├── pullRepository()                │       │
│  │  ├── pushRepository()    ├── fetchRepository()               │       │
│  │  ├── stageFiles()        ├── unstageFiles()                  │       │
│  │  ├── removeFiles()       ├── revertFiles()                   │       │
│  │  ├── revert()            ├── setCredentials()                │       │
│  │  ├── getRepository()     ├── getDiffList()                   │       │
│  │  ├── getDiffText()       ├── prepareTreeParser()             │       │
│  │  ├── formatDate()        ├── getTags()                       │       │
│  │  └── repositoryCloned()                                     │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │ uses                                        │
│  ┌──────────────────────▼──────────────────────────────────────┐       │
│  │  CredentialStorage.java (static)                            │       │
│  │  ├── checkCredentials() → dialog + JSch key handling         │       │
│  │  ├── getPassword() / setPassword() → in-memory cache        │       │
│  │  └── getPassphrase() / setPassphrase() → in-memory cache     │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  SERVICES (IntentService background)                        │       │
│  │                                                              │       │
│  │  GitService (abstract)          GitClone                     │       │
│  │  ├── startNotification()        GitPull                      │       │
│  │  ├── updateNotification()       GitPush                      │       │
│  │  ├── finishNotification*()      GitFetch                     │       │
│  │  └── broadcastMessage()                                     │       │
│  └─────────────────────────────────────────────────────────────┘       │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │ persists
┌──────────────────────────▼──────────────────────────────────────────────┐
│                        DATA LAYER                                      │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  PocketDbHelper (SQLiteOpenHelper v2)                       │       │
│  │  ProjectsDataSource (manual CRUD)                           │       │
│  │  └── projects TABLE: id, name, url, local_path,             │       │
│  │       authentication, username, password, privatekey, state │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  DATA MODELS (POJO)                                         │       │
│  │  Project  GitFile  DiffLine  BlameLine  TypedRefSpec         │       │
│  └─────────────────────────────────────────────────────────────┘       │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────────────┐
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  ANDROID OS LAYER                                           │       │
│  │  SharedPreferences  File System  Notifications  Net/SSL     │       │
│  │  JGit 6.2 + JSch   Glide 4.13  Material Dialogs 0.9.6      │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. TARGET ARCHITECTURE DIAGRAM

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      PRESENTATION LAYER                                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │              SINGLE ACTIVITY + NAVIGATION COMPONENT          │       │
│  │                                                               │       │
│  │  MainActivity (host)                                          │       │
│  │  ├── NavHostFragment                                          │       │
│  │  └── BottomNavigationBar                                      │       │
│  └─────────────────────────┬────────────────────────────────────┘       │
│                            │ contains                                  │
│  ┌─────────────────────────▼────────────────────────────────────┐       │
│  │                   FRAGMENTS / SCREENS                         │       │
│  │                                                               │       │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │       │
│  │  │  HOME    │ │  REPOS   │ │    AI    │ │ ACTIVITY │        │       │
│  │  │ Screen   │ │ Screen   │ │ Screen   │ │ Screen   │        │       │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘        │       │
│  │       │             │            │             │              │       │
│  │       │    ┌────────▼──────┐     │             │              │       │
│  │       │    │ Repo Detail   │     │             │              │       │
│  │       │    │ ├─ Dashboard  │     │             │              │       │
│  │       │    │ ├─ FileExplor │     │             │              │       │
│  │       │    │ ├─ CommitHist │     │             │              │       │
│  │       │    │ ├─ Branches   │     │             │              │       │
│  │       │    │ ├─ Remotes    │     │             │              │       │
│  │       │    │ └─ Stashes    │     │             │              │       │
│  │       │    └───────────────┘     │             │              │       │
│  │       │                         │             │              │       │
│  │  ┌────▼─────┐              ┌────▼─────┐  ┌────▼─────┐       │       │
│  │  │  Add/Edit│              │ AI Chat  │  │ Settings │       │       │
│  │  │  Project │              └──────────┘  └──────────┘       │       │
│  │  └──────────┘                                              │       │
│  └─────────────────────────┬────────────────────────────────────┘       │
│                            │ uses                                      │
│  ┌─────────────────────────▼────────────────────────────────────┐       │
│  │                   SHARED COMPONENTS                          │       │
│  │  GitStatusBadge  DiffLineView  BranchLaneView                │       │
│  │  GravatarImage   ProgressOverlay  ConfirmDialog              │       │
│  │  SearchBar       BreadcrumbNav  StatsCard                    │       │
│  └─────────────────────────────────────────────────────────────┘       │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │ observes
┌──────────────────────────▼──────────────────────────────────────────────┐
│                       VIEWMODEL LAYER                                   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  HomeViewModel    │  RepoViewModel    │  AIService            │       │
│  │  FileExplorerVM   │  CommitHistoryVM  │  BranchViewModel      │       │
│  │  RemoteViewModel  │  StashViewModel   │  TagViewModel         │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │ uses                                         │
└──────────────────────────┼──────────────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────────────┐
│                       DOMAIN/USE CASE LAYER                             │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  GitOperations.kt (WRAPS GitUtils.java)                    │       │
│  │  ├── clone()    ├── pull()    ├── push()    ├── fetch()     │       │
│  │  ├── stage()    ├── unstage() ├── remove()  ├── revert()   │       │
│  │  ├── commit()   ├── diff()    ├── blame()   ├── log()      │       │
│  │  ├── branch*()  ├── tag*()    ├── stash*()  ├── merge()    │       │
│  │  └── status()   └── config()                                 │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │ delegates                                    │
│  ┌──────────────────────▼──────────────────────────────────────┐       │
│  │  GitUtils.java (PRESERVED - UNMODIFIED)                     │       │
│  │  (All existing static methods remain intact)                │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  CredentialManager.kt                                       │       │
│  │  ├── GitHub OAuth / Personal Access Token support           │       │
│  │  ├── Android Keystore integration                          │       │
│  │  └── EncryptedSharedPreferences backend                    │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  WorkerManager (Replaces IntentService)                    │       │
│  │  CloneWorker  PullWorker  PushWorker  FetchWorker           │       │
│  └─────────────────────────────────────────────────────────────┘       │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────────────┐
│                       DATA LAYER                                        │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  ROOM DATABASE                                                │       │
│  │  ├── ProjectEntity (id, name, url, localPath, auth, state)   │       │
│  │  ├── AppDatabase (version 1)                                 │       │
│  │  ├── ProjectDao (@Insert, @Update, @Delete, @Query)          │       │
│  │  └── Migration SQLite→Room (preserves existing data)         │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │                                              │
│  ┌──────────────────────▼──────────────────────────────────────┐       │
│  │  DATASOURCE LAYER                                          │       │
│  │  ┌──────────────────────────────────────────────────────┐  │       │
│  │  │  EncryptedSharedPreferences (replaces plaintext DB)  │  │       │
│  │  │  ├── Encrypted passwords                             │  │       │
│  │  │  └── Encrypted SSH private key paths                 │  │       │
│  │  └──────────────────────────────────────────────────────┘  │       │
│  │                                                             │       │
│  │  ┌──────────────────────────────────────────────────────┐  │       │
│  │  │  WorkManager (background execution)                  │  │       │
│  │  │  └── Git operations survive app kill                 │  │       │
│  │  └──────────────────────────────────────────────────────┘  │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  DATA MODELS (Kotlin data classes)                         │       │
│  │  Project  GitFileState  CommitInfo  BranchInfo  RemoteInfo │       │
│  │  DiffLine  BlameLine  TypedRefSpec                        │       │
│  └─────────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. UI Screen Map

### Current Screens (14 Activities)

| Screen | Trigger | Purpose | Status |
|---|---|---|---|
| `MainActivity` | App launch | Project list + clone | **REPLACE** with HomeScreen |
| `ProjectActivity` | Add/Edit project | Form (name, URL, path, auth) | **REPLACE** with AddRepoSheet |
| `FilesActivity` | Project click | File browser + all git ops | **EXTRACT** → 6+ screens |
| `LogActivity` | Menu→Log | Commit graph | **REPLACE** with CommitHistoryScreen |
| `CommitActivity` | Log→commit | Commit detail + diff | **REPLACE** with CommitDetailScreen |
| `DiffActivity` | Commit/file diff | Unified diff view | **REPLACE** with DiffViewer |
| `BlameActivity` | CAB→Blame | Line-by-line blame | **REPLACE** with BlameViewer |
| `RemotesActivity` | Menu→Remotes | Remote list | **REPLACE** with RemoteListScreen |
| `RemoteActivity` | Remote→tap | Remote detail + refspecs | **REPLACE** with RemoteDetailScreen |
| `PickerActivity` | Path selection | File/folder browser | **REPLACE** with SAF document picker |
| `HelpActivity` | Menu→Help | WebView help | **REPLACE** with HelpScreen |
| `PreferencesActivity` | Menu→Settings | App preferences | **REPLACE** with SettingsScreen |
| `PreferencesFragment` | (in Preferences) | Settings UI | **REPLACE** with SettingsScreen |
| `UpdatableActivity` | (base class) | Broadcast + snackbar | **REMOVE** (replaced by ViewModel+Snackbar) |

### Target Screens (Fragments)

| Screen | Route | Purpose |
|---|---|---|
| **HomeScreen** | `/home` | Dashboard: recent repos, quick actions, activity |
| **RepoDashboardScreen** | `/repo/{id}` | Stats, branch info, actions, tool links |
| **FileExplorerScreen** | `/repo/{id}/files/{path?}` | File tree with status badges, breadcrumb |
| **CommitHistoryScreen** | `/repo/{id}/commits` | Commit timeline with graph, filter, search |
| **CommitDetailScreen** | `/repo/{id}/commit/{hash}` | Metadata, diff list, file changes |
| **DiffViewerScreen** | `/repo/{id}/diff` | Unified diff with syntax highlighting |
| **BlameViewerScreen** | `/repo/{id}/blame/{file}` | Line-by-line with author/sha |
| **BranchListScreen** | `/repo/{id}/branches` | Local + remote branch management |
| **TagListScreen** | `/repo/{id}/tags` | Tag management |
| **RemoteListScreen** | `/repo/{id}/remotes` | Remote management |
| **RemoteDetailScreen** | `/repo/{id}/remote/{name}` | Remote URL + refspecs |
| **StashListScreen** | `/repo/{id}/stashes` | Stash management |
| **MergeScreen** | `/repo/{id}/merge` | Branch merge |
| **AddRepoScreen** | `/repo/add` | Clone/create repository |
| **AIChatScreen** | `/ai` | AI assistant chat |
| **ActivityScreen** | `/activity` | Recent git activity |
| **SettingsScreen** | `/settings` | App configuration |
| **HelpScreen** | `/help` | Help & documentation |

---

## 5. NAVIGATION FLOW

### Current (Intent-based)

```
MainActivity
  ├── startActivityForResult(ProjectActivity, 1)    → Add project
  ├── startActivityForResult(ProjectActivity, 2)    → Edit project
  ├── startService(GitClone)                        → Clone (background)
  └── startActivity(FilesActivity)                  → Open project

FilesActivity
  ├── startActivity(LogActivity)                    → Commit history
  ├── startActivity(CommitActivity)                 → Commit detail
  ├── startActivity(DiffActivity)                   → Diff view
  ├── startActivity(BlameActivity)                  → Blame view
  ├── startActivity(RemotesActivity)                → Remotes
  ├── startActivity(HelpActivity)                   → Help
  ├── startActivity(LogActivity+result)             → Checkout from log
  ├── startService(GitPull/GitPush/GitFetch)        → Remote ops
  ├── startActivity(Intent.ACTION_VIEW)              → Open file externally
  └── AlertDialogs (MaterialDialog)                 → Stash, tag, branch, etc.

RemotesActivity
  └── startActivity(RemoteActivity)                 → Edit remote

ProjectActivity
  ├── startActivityForResult(PickerActivity, 1)     → Select folder
  └── startActivityForResult(PickerActivity, 2)     → Select private key
```

### Target (Navigation Component)

```
NavGraph
│
├── bottomNavItems[]
│   ├── "Home"         → /home                       (start)
│   ├── "Repositories" → /repos                      (list)
│   ├── "AI"           → /ai                         (chat)
│   ├── "Activity"     → /activity                   (feed)
│   └── "Profile"      → /settings                   (settings)
│
├── nested Graphs
│   │
│   ├── homeGraph
│   │   ├── /home                                   HomeScreen
│   │   ├── /repo/add                               AddRepoScreen
│   │   └── /repo/{id}                              → repoGraph
│   │
│   ├── repoGraph (nested)
│   │   ├── /repo/{id}                              RepoDashboardScreen
│   │   ├── /repo/{id}/files/{path?}                FileExplorerScreen
│   │   │   └── /repo/{id}/diff/{file}             DiffViewerScreen
│   │   │   └── /repo/{id}/blame/{file}            BlameViewerScreen
│   │   ├── /repo/{id}/commits                      CommitHistoryScreen
│   │   │   └── /repo/{id}/commit/{hash}           CommitDetailScreen
│   │   │       └── /repo/{id}/diff/{hash}/{file}   DiffViewerScreen
│   │   ├── /repo/{id}/branches                     BranchListScreen
│   │   │   └── /repo/{id}/merge/{branch}          MergeScreen
│   │   ├── /repo/{id}/tags                         TagListScreen
│   │   ├── /repo/{id}/remotes                      RemoteListScreen
│   │   │   └── /repo/{id}/remote/{name}           RemoteDetailScreen
│   │   └── /repo/{id}/stashes                      StashListScreen
│   │
│   ├── activityGraph
│   │   └── /activity                               ActivityScreen
│   │
│   └── settingsGraph
│       ├── /settings                               SettingsScreen
│       └── /help                                   HelpScreen
```

---

## 6. DATABASE MIGRATION PLAN

### Strategy: Room Migration with Data Preservation

**Why Room**: Type-safe DAOs, compile-time SQL verification, Flow integration, automatic migration testing.

**Migration Path**:

```
Current: SQLiteOpenHelper v2 (PocketGit.db)
    │
    ├── Phase 2a: Add Room without removing SQLite
    │   ├── Create AppDatabase with ProjectEntity
    │   ├── Write Migration(2→3) from current schema to Room
    │   ├── Keep ProjectsDataSource working alongside Room
    │   └── Verify data preserved
    │
    ├── Phase 2b: Replace ProjectsDataSource calls with ProjectDao
    │   ├── Create ProjectRepository wrapping ProjectDao
    │   ├── Migrate callers one by one
    │   └── Delete ProjectsDataSource when no callers remain
    │
    └── Remove PocketDbHelper when all migration complete
```

### Room Entity Design

```kotlin
@Entity(tableName = "projects")
data class ProjectEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val name: String,
    val url: String,
    val localPath: String,
    val authentication: Int = 0,        // 0=NONE, 1=PASSWORD, 2=PRIVATE_KEY
    val username: String = "",
    // ⚠ Password and privateKey migrated to EncryptedSharedPreferences
    val state: String = "UNKNOWN"
)
```

### Migration SQL (v2 → v3)

```sql
-- Current schema (v2):
CREATE TABLE projects (
    id INTEGER PRIMARY KEY,
    name VARCHAR,
    url VARCHAR,
    local_path VARCHAR,
    authentication INTEGER,
    username VARCHAR,
    password VARCHAR,
    privatekey VARCHAR,
    state VARCHAR
);

-- Migration (v2 → Room v3):
-- Move password and privatekey out of DB
-- All other columns remain identical
-- Write migration script:
ALTER TABLE projects RENAME TO projects_old;
CREATE TABLE projects (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    url TEXT NOT NULL,
    local_path TEXT NOT NULL,
    authentication INTEGER NOT NULL DEFAULT 0,
    username TEXT NOT NULL DEFAULT '',
    state TEXT NOT NULL DEFAULT 'UNKNOWN'
);
INSERT INTO projects (id, name, url, local_path, authentication, username, state)
    SELECT id, name, url, local_path, authentication, username, state
    FROM projects_old;
DROP TABLE projects_old;
```

### Data Export Script (for password migration)

Before dropping password columns, export existing passwords:

```sql
SELECT id, password, privatekey FROM projects WHERE password != '' OR privatekey != '';
```

→ Write to EncryptedSharedPreferences
→ Verify before completing migration

---

## 7. SECURITY MIGRATION PLAN

### Phase-by-Phase Security Improvements

| Phase | Fix | Method | Risk Level |
|---|---|---|---|
| **Phase 1** | MD5 → SHA-256 for Gravatar | Replace `MD5Util.md5Hex()` with `MessageDigest.getInstance("SHA-256")` | None |
| | Gravatar HTTP → HTTPS | Change `http://` to `https://` in CommitActivity | None |
| | CP1252 → UTF-8 | Change charset in MD5Util | None |
| **Phase 2a** | Plaintext passwords → EncryptedSharedPreferences | Migrate on Room migration; remove password column from DB | Medium |
| | Private key paths → EncryptedSharedPreferences | Same as passwords | Medium |
| **Phase 2b** | SSH StrictHostKeyChecking → proper verification | Add known_hosts support in GitUtils | **HIGH** (may break existing SSH) |
| | SSL verification re-enabled | Remove `sslVerify=false` from GitClone | **HIGH** (may break self-signed certs) |
| **Phase 3** | Runtime permission for MANAGE_EXTERNAL_STORAGE | Add proper permission flow before file access | Low |
| | Intent extra passwords → secure channel | Remove sensitive data from Intent extras | Medium |

### EncryptedSharedPreferences Implementation

```kotlin
// New class: SecurePreferences.kt
class SecurePreferences(context: Context) {
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    private val prefs = EncryptedSharedPreferences.create(
        context,
        "pocketgit_secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun savePassword(projectId: Int, password: String) {
        prefs.edit().putString("password_$projectId", password).apply()
    }

    fun getPassword(projectId: Int): String? {
        return prefs.getString("password_$projectId", null)
    }

    fun savePrivateKeyPath(projectId: Int, path: String) {
        prefs.edit().putString("privatekey_$projectId", path).apply()
    }

    fun getPrivateKeyPath(projectId: Int): String? {
        return prefs.getString("privatekey_$projectId", null)
    }
}
```

### SSH Host Key Verification Plan

This is the most sensitive change. Current code disables verification entirely. Fix:

1. Add a `known_hosts` file in app's internal storage
2. In `setPrivateKey()`, `setSSHPassword()`, `setUnknownHost()`:
   - Load known_hosts
   - Set `StrictHostKeyChecking=yes`
   - On first connection to new host, prompt user to accept fingerprint
   - Store accepted fingerprints in known_hosts

```kotlin
// Future approach for SSH host key verification
session.setConfig("StrictHostKeyChecking", "ask")  // or "yes" with known_hosts
// Add HostKeyRepository that stores to internal file
```

**Risk Mitigation**: Make this a preference option. Default `"no"` initially, with a warning. After verification, change default to `"ask"`.

---

## 8. RECYCLERVIEW MIGRATION PLAN

### Mapping Table: ListView → RecyclerView

| Current Screen | Adapter | ListView ID | Target RecyclerView |
|---|---|---|---|
| MainActivity | ProjectAdapter | `list_projects` | HomeRepoAdapter |
| FilesActivity | GitFileAdapter | `list_files` | FileAdapter |
| LogActivity | CommitAdapter | `list_log` | CommitAdapter |
| CommitActivity | DiffAdapter | `list_differences` | DiffEntryAdapter |
| DiffActivity | DiffLineAdapter | `list_diff` | DiffLineAdapter |
| BlameActivity | BlameLineAdapter | `list_blame` | BlameLineAdapter |
| RemotesActivity | RemoteAdapter | `list_remotes` | RemoteAdapter |
| RemoteActivity | RefSpecAdapter | `list_refspec` | RefSpecAdapter |
| PickerActivity | FolderAdapter | `list_folders` | PickerAdapter |
| TagListDialog | TagAdapter | (dialog list) | TagAdapter |
| StashListDialog | StashAdapter | (dialog list) | StashAdapter |
| BranchDrawer | BranchAdapter | `list_branches` | BranchAdapter |

### Migration Strategy

```
Phase 1: No RecyclerView changes (keep ListView working)
Phase 3: Convert ONE screen at a time, starting with lowest risk
```

**Recommended conversion order** (lowest risk first):

1. **DiffLineAdapter** → `DiffActivity` is read-only, no side effects
2. **BlameLineAdapter** → `BlameActivity` is read-only
3. **CommitAdapter** → `LogActivity` is read-only display
4. **DiffAdapter** → `CommitActivity` is read-only
5. **FolderAdapter** → `PickerActivity` is file selection only
6. **RemoteAdapter / RefSpecAdapter** → Remote management
7. **BranchAdapter** → Branch list (side drawer)
8. **StashAdapter / TagAdapter** → Dialog lists
9. **GitFileAdapter** → File browser (core, high impact)
10. **ProjectAdapter** → Home screen (last, most visible)

### Adapter Pattern (each conversion)

```kotlin
// Target pattern for ALL adapters:
class FileAdapter(
    private val onFileClick: (GitFile) -> Unit
) : ListAdapter<GitFile, FileAdapter.ViewHolder>(DiffCallback()) {

    class ViewHolder(private val binding: ItemGitFileBinding) :
        RecyclerView.ViewHolder(binding.root)

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val file = getItem(position)
        // bind data
    }

    class DiffCallback : DiffUtil.ItemCallback<GitFile>() {
        override fun areItemsTheSame(old: GitFile, new: GitFile) =
            old.file.absolutePath == new.file.absolutePath
        override fun areContentsTheSame(old: GitFile, new: GitFile) =
            old.states == new.states
    }
}
```

---

## 9. MATERIAL 3 MIGRATION PLAN

### Theme Migration

**Current**:
```xml
<!-- themes.xml - Material3 Light (already partially M3!) -->
<style name="AppTheme" parent="Theme.Material3.Light.NoActionBar">
```

**Target**:
```xml
<!-- themes.xml -->
<style name="AppTheme" parent="Theme.Material3.DayNight.NoActionBar">
    <!-- Dynamic Color -->
    <item name="colorPrimary">@color/md_theme_light_primary</item>
    <item name="dynamicColorThemeOverlay">@style/ThemeOverlay.App.DynamicColors</item>
</style>

<style name="Widget.PocketGitX.Button" parent="Widget.Material3.Button">
    <!-- consistent button styling -->
</style>
```

### Component Replacement Map

| Legacy Component | Material 3 Replacement | Affected Files |
|---|---|---|
| `com.afollestad.material-dialogs` | `MaterialAlertDialogBuilder` | 30+ dialog instances |
| `com.github.castorflex.smoothprogressbar` | `LinearProgressIndicator` | `widget_progress_bar.xml` |
| Custom `FloatingActionButton` | `com.google.android.material.floatingactionbutton.FloatingActionButton` | All FAB usages |
| `androidx.appcompat.widget.Toolbar` | `com.google.android.material.appbar.MaterialToolbar` | All activity layouts |
| `androidx.appcompat.app.AppCompatActivity` | `MaterialActivity` or Material3 theme | UpdatableActivity |
| `ListPopupWindow` / Spinner | `ExposedDropdownMenuBox` (Compose) or Material3 Dropdown | Remote selection |
| `androidx.preference.SwitchPreference` | Material3 Switch | Settings |
| `androidx.preference.EditTextPreference` | Material3 TextField dialog | Settings |
| `androidx.drawerlayout.widget.DrawerLayout` | `NavigationView` + `ModalDrawerLayout` or Bottom Nav | FilesActivity drawer |
| Legacy colors | Material Color roles + Dynamic Color | `colors.xml` |
| Old adaptive icons | New adaptive icons | mipmap resources |

### Migration Order

```
Phase 3a: Theme only (no functional changes)
  ├── Update themes.xml to use DayNight theme
  ├── Add Dynamic Color support
  ├── Update colors.xml to Material3 color roles
  └── Results: App picks up M3 styling with minimal code change

Phase 3b: Progress bar
  ├── Replace SmoothProgressBar with LinearProgressIndicator
  └── One layout change, one resource string removed

Phase 3c: FAB replacement
  ├── Add Material FAB to build.gradle (already included in material:1.6.1)
  ├── Remove custom FloatingActionButton.java
  └── Replace all usages with Material FAB

Phase 3d: Dialog replacement
  ├── Add MaterialAlertDialogBuilder helper
  ├── Replace one dialog at a time (start with simplest: ConfirmDialog)
  └── Remove material-dialogs dependency

Phase 3e: Toolbar replacement
  └── Replace all Toolbar with MaterialToolbar

Phase 4+: Full screen redesigns with M3 components
```

---

## 10. AI ASSISTANT INTEGRATION PLAN

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     AIScreen (Fragment)                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  ChatBubble (incoming)  ChatBubble (outgoing)        │  │
│  │  ┌────────────────────┐ ┌────────────────────┐       │  │
│  │  │ AI explanation     │ │ User: explain this │       │  │
│  │  │ of file...         │ │ file to me...      │       │  │
│  │  └────────────────────┘ └────────────────────┘       │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  [Input field]  [Send]  [Attach file] [Attach code] │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │ observes
┌──────────────────────────▼──────────────────────────────────┐
│                      AIViewModel                             │
│  ├── chatMessages: StateFlow<List<ChatMessage>>              │
│  ├── isLoading: StateFlow<Boolean>                           │
│  ├── sendMessage(text, context?)                             │
│  ├── explainFile(filePath)                                   │
│  ├── explainCommit(commitHash)                               │
│  └── generateCommitMessage(stagedFiles)                      │
└──────────────────────────┬──────────────────────────────────┘
                           │ calls
┌──────────────────────────▼──────────────────────────────────┐
│                      AIService (interface)                    │
│  ├── sendPrompt(prompt, context): Flow<String>               │
│  ├── OpenAICompatibleService (OpenAI, Anthropic, Ollama)    │
│  ├── LocalModelService (ML Kit, TensorFlow Lite optional)    │
│  └── OfflineService (when no network, limited responses)    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                   AIPromptBuilder                             │
│  ├── buildExplainProjectPrompt(project)                      │
│  ├── buildExplainFilePrompt(filePath, content)               │
│  ├── buildExplainFunctionPrompt(functionCode)                │
│  ├── buildFindBugsPrompt(filePath, content)                  │
│  ├── buildCodeReviewPrompt(diff)                             │
│  ├── buildCommitMessagePrompt(stagedChanges)                 │
│  ├── buildGenerateReadmePrompt(project)                      │
│  └── buildSuggestImprovementsPrompt(project, stats)          │
└─────────────────────────────────────────────────────────────┘
```

### Prompt Templates (examples)

```
EXPLAIN FILE:
"You are a senior developer. Explain the following file from a Git repository.
File: {filePath}
Repository: {repoName}
Content:
```{language}
{fileContent}
```
Please provide:
1. What this file does
2. Key functions/classes
3. Dependencies and relationships
4. Any issues or improvements"

GENERATE COMMIT MESSAGE:
"Generate a concise, descriptive git commit message for the following changes:

Files changed:
{stagedFiles}

Diff summary:
{diffSummary}

Use conventional commits format (feat/fix/docs/refactor/test)."

FIND BUGS:
"Review this code for potential bugs, security vulnerabilities, and performance issues:
{fileContent}

Focus on:
- Null pointer risks
- Resource leaks
- Concurrency issues
- Security vulnerabilities
- Performance bottlenecks"
```

### Integration Timing

| Phase | Component | What's Built |
|---|---|---|
| Phase 0 | `AIService` interface | Contract definition only |
| Phase 4a | `AIScreen` + `AIViewModel` | UI shell (no actual AI) |
| Phase 4b | `AIPromptBuilder` | Prompt generation logic |
| Phase 4c | `OpenAICompatibleService` | First AI provider |
| Phase 4d | Integration actions | Commit message gen, file explain |
| Phase 5 | Provider selection | Settings UI for API key/config |

**No-AI fallback**: AI screen shows "Configure AI provider in Settings" with instructions when no provider is configured.

---

## 11. RISK ASSESSMENT MATRIX

| # | Change | Risk Level | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Room database migration | **MEDIUM** | Data loss if migration fails | Test on copy of DB first; keep backup |
| R2 | SSH StrictHostKeyChecking fix | **HIGH** | Breaking SSH connections | Make configurable; opt-in initially |
| R3 | SSL verification fix | **HIGH** | Breaking HTTPS with self-signed certs | Make configurable; warning on failure |
| R4 | EncryptedSharedPreferences | **MEDIUM** | Locked out of stored passwords | Export+verify before migration |
| R5 | IntentService → WorkManager | **MEDIUM** | Background ops may not transfer | Run both in parallel during migration |
| R6 | AsyncTask → Coroutines | **LOW** | Threading bugs | Structured concurrency is safer |
| R7 | ListView → RecyclerView | **LOW** | Visual differences | Test all screens after conversion |
| R8 | Material Dialogs removal | **LOW** | Minor visual diffs in dialogs | Replace one by one |
| R9 | FilesActivity extraction | **HIGH** | Breaking existing features | Extract to new files; keep originals in place |
| R10 | Navigation Component | **MEDIUM** | Deep link breakage | Test all navigation paths |

### Rollback Strategy

Each phase must be **reversible**:
- Database changes: Keep backup of existing DB
- File changes: Keep original files until new system is verified
- Gradle changes: Keep old `build.gradle` as `.backup`
- New features: Behind feature flags when possible

---

## 12. FILE MODIFICATION STRATEGY

### Files to Modify First (Phase 0 - Foundation)

| File | Change | Buildable? |
|---|---|---|
| `build.gradle` (root) | AGP 8.5, Gradle 8.7 | ✅ |
| `settings.gradle` | Remove JCenter, add version catalog | ✅ |
| `app/build.gradle` | Add Room, Hilt, Coroutines, Compose deps | ✅ |
| `gradle.properties` | Update JVM args | ✅ |
| `proguard-rules.pro` | Add Room/Retrofit/R8 rules | ✅ |

### Files to Never Touch (keep as-is)

| File | Reason | Instead |
|---|---|---|
| `GitUtils.java` | All Git operations depend on it | Wrap with `GitOperations.kt` |
| `CredentialStorage.java` | Complex credential flow | Create `CredentialManager.kt` as new layer |
| `GitClone.java` | Clone with retry logic | Wrap in `CloneWorker.kt` |
| `GitPull.java` | Pull service | Wrap in `PullWorker.kt` |
| `GitPush.java` | Push service | Wrap in `PushWorker.kt` |
| `GitFetch.java` | Fetch service | Wrap in `FetchWorker.kt` |
| `Project.java` | Data model used everywhere | Convert to Kotlin data class **last** |

### Extraction Order for FilesActivity

```
Phase 1: 
  Extract inner classes to separate files
  (SearchTask, anonymous listeners)

Phase 2:
  Extract git state calculation → FileStateCalculator.java
  (calculateFileStates, contains, containsChild, containsParent)
  
Phase 3:
  Extract branch operations → BranchManager.java/Kt
  (initializeBranches, actionCheckoutLocal, actionCheckoutRemote,
   optionCreateBranch, optionDeleteBranch, actionDeleteBranch)

Phase 4:
  Extract remote operations → RemoteActionManager.java/Kt
  (optionPull, optionFetch, optionPush, optionTagPush,
   executePull, executeFetch, executePush, executePushTags)

Phase 5:
  Extract stash operations → StashManager.java/Kt
  (optionStashCreate, optionStashList, optionStashApply, actionStashDelete)

Phase 6:
  Extract tag operations → TagManager.java/Kt
  (optionTagCreate, optionTagList)

Phase 7:
  Extract file operations → FileActionManager.java/Kt
  (actionStage, actionUnstage, actionRevertFiles, actionDeleteFiles,
   actionRemoveFiles, optionNewFile, optionNewFolder)

Phase 8:
  Extract commit flow → CommitManager.java/Kt
  (commit, getStagedChanges, optionSetAuthor)

Phase 9:
  Remaining FilesActivity = UI shell (~400 lines)
  → Convert to FilesFragment + FilesViewModel
```

---

## 13. PHASE RECOMMENDATIONS

### Phase 0 (RECOMMENDED FIRST IMPLEMENTATION)

**Name**: Foundation & Build System Upgrade  
**Duration**: ~1 week  
**Buildable**: ✅ Compiles independently  
**Risk**: LOW (no functional changes)  

**Changes**:

| File | Before | After |
|---|---|---|
| `build.gradle` (root) | AGP 7.0.1, Gradle 7.0.1 wrapper | AGP 8.5.0, Gradle 8.7 wrapper |
| `app/build.gradle` | No version catalog, old deps | Version catalog, updated deps |
| `settings.gradle` | Includes JCenter | MavenCentral only, version catalog |
| `gradle.properties` | Basic config | Updated JVM, parallel build |
| `proguard-rules.pro` | Empty | Room/Retrofit/Glide rules |
| Add `gradle/libs.versions.toml` | — | Version catalog |
| Add `app/proguard-rules.pro` | — | Security rules |

**Why Phase 0 first?**:
1. Zero risk to existing functionality
2. Every subsequent phase benefits from modern build system
3. Version catalog makes dependency updates easy
4. Sets the foundation for all other work
5. Can be verified by `./gradlew build` alone

**Exact dependency versions to use**:

```toml
[versions]
agp = "8.5.2"
kotlin = "2.0.0"
compose-bom = "2024.06.00"
room = "2.6.1"
hilt = "2.51.1"
coroutines = "1.8.1"
jgit = "6.2.0.202206071550-r"  # SAME - preserve existing
glide = "4.16.0"
androidx-appcompat = "1.7.0"
material = "1.12.0"
androidx-activity = "1.9.0"
androidx-fragment = "1.7.1"
navigation = "2.7.7"
work-manager = "2.9.0"
security-crypto = "1.1.0-alpha06"
```

### Alternate Starting Point: Phase 1 (Lowest Risk Code Change)

If build upgrade is too aggressive, start with:

**Name**: Low-Hanging Fruit Fixes  
**Duration**: ~2 days  
**Files changed**: 4  

1. `MD5Util.java` → Replace CP1252 with UTF-8, replace StringBuffer with StringBuilder
2. `CommitActivity.java:75` → Change `http://www.gravatar.com` to `https://www.gravatar.com`
3. `GitFetch.java:14` → Fix copy-paste bug ("Git Pull Service" → "Git Fetch Service")
4. `DiffLine.java:17` → Fix `Marker.ANY_NON_NULL_MARKER` to `"+"`

---

## 14. APPROVAL REQUEST

### Recommended First Phase: **Phase 0 - Foundation**

**Rationale**:
- Zero risk to existing functionality
- Sets up proper build infrastructure
- Enables Kotlin, Room, Hilt, Navigation in subsequent phases
- Can be verified with `./gradlew build`
- Every developer benefit

**What Phase 0 does NOT touch**:
- ❌ No Git operations
- ❌ No UI code
- ❌ No database code
- ❌ No business logic
- ❌ No behavior changes whatsoever

**Phase 0 deliverable**: `./gradlew build` succeeds with modern toolchain.

---

### Ready for your decision. Do you approve Phase 0 as the first implementation step?
