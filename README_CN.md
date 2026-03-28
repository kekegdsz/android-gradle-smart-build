# Android Gradle Smart Build Optimization

[English](https://github.com/kekegdsz/android-gradle-smart-build/blob/main/README.md)


🚀 **一个安全、可回退的 Android Gradle 构建裁剪方案**  
基于 Git Diff + TaskGraph 的智能编译加速，**裁剪失败自动回退全量编译**，适合中大型 Android 工程 /本地 Server 构建，项目编译从十分钟到10秒。

---

## ✨ 特性亮点

- **Git Diff 驱动的模块级裁剪**
- **Fail-Safe 机制**：裁剪失败 → 下次自动全量编译
- **零侵入**：无需改任何模块源码
- **显著缩短 构建时间**
- 支持 Kotlin / Java / KAPT / KSP
- 适用于多 Module Android 工程
- **（可选）DEX 重复类失败自动恢复**：`mergeProjectDex` / `DexArchiveMergerException` 时自动 `clean` 并重跑同一组任务（见下文，`apply` 本脚本即启用）

---

## 📅 更新日志

### 2026-03-28

**改动概要**

- **`build-optimization.gradle`（优化与健壮性）**
  - **Git 子进程跨平台**：Windows 使用 `cmd.exe /c git ...` 调用 Git，Unix 直接 `git`，避免无 Bash 环境失败。
  - **避免管道死锁**：对 Git 子进程的 stdout/stderr 使用独立线程按 UTF-8 消费，再 `waitFor()`，防止输出过多时管道写满导致构建挂死。
  - **统一封装 `execGit`**：上述行为集中在一处，供 HEAD/分支记录与多路 `git diff` 复用。
  - **变更模块路径解析**：将 Git 在 Windows 上可能输出的反斜杠路径规范为 `/`，再按路径段解析到 Gradle 模块名，与 Unix 行为一致，减少误判「未变更模块」。
- **`dex_dup_auto_recovery.gradle`（新增）**
  - 仅在**根工程** `apply` 后生效；子工程不会重复注册。
  - 构建失败且堆栈符合 **D8 / `mergeProjectDex` 重复类**（如 `DexArchiveMergerException`、`defined multiple times` 等）时，自动以子进程执行 **`clean` + 原任务列表** 全量重试；子进程 stdout/stderr 同样用后台线程刷日志，避免 Daemon 下无输出或管道阻塞。
  - 通过 **`-PdexDupAutoRecovery=false`**（或工程属性 `dexDupAutoRecovery=false`）可关闭；内部重试通过 `-PdexDupAutoRecoveryInner=true` 防止递归触发。

**优化效果**

- Git 相关逻辑在 **Windows CI / 大 diff 输出** 场景下更稳定，智能裁剪对路径格式的兼容性更好。
- 遇到 **DEX 合并重复类** 这类常见「脏构建」问题时，可减少人工 `clean` 再跑的步骤（不需要时勿 `apply` 或关闭开关）。

---

## 📦 适用场景

✅ **推荐使用**

- 多 Module Android 工程
- local or Server 构建（Jenkins / GitLab CI / GitHub Actions）
- 日常 Debug / Dev 构建
- 构建时间 > 5 分钟的项目

❌ **不推荐**

- 单模块或极小工程
- 非 Git 管理项目
- 强依赖运行期反射 / 动态代码生成的工程（需谨慎）

---

## 🧠 工作原理概览

### 1️⃣ Git 状态感知

- 记录上一次：
    - Git HEAD
    - Git 分支
- 发生变化 → 自动全量编译（保证安全）

### 2️⃣ Git Diff → 模块映射

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
```

# Android Gradle 构建优化脚本使用说明

## 🚀 快速接入

### 1️⃣ 拷贝脚本
将 `build-optimization.gradle`（以及按需的 `dex_dup_auto_recovery.gradle`）拷贝到项目根目录（与 `settings.gradle` 同级）：

```text
project-root/

├── build-optimization.gradle
├── dex_dup_auto_recovery.gradle   # 可选：需要 DEX 重复类自动恢复时加入

├── settings.gradle

├── app/

└── ...
```

### 2️⃣ 在 Root build.gradle 引入

```groovy
apply from: rootProject.file("build-optimization.gradle")
```

⚠️ **确保路径正确，否则 Gradle 无法加载脚本。**

### 3️⃣（可选）DEX 重复类自动恢复 — `dex_dup_auto_recovery.gradle`

适用于 **CI 或本机构建**：当 D8 合并 dex 报「类重复 / `mergeProjectDex`」时自动 `clean` 并重跑本次传入的 Gradle 任务。

1. 将 `dex_dup_auto_recovery.gradle` 放到根目录（与上表一致）。
2. 在根 `build.gradle` 中 **先** `apply` `build-optimization.gradle`，**再** `apply` 本脚本：

```groovy
apply from: rootProject.file("build-optimization.gradle")
apply from: rootProject.file("dex_dup_auto_recovery.gradle")
```

3. **关闭自动恢复**（例如排查问题时要保留首次失败现场）：命令行增加 `-PdexDupAutoRecovery=false`，或在 `gradle.properties` 中设置 `dexDupAutoRecovery=false`。

**说明**：自动恢复会启动新的 Gradle 子进程执行 `clean` 与原有 `taskNames`，成功或失败后子进程退出码会传递给当前进程（可能以非 0 结束）；不需要该行为时请勿 `apply` 本文件。

## 📂 生成的缓存文件说明

| 文件 | 作用 |
|------|------|
| `build/last_git_head.txt` | 记录上一次 Git HEAD，用于判断代码是否变化 |
| `build/last_git_branch.txt` | 记录上一次 Git 分支，用于判断分支是否切换 |
| `build/last_trim_failed.flag` | 记录上一次裁剪构建是否失败，若存在则下次自动全量编译 |

## 💡 提示
这些文件可安全删除，脚本会在下一次构建时重新生成，不影响功能。

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