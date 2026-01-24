# Android Gradle Smart Build Optimization

[English](https://github.com/kekegdsz/android-gradle-smart-build/blob/main/README.md)


🚀 A Safe, Rollback-Compatible Android Gradle Build Pruning Solution​

A smart compilation acceleration based on Git Diff + TaskGraph, with automatic fallback to full compilation on pruning failure. Suitable for medium to large Android projects / local server builds, reducing project compilation from minutes to 10 seconds.

✨ Key Features
Git Diff-driven module-level pruning
Fail-Safe mechanism: Pruning failure → Automatic fallback to full compilation next time
Zero intrusion: No need to modify any module source code
Significantly reduces build time
Supports Kotlin / Java / KAPT / KSP
Compatible with multi-module Android projects
📦 Applicable Scenarios

✅ Recommended for
Multi-module Android projects
Local or server builds (Jenkins / GitLab CI / GitHub Actions)
Daily Debug / Dev builds
Projects with build times > 5 minutes
❌ Not recommended for
Single-module or very small projects
Projects not managed by Git
Projects heavily reliant on runtime reflection / dynamic code generation (use with caution)
🧠 How It Works

1️⃣ Git State Awareness
Records the last:
Git HEAD
Git branch
If changes are detected → Automatically performs full compilation (ensures safety)
2️⃣ Git Diff → Module Mapping
text
复制
git diff --name-only
↓
moduleA/src/...
↓
:moduleA
Android Gradle Build Optimization Script - Usage Guide

🚀 Quick Setup

1️⃣ Copy the Script

Copy build-optimization.gradleto the project root directory (same level as settings.gradle):
text
复制
project-root/
├── build-optimization.gradle
├── settings.gradle
├── app/
└── ...
2️⃣ Include in Root build.gradle

Groovy:
groovy
复制
apply from: rootProject.file("build-optimization.gradle")
⚠️ Ensure the path is correct, otherwise Gradle cannot load the script.

📂 Generated Cache Files
File
Purpose
build/last_git_head.txt
Records the last Git HEAD to detect code changes
build/last_git_branch.txt
Records the last Git branch to detect branch switches
build/last_trim_failed.flag
Records if the last pruned build failed; if present, automatically performs full compilation next time
💡 Note

These files can be safely deleted. The script will regenerate them during the next build without affecting functionality.

