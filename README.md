# 🏦 Android Vault

## Android `android:debuggable` & `run-as` Sandbox Research Lab

---

## Overview

**Android Vault** is a purpose-built Android lab application designed to demonstrate how the `android:debuggable` flag impacts Android’s sandbox enforcement and the behavior of the `run-as` command.

This project provides:

* A prebuilt release APK (available under **Releases**)
* A CLI-only walkthrough (no Android Studio required)
* A controlled demonstration of build configuration effects
* A reproducible Android sandbox research lab

The included app is a simple note-taking application that stores data inside:

```
/data/data/com.fresh.forensics.vault/files/secrets.txt
```

The lab demonstrates how changing a single manifest flag alters local debugging behavior.

---

# 🔬 What This Lab Demonstrates

* Android app isolation is UID-based
* `run-as` is restricted to debuggable builds
* Build configuration affects local access behavior
* APK integrity is enforced via signatures
* Rebuilt APKs must be re-signed
* Signature mismatch prevents silent replacement

This is a configuration-based demonstration — not an exploit.

---

# 🔧 Requirements

This lab is entirely CLI-based. Android Studio is **not required**.

## 1️⃣ ADB (Android Debug Bridge)

```bash
adb version
```

Install (Debian/Kali/Ubuntu):

```bash
sudo apt install adb
```

---

## 2️⃣ apktool

Used for decompiling and rebuilding the APK.

```bash
apktool --version
```

Install:

```bash

    Download the Linux wrapper script. (Right click, Save Link As apktool)
    Download the latest version of Apktool.
    Rename the downloaded jar to apktool.jar.
    Move both apktool.jar and apktool to /usr/local/bin. (root needed)
    Make sure both files are executable. (chmod +x)
    Try running apktool via CLI.

```

---

## 3️⃣ Java (JDK)

Required for signing rebuilt APKs.

```bash
java --version
keytool -help
```

Install if needed:

```bash
sudo apt install default-jdk
```

---

## 4️⃣ Android Build Tools (`apksigner`)

Verify:

```bash
apksigner --version
```

---

## 5️⃣ Android Device

* Developer Options enabled
* USB Debugging enabled
* Physical device access
* ADB authorization granted

Verify:

```bash
adb devices
```

---

## ✅ Tested Environment

This walkthrough was tested on:

* Linux (Kali Linux)
* OpenJDK 21
* Android SDK Build Tools 36.x
* ADB (platform-tools)
* apktool.jar & apktool wrapper script

Verify your environment:

```bash
java --version
adb version
apktool --version
apksigner version
```

---

# 📱 Lab Walkthrough (CLI-Only)

## 1️⃣ Download the Release APK

Go to the repository’s **Releases** section and download:

```
FreshForensicsVault_v1.0_release.apk
```

This build is configured with:

```xml
android:allowBackup="false"
```

---

## 2️⃣ Install the APK

```bash
adb install FreshForensicsVault_v1.0_release.apk
```

Launch the app once.

Tap **Load** to confirm the pre-seeded `secrets.txt` content is visible.

---

## 3️⃣ Verify Sandbox Enforcement

Confirm the app is not debuggable:

```bash
adb shell dumpsys package com.fresh.forensics.vault | grep -i debug
```

Expected:

```
NO OUTPUT
```

Attempt `run-as`:

```bash
adb shell run-as com.fresh.forensics.vault
```

Expected:

```
run-as: Package 'com.fresh.forensics.vault' is not debuggable
```

At this stage, Android’s sandbox is functioning as intended.

---

## 4️⃣ Extract the Installed APK

```bash
adb shell pm path com.fresh.forensics.vault
```

Example:

```
package:/data/app/~~abc123/base.apk
```

Pull it:

```bash
adb pull /data/app/~~abc123/base.apk
```

---

## 5️⃣ Decompile with apktool

```bash
apktool d -o FreshVault_Extracted base.apk
```

Open:

```
FreshVault_Extracted/AndroidManifest.xml
```

Add:

```xml
android:debuggable="true"
```

Save.

---

## 6️⃣ Rebuild the APK

```bash
apktool b FreshVault_Extracted -o FreshVault_modified.apk
```

---

## 7️⃣  Align APK with zipalign

```bash
zipaling -v 4 FreshVault_modified.apk -o FreshVault_aligned.apk
```

## 8️⃣  Create a Signing Key (One Time)

```bash
keytool -genkey -v \
-keystore lab.keystore \
-alias labkey \
-keyalg RSA \
-keysize 2048 \
-validity 10000
```

---

## 9️⃣  Sign the Modified APK

```bash
apksigner sign \
--ks lab.keystore \
--ks-key-alias labkey \
FreshVault_aligned.apk
```

Verify:

```bash
apksigner verify FreshVault_aligned.apk
```

---

## 🔟 Replace the Installed App

Uninstall original:

```bash
adb uninstall com.fresh.forensics.vault
```

Install modified version:

```bash
adb install FreshVault_aligned.apk
```

Launch once.

---

## 🔟 Demonstrate `run-as` Access

```bash
adb shell run-as com.fresh.forensics.vault cat files/secrets.txt
```

You should now see the file contents.

Confirm state:

```bash
adb shell dumpsys package com.fresh.forensics.vault | grep -i debug
```

Expected:

```
debuggable=true
```

---

# 🔐 Security Explanation

Android assigns each application a unique Linux UID. Private app data is stored in:

```
/data/data/<package_name>/
```

Access is enforced through:

* Linux filesystem permissions
* SELinux policies
* Signature validation
* Debuggable flag checks

The `run-as` command only works for applications marked as debuggable.

This lab demonstrates how build configuration impacts local debugging behavior — within Android’s intended security model.


                         ┌───────────────────────────┐
                         │         ADB Shell         │
                         │  (User interacting via PC)│
                         └─────────────┬─────────────┘
                                       │
                                       │  run-as (if allowed)
                                       ▼
                        ┌─────────────────────────────┐
                        │     Android OS Framework    │
                        │                             │
                        │  • Package Manager          │
                        │  • Signature Verification   │
                        │  • Debuggable Flag Check    │
                        └─────────────┬───────────────┘
                                      │
                                      │ UID Enforcement
                                      ▼
        ┌────────────────────────────────────────────────────────┐
        │                Linux Kernel Layer                      │
        │                                                        │
        │  Each App Assigned Unique Linux UID                    │
        │                                                        │
        │  Example:                                              │
        │    com.fresh.forensics.vault → UID 10345               │
        │                                                        │
        └───────────────┬────────────────────────────────────────┘
                        │
                        │ Filesystem Ownership
                        ▼
        ┌────────────────────────────────────────────────────────┐
        │  /data/data/com.fresh.forensics.vault/                 │
        │                                                        │
        │  Owned by UID 10345                                    │
        │                                                        │
        │  secrets.txt                                           │
        │                                                        │
        │  Accessible ONLY if:                                   │
        │    • Process runs as UID 10345                         │
        │    • OR app is debuggable and run-as permitted         │
        └────────────────────────────────────────────────────────┘

---

# ⚖️ Ethical & Responsible Use

This repository:

* Contains a self-authored demonstration application
* Does not modify third-party applications
* Does not bypass Android security mechanisms
* Does not exploit vulnerabilities
* Does not escalate privileges

This is a controlled educational lab focused on build configuration and sandbox behavior.

---

## ☕ Support This Project

If **AndroidVault™** helped you better understand Android, consider supporting continued development:

<p align="center">
  <a href="https://www.buymeacoffee.com/dfreshZ" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" style="height: 60px !important;width: 217px !important;" ></a>
</p>

---
<!-- 
 _____              _       _____                        _          
|  ___| __ ___  ___| |__   |  ___|__  _ __ ___ _ __  ___(_) ___ ___ ™️
| |_ | '__/ _ \/ __| '_ \  | |_ / _ \| '__/ _ \ '_ \/ __| |/ __/ __|
|  _|| | |  __/\__ \ | | | |  _| (_) | | |  __/ | | \__ \ | (__\__ \
|_|  |_|  \___||___/_| |_| |_|  \___/|_|  \___|_| |_|___/_|\___|___/
        freshforensicsllc@tuta.com Fresh Forensics, LLC 2026 -->
