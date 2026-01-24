# Android Gradle Smart Build Optimization

[中文文档](https://github.com/kekegdsz/android-gradle-smart-build/blob/main/README_CN.md)

# 🚀 A Safe, Rollback-Safe Android Gradle Build Pruning Solution
Based on Git Diff + TaskGraph for intelligent compilation acceleration. **Automatically falls back to a full build if pruning fails**. Suitable for medium-to-large Android projects / local or CI server builds. Reduces project build time from 10 minutes to 10 seconds.

---

## ✨ Key Features

- **Git Diff-driven module-level pruning**
- **Fail-Safe Mechanism**: If pruning fails → automatically triggers a full build next time
- **Zero Intrusive**: No need to modify any module source code
- **Significantly reduces build time**
- Supports Kotlin / Java / KAPT / KSP
- Suitable for multi-module Android projects

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

text
git diff --name-only
↓
moduleA/src/...
↓



---

# Android Gradle Build Optimization Script - Usage Guide

## 🚀 Quick Integration

### 1️⃣ Copy the Script
Copy the `build-optimization.gradle` file to your project root directory (at the same level as `settings.gradle`):

text
project-root/
├── build-optimization.gradle
├── settings.gradle
├── app/
└── ...


### 2️⃣ Apply in Root build.gradle

Add the following line to your root `build.gradle` file:

groovy
apply from: rootProject.file("build-optimization.gradle")


⚠️ **Ensure the path is correct, otherwise Gradle will fail to load the script.**

## 📂 Generated Cache Files

| File | Purpose |
|------|---------|
| `build/last_git_head.txt` | Records the previous Git HEAD to detect code changes |
| `build/last_git_branch.txt` | Records the previous Git branch to detect branch switches |
| `build/last_trim_failed.flag` | Flag indicating if the previous pruned build failed. If present, a full build is triggered automatically next time. |

## 💡 Tip
These cache files can be safely deleted. The script will regenerate them during the next build without affecting functionality.
