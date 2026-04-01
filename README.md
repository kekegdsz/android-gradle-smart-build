# Android Gradle Smart Build Optimization

[中文文档](https://github.com/kekegdsz/android-gradle-smart-build/blob/main/README_CN.md)

# 🚀 A Safe, Rollback-Safe Android Gradle Build Pruning Solution
Based on Git Diff + TaskGraph for intelligent compilation acceleration. **Automatically falls back to a full build if pruning fails**. Suitable for medium-to-large Android projects / local or CI server builds. Reduces project build time from 10 minutes to 10 seconds.

---

## ✨ Key Features

- **Git Diff-driven module-level pruning**
- **Dependency expansion**: if Git marks a library module as changed, any Android module that depends on it via `implementation` / `api` / `compileOnly` is also compiled, reducing missed recompiles when only the library’s public API changed. Disable with `-PsmartBuildExpandDependents=false`.
- **Fail-Safe Mechanism**: If pruning fails → automatically triggers a full build next time
- **Zero Intrusive**: No need to modify any module source code
- **Significantly reduces build time**
- Supports Kotlin / Java / KAPT / KSP
- Suitable for multi-module Android projects
- **(Optional) Auto-recovery for DEX duplicate-class failures**: on `mergeProjectDex` / `DexArchiveMergerException`, automatically runs `clean` and retries the same task list (see below; enabled when you `apply` the script)

---

## 📅 Changelog

### 2026-03-28

**What changed**

- **`build-optimization.gradle` (robustness & behavior)**
  - **Cross-platform Git invocation**: on Windows uses `cmd.exe /c git ...`; on Unix invokes `git` directly.
  - **Avoid pipe deadlocks**: stdout/stderr from Git child processes are drained on background threads as UTF-8 before `waitFor()`, so large output cannot block the pipe and hang the build.
  - **`execGit` helper**: centralizes the above for HEAD/branch bookkeeping and multiple `git diff` calls.
  - **Changed-module path parsing**: normalizes Windows backslash paths to `/` before splitting path segments, keeping module detection consistent with Unix and reducing incorrect “unchanged module” pruning.
  - **Dependency expansion**: after Git-derived changed modules, closure over `ProjectDependency` on `implementation` / `api` / `compileOnly`. Turn off with `-PsmartBuildExpandDependents=false` or `smartBuildExpandDependents=false` in `gradle.properties`.
  - **Configurable defaults**: pruning/Git/Kotlin/Java/Test/Android and `build/` state filenames are edited in the **`SMART_BUILD` map** at the top of `build-optimization.gradle` (see §2.1).
- **`dex_dup_auto_recovery.gradle` (new)**
  - Runs only when **`apply`'d on the root project**; subprojects do not register the hook.
  - If the build fails with a **D8 / `mergeProjectDex` duplicate-class** pattern (e.g. `DexArchiveMergerException`, `defined multiple times`, etc.), spawns a child process to run **`clean`** plus the **original task list**. Child stdout/stderr are streamed on background threads (avoids silent Daemon I/O and pipe stalls).
  - Disable with **`-PdexDupAutoRecovery=false`** or `dexDupAutoRecovery=false` in project properties. Inner retries use `-PdexDupAutoRecoveryInner=true` to prevent recursive recovery.

**Why it helps**

- More reliable Git usage on **Windows CI** and when **diff output is large**; smarter pruning with consistent path handling.
- Fewer manual **clean + rebuild** cycles when hitting transient **DEX merge duplicate-class** issues (skip `apply` or use the disable flag if unwanted).

---

## 📦 Use Cases

✅ **Recommended for**

- Multi-module Android projects
- Local or CI server builds (Jenkins / GitLab CI / GitHub Actions)
- Daily debug / development builds
- Projects with build times > 5 minutes

❌ **Not recommended for**

- Single-module or very small projects
- Projects not managed by Git
- Projects heavily relying on runtime reflection / dynamic code generation (use with caution)

---

## 🧠 How It Works

### 1️⃣ Git State Awareness

- Records the previous:
  - Git HEAD
  - Git branch
- Automatically triggers a full build if any change is detected (ensures safety)

### 2️⃣ Git Diff → Module Mapping

```text
git diff --name-only
  ↓
moduleA/src/...
  ↓
:moduleA

```

```text
🧠 Build Strategy Evaluation:
  Git HEAD Changed: false
  Git Branch Changed: false
  Last Pruning Failed: false
  Allow Task Pruning: true

📌 Git Changed Modules:
  :component-biz:biz-debug-tool

📌 Dependent recompile (implementation / api / compileOnly):
  :app
```

### 3️⃣ Dependency expansion (optional disable)

After Git change detection, the script walks **`ProjectDependency`** edges on **`implementation` / `api` / `compileOnly`** and adds consuming Android modules transitively. **No full-tree file scan and no MD5**—only a configuration pass.

Use **`-PsmartBuildExpandDependents=false`** when you need to compare behavior to “Git-only” pruning.

# Android Gradle Build Optimization Script - Usage Guide

## 🚀 Quick Integration

### 1️⃣ Copy the scripts
Copy `build-optimization.gradle` (and optionally `dex_dup_auto_recovery.gradle`) to your project root (next to `settings.gradle`):

```text
project-root/
├── build-optimization.gradle
├── dex_dup_auto_recovery.gradle   # optional: DEX duplicate-class auto-recovery
├── settings.gradle
├── app/
└── ...
```

### 2️⃣ Apply in root `build.gradle`

```groovy
apply from: rootProject.file("build-optimization.gradle")
```

⚠️ **Ensure the path is correct, otherwise Gradle will fail to load the script.**

### 2️⃣.1 In-script `SMART_BUILD` map

Defaults live at the **top of `build-optimization.gradle`** in **`def SMART_BUILD = [ ... ]`**. Edit lists/strings there for your project; you do **not** need a second copy in root `build.gradle`.

| Key | Type | Purpose |
|-----|------|---------|
| `skipTaskNameSubstrings` | `List<String>` | Disable task if name contains any entry |
| `pruneTaskPathSubstrings` | `List<String>` | Task participates in “unchanged module” pruning if `task.path` contains any entry |
| `assembleTaskNamePrefixes` | `List<String>` | Pruning logic runs only when some task name starts with one of these prefixes |
| `gitDiffArgLists` | `List<List<String>>` | Multiple `git` argument lists |
| `modulePathStopSegment` | `String` | Stop parsing repo path at this directory segment (default `src`) |
| `dependentResolutionConfigurations` | `List<String>` | Configurations scanned for `ProjectDependency` expansion |
| `kotlinCompileTaskClassNames` | `List<String>` | Kotlin compile task classes (empty list skips Kotlin tuning) |
| `kotlinJvmTarget` / `kotlinFreeCompilerArgs` / `kotlinAllWarningsAsErrors` / `kotlinSuppressWarnings` / `kotlinIncremental` | — | Kotlin options |
| `javaCompileIncremental` / `javaCompileFork` / `javaCompileMemoryMax` | — | Java compile |
| `testForkEvery` / `testMaxParallelForksDivisor` | — | Test forks = processors / divisor |
| `androidAaptCruncherEnabled` | `Boolean` | AAPT cruncher (default `false`) |
| `gitStreamJoinTimeoutMs` | `Long` | Join timeout for Git stream reader threads |
| `trimFailedFlagFileName` / `gitHeadRecordFileName` / `gitBranchRecordFileName` | `String` | Filenames under `build/` |

### 3️⃣ (Optional) DEX duplicate-class auto-recovery — `dex_dup_auto_recovery.gradle`

Typical for **CI or local builds**: when D8 fails on duplicate classes during `mergeProjectDex`, the script runs `clean` and retries the same Gradle tasks you passed on the command line.

1. Place `dex_dup_auto_recovery.gradle` in the project root (as above).
2. In root `build.gradle`, **`apply` `build-optimization.gradle` first**, then **`apply` this script**:

```groovy
apply from: rootProject.file("build-optimization.gradle")
apply from: rootProject.file("dex_dup_auto_recovery.gradle")
```

3. **Turn off auto-recovery** when you need the first failure preserved: pass `-PdexDupAutoRecovery=false` or set `dexDupAutoRecovery=false` in `gradle.properties`.

**Note**: recovery starts a **new Gradle process** for `clean` + the original `taskNames`; the exit code of that process is propagated (your outer invocation may end non-zero). If you do not want this behavior, omit `apply` for this file.

## 📂 Generated Cache Files

| File | Purpose |
|------|---------|
| `build/last_git_head.txt` | Previous Git HEAD (filename: `SMART_BUILD.gitHeadRecordFileName` in the script) |
| `build/last_git_branch.txt` | Previous branch (`gitBranchRecordFileName`) |
| `build/last_trim_failed.flag` | Pruning failure flag (`trimFailedFlagFileName`) |

## 💡 Tip
These cache files can be safely deleted. The script will regenerate them during the next build without affecting functionality.

If dependency expansion compiles more modules than you expect, disable it with **`-PsmartBuildExpandDependents=false`** or **`smartBuildExpandDependents=false`** in `gradle.properties`.

## License
```text
Copyright [2026] [keke of copyright owner]

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

```

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=kekegdsz/android-gradle-smart-build&type=date&legend=top-left)](https://www.star-history.com/#kekegdsz/android-gradle-smart-build&type=date&legend=top-left)
