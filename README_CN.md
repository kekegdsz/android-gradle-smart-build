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
将 `build-optimization.gradle` 拷贝到项目根目录（与 `settings.gradle` 同级）：

```text
project-root/

├── build-optimization.gradle

├── settings.gradle

├── app/

└── ...
```

### 2️⃣ 在 Root build.gradle 引入


groovy
```text
apply from: rootProject.file("build-optimization.gradle")
```


⚠️ **确保路径正确，否则 Gradle 无法加载脚本。**

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