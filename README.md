# AVD Emulator Detection Testing

Testing RASP (Runtime Application Self-Protection) emulator detection capabilities against ARM-native Android Virtual Devices on Apple Silicon.

## Background

A penetration test finding (5.3.1 — Liveness Process Bypass Using Emulator) demonstrated that an application's liveness detection and facial recognition could be bypassed using an emulator on an Apple Silicon Mac. The attack chain uses AVD with root hiding to evade detection, then pipes a live video feed through a virtual camera to spoof biometric authentication.

The key question: **does RASP emulator detection (specifically Promon SHIELD) catch ARM-native AVD on Apple Silicon, where traditional x86-based emulator fingerprinting doesn't apply?**

## Attack Chain Overview

| Step | Tool | Purpose |
|------|------|---------|
| 1 | Android Studio AVD | ARM-native emulator on Apple Silicon (M3) |
| 2 | rootAVD | Root the AVD system image |
| 3 | Magisk (28.1+) | Root management + Zygisk |
| 4 | Shamiko 1.2.5 | Hide root from target app (denylist) |
| 5 | LSPosed v1.9.2 | Xposed framework for module injection |
| 6 | Hide My Applist v3.6.1 | Hide installed root tools from app queries |
| 7 | OBS Virtual Camera | Pipe live video feed to AVD front camera |

## Setup Guide

### Prerequisites

- macOS with Apple Silicon (M3 or similar)
- Android Studio (latest recommended for Apple Silicon AVD support)
- OBS Studio (for camera spoofing tests)

### 1. Create AVD

- Android Studio → Device Manager → Create Virtual Device
- Select device profile (e.g. Pixel 7)
- Select an ARM system image (API 33, `Google APIs` variant)
- **Important:** Use `google_apis` images, NOT `google_apis_playstore` — play store images have locked system partitions and rootAVD won't patch them
- On Apple Silicon, ARM images are the default — the "Recommended" tab should show compatible images. If greyed out, download them first.

> **Note:** On some Android Studio versions, the "Recommended" tab may not enable the Next button when selecting a system image. Check "ARM Images" or "Other Images" tabs as a workaround. Clicking a different image and clicking back can also un-stick the button.

### 2. Extract Target APK from Real Device

The target app is distributed as split APKs. Extract from a real device:

```bash
# Find the package path
adb -s <device_serial> shell pm path com.example.target

# Pull all split APKs (use -d shortcut for real device)
adb -d pull /data/app/~~<hash>==/com.example.target-<hash>==/base.apk
adb -d pull /data/app/~~<hash>==/com.example.target-<hash>==/split_config.arm64_v8a.apk
adb -d pull /data/app/~~<hash>==/com.example.target-<hash>==/split_config.en.apk
adb -d pull /data/app/~~<hash>==/com.example.target-<hash>==/split_config.th.apk
adb -d pull /data/app/~~<hash>==/com.example.target-<hash>==/split_config.xxhdpi.apk
```

> **Note:** If `adb pull` gives permission denied, try `adb shell cat <path> > output.apk` as a workaround.

> **Note:** Disconnect the real device before working with the emulator to avoid `adb: more than one device/emulator` errors. Use `-d` (real device) or `-e` (emulator) flags to target specific devices if both are connected.

### 3. Install APK on Emulator

```bash
# Install all splits on emulator
adb -e install-multiple base.apk split_config.arm64_v8a.apk split_config.en.apk split_config.th.apk split_config.xxhdpi.apk
```

### 4. Root with rootAVD

```bash
# Set environment
export ANDROID_HOME=~/Library/Android/sdk
export PATH=~/Library/Android/sdk/platform-tools:$PATH

# Clone rootAVD
git clone https://gitlab.com/newbit/rootAVD.git
cd rootAVD

# IMPORTANT: Emulator must be running and be the ONLY adb device connected
# Disconnect any real phones first
adb devices  # verify only emulator-5554 is listed

# Use RELATIVE path (not absolute) from ANDROID_HOME
./rootAVD.sh system-images/android-33/google_apis/arm64-v8a/ramdisk.img

# When prompted, choose stable (latest version, e.g. 30.6)
# rootAVD will patch ramdisk, install Magisk app, and shut down the AVD
```

> **Troubleshooting:**
> - If rootAVD shows help text instead of running → set `ANDROID_HOME` and use relative path
> - If "no ADB connection possible" → ensure emulator is running and no other devices are connected
> - Run `./rootAVD.sh ListAllAVDs` to see exact commands for your installed images

After patching, **Cold Boot** the AVD from Android Studio (Device Manager → ⋮ → Cold Boot Now).

### 5. Configure Magisk

- Open Magisk app on the emulator (installed automatically by rootAVD)
- If Magisk version is old (e.g. 26.4), upgrade:
  - Magisk app should show update available → Install → Direct Install → Reboot
  - Or manually: download latest APK and `adb -e install Magisk.apk`
- Settings → Enable **Zygisk** → Reboot
- **Shamiko requires Magisk Canary 27005+** — upgrade if needed

### 6. Install Shamiko

```bash
# Download Shamiko (v1.2.5 or latest from https://github.com/LSPosed/LSPosed.github.io/releases)
# Push to emulator
adb -e push Shamiko-v1.2.5.zip /data/local/tmp/

# Install via magisk CLI
adb -e shell su -c "magisk --install-module /data/local/tmp/Shamiko-v1.2.5.zip"

# Reboot
adb -e reboot
```

Configure Shamiko:
- Magisk → Settings → **Configure DenyList** → find target app → check **all processes**
- Magisk → Settings → **Enforce DenyList** → **OFF** (Shamiko reads the list but handles hiding itself)
- Magisk → Settings → **Hide the Magisk app** (randomizes package name)

### 7. Install LSPosed (Optional — for Hide My Applist)

```bash
# Download LSPosed zygisk version
# https://github.com/LSPosed/LSPosed/releases/download/v1.9.2/LSPosed-v1.9.2-7024-zygisk-release.zip
adb -e push LSPosed.zip /data/local/tmp/
adb -e shell su -c "magisk --install-module /data/local/tmp/LSPosed.zip"
adb -e reboot
```

After reboot, LSPosed manager should appear as a notification.

### 8. Install Hide My Applist (Optional)

```bash
# Download HMA APK from https://github.com/Dr-TSNG/Hide-My-Applist/releases
adb -e install HMA-v3.6.1.apk
```

Configure:
1. LSPosed → Modules → enable **Hide My Applist** → scope to **System Framework** → Reboot
2. Open HMA → verify "Module Activated"
3. App manage → find target app → enable hide
4. Blacklist: Magisk, LSPosed, Shamiko, HMA itself
5. Reboot

### 9. Disable USB Debugging

- Settings → Developer Options → disable USB debugging
- Some RASP solutions check for this as an indicator

### 10. Test RASP Detection

Launch the app and observe:
- Does the app block/crash/warn? → RASP detected the emulator
- Does the app open a browser to a specific URL? → Internal app-level check (e.g. Flutter root detection plugin)
- Does the app run normally? → Detection gap

### 11. (Optional) Camera Spoofing for Liveness Bypass

- In AVD config, set **Front Camera → Webcam0**
- In OBS, enable **Virtual Camera** with a video source (e.g. live Teams/Zoom call, pre-recorded video)
- OBS Virtual Camera maps to Webcam0 → AVD front camera
- Attempt liveness/facial recognition authentication

> **Note:** Only proceed with camera spoofing if the app shield is bypassed first. If the shield blocks the app, camera spoofing is irrelevant.

## Test Matrix

| # | Test Case | App Shield | App-Level Check | Notes |
|---|-----------|-----------|----------------|-------|
| 1 | AVD only (no root hiding) | ✅ Blocked | N/A | Shield detected emulator immediately |
| 2 | AVD + Magisk + Shamiko + Zygisk | ✅ Blocked | N/A | Shield still detected |
| 3 | AVD + Magisk + Shamiko + LSPosed + HMA | ✅ Blocked | N/A | Xposed injection may add detection surface |
| 4 | Camera spoofing via OBS | — | — | Not tested (shield not bypassed) |

## Detection Layers

The app has multiple detection layers:

1. **App Shield (Promon SHIELD)** — RASP-level emulator/root/tamper detection. This is the primary protection layer.
2. **App-Level Root Check** — Flutter plugin-based check that redirects to a specific URL when root/emulator is detected. Weaker secondary layer.
3. **Google Play Integrity API** — Checks device integrity on specific screens. Supplementary check added after the pentest finding.

> **Important:** If the app shield is bypassed, the app-level check and Play Integrity are significantly weaker controls. The shield is the critical layer.

## Pentest Report Reference (5.3.1)

The original finding used:
- AVD on MacBook Pro M3
- rootAVD
- Magisk 28.1 (28100)
- Shamiko 1.2.1 (383)
- Zygisk enabled
- Hide Magisk enabled
- USB debugging disabled
- OBS Virtual Camera → Webcam0 → AVD front camera
- Live stream from Microsoft Teams meeting as face source

Status at retest (2025-05-20): **Partially Fixed** — Google Play Integrity API added on some screens.

## Key Learnings

- **Always disconnect real devices** before running rootAVD or any emulator-specific adb commands
- **rootAVD requires a relative path** from `ANDROID_HOME`, not an absolute path
- **Shamiko version compatibility matters** — v1.2.5 requires Magisk Canary 27005+, older Shamiko versions work with older Magisk
- **Split APKs** are common for Play Store apps — use `adb install-multiple` to install all splits
- **Cold Boot** is required after rootAVD patching (not just reboot)
- **google_apis vs google_apis_playstore** — only google_apis images can be rooted with rootAVD
- Adding more modules (LSPosed, HMA) can actually **increase** the detection surface for RASP solutions

## Troubleshooting Log

Real issues encountered during setup and testing, documented for future reference.

---

### AVD: Next Button Greyed Out in System Image Selection

**Problem:** When creating an AVD, selecting a system image in the "Recommended" tab doesn't enable the Next button.

**Solution:**
- Check if the system image needs to be **downloaded first** (look for a download icon next to it)
- Try the **"ARM Images"** or **"Other Images"** tabs instead
- Click a different image, then click back to the desired one — this can un-stick the UI
- On Apple Silicon, make sure you're selecting **arm64-v8a** images, not x86_64

> **Note:** On Apple Silicon, images won't be labeled `arm64-v8a` explicitly in the UI since ARM is the only architecture available. In the SDK folder structure they still live under `arm64-v8a/` (relevant for the rootAVD command).

**Recommendation:** Use the latest Android Studio version for best Apple Silicon AVD support.

---

### google_apis vs google_apis_playstore Images

**Problem:** rootAVD fails to patch the system image.

**Cause:** You may have selected a `google_apis_playstore` image, which has a locked/read-only system partition.

**Fix:** Use `google_apis` images (without Play Store). These have an unlockable system partition that rootAVD can patch.

---

### ADB: "more than one device/emulator"

**Problem:** Any `adb` command fails with `adb: more than one device/emulator` when both a real phone and emulator are connected.

**Solution:** Either disconnect the real device or use target flags:
```bash
adb -d ...   # target real device only
adb -e ...   # target emulator only
adb -s <serial> ...  # target specific device by serial
```

rootAVD specifically requires the emulator to be the **only** connected device — it doesn't support `-e` flag. Always disconnect real phones before running rootAVD.

---

### rootAVD: Shows Help Text Instead of Running

**Problem:** Running `./rootAVD.sh ~/Library/Android/sdk/system-images/...` just prints usage info and doesn't execute.

**Solution:** rootAVD expects a **relative path** from `ANDROID_HOME`, not an absolute path:
```bash
# Wrong — absolute path
./rootAVD.sh ~/Library/Android/sdk/system-images/android-33/google_apis/arm64-v8a/ramdisk.img

# Right — relative path with ANDROID_HOME set
export ANDROID_HOME=~/Library/Android/sdk
./rootAVD.sh system-images/android-33/google_apis/arm64-v8a/ramdisk.img
```

Run `./rootAVD.sh ListAllAVDs` to get exact copy-paste commands for all installed images.

---

### rootAVD: "no ADB connection possible"

**Problem:** rootAVD can't connect even though the path is correct.

**Solution:** The emulator must be **running** before executing rootAVD. Boot the AVD from Android Studio first, then verify:
```bash
adb devices
# Should show: emulator-5554   device
```

Also ensure `platform-tools` is in your PATH:
```bash
export PATH=~/Library/Android/sdk/platform-tools:$PATH
```

---

### APK Pull: Permission Denied

**Problem:** `adb pull` from `/data/app/` fails with permission denied.

**Solution:** `adb pull` with the full path often works even when `adb shell ls` doesn't:
```bash
# This may fail
adb -d shell ls /data/app/~~<hash>==/com.example.app/

# But this often works
adb -d pull /data/app/~~<hash>==/com.example.app/base.apk
```

Alternative workaround using `cat` through shell:
```bash
adb -d shell "cat /data/app/~~<hash>==/com.example.app/base.apk" > base.apk
```

---

### Shamiko: "Please install Magisk Canary (>27005)"

**Problem:** Shamiko v1.2.5 refuses to install on Magisk 26.4 (or other older versions).

**Solution:** Upgrade Magisk before installing Shamiko:
- Open Magisk app → if update available, tap Install → Direct Install → Reboot
- Or manually download and install latest Magisk APK:
```bash
adb -e install Magisk-latest.apk
```
- Alternatively, re-run rootAVD and select the latest stable version (e.g. 30.6) instead of the local/older version

> **Note:** Always check Shamiko's release notes for minimum Magisk version requirements. Newer Shamiko versions may require Magisk Canary builds.

---

### Shamiko ZIP: Unzip Error in Magisk UI

**Problem:** Installing Shamiko via Magisk → Modules → Install from storage fails with unzip error.

**Solution:** Install via adb command line instead, bypassing the Magisk UI file picker:
```bash
adb -e push shamiko.zip /data/local/tmp/
adb -e shell su -c "magisk --install-module /data/local/tmp/shamiko.zip"
```

This also works for LSPosed and other Magisk module zips that fail in the UI.

---

### Shamiko Download Source

**Problem:** Shamiko releases move around — the original LSPosed repo may be archived or reorganized.

**Workaround:** Check [LSPosed GitHub releases](https://github.com/LSPosed/LSPosed.github.io/releases) for official builds. Verify checksums before installing. Avoid random third-party download sites.

---

### curl: Downloaded ZIP is Actually HTML

**Problem:** Downloading Shamiko or other GitHub release ZIPs via `curl` results in a broken file (actually an HTML redirect page).

**Solution:** Always use `-L` flag to follow redirects:
```bash
curl -L -o output.zip "https://github.com/.../release.zip"

# Verify the file is actually a zip
file output.zip
# Should say "Zip archive data" — if it says "HTML", the download failed
```

If it still fails, download manually from the browser and push via adb:
```bash
adb -e push ~/Downloads/module.zip /data/local/tmp/
```

Manual browser download is generally more reliable for GitHub release assets.

---

### RASP vs Liveness — Scope Confusion

**Clarification:** During analysis, it's easy to conflate RASP (e.g. Promon SHIELD) with liveness SDK responsibilities. These are separate layers:
- **RASP** blocks the app from running in a hostile environment (emulator, rooted device)
- **Liveness SDK** validates that biometric input is from a real person, regardless of platform

A RASP detection gap doesn't mean the liveness SDK is broken, and vice versa. Test and evaluate them independently.

---

### Adding More Modules Can Increase Detection Surface

**Observation:** During testing, adding LSPosed + Hide My Applist on top of Magisk + Shamiko caused the app shield to detect the environment in cases where Magisk + Shamiko alone might not have triggered detection. Xposed framework injection introduces additional artifacts (zygote modifications, injected classes) that RASP solutions specifically look for.

**Takeaway:** More hiding tools ≠ better hiding. Each additional module adds its own detection surface. Test incrementally — start with the minimum stack and add tools one at a time to identify which combination triggers or evades detection.

## TODO

- [x] Reproduce AVD setup on Apple Silicon Mac
- [x] Confirm rootAVD + Magisk + Shamiko installation
- [x] Test SHIELD detection — emulator only
- [x] Test SHIELD detection — Magisk + Shamiko
- [x] Test SHIELD detection — full stack (LSPosed + HMA)
- [ ] Obtain pentester's exact tool versions and reproduction steps
- [ ] Re-test with pentester's exact configuration
- [ ] Test camera spoofing via OBS (if shield bypass achieved)
- [ ] Share results + logs with Promon
- [ ] Evaluate liveness SDK independently
