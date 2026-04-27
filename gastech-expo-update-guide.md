# GasTech Mobile — Expo OTA Update Guide
> **Project:** `gastech-mobile-v1`  
> **Path:** `D:\EverestX\New folder\gastech-mobile-v1`  
> **Stack:** React Native · Expo SDK 54 · EAS Build & EAS Update  
> **Author-Maintained Doc — Last Updated:** 2026-04-27

---

## Table of Contents

1. [Overview — What is an OTA Update?](#1-overview)
2. [How the Update System Works in This Project](#2-how-the-update-system-works)
3. [Key Files & Their Responsibilities](#3-key-files--their-responsibilities)
4. [Environment Setup — Staging vs Production](#4-environment-setup--staging-vs-production)
5. [The Env Caching Issue (Critical)](#5-the-env-caching-issue--critical)
6. [Step-by-Step: Pushing an OTA Update](#6-step-by-step-pushing-an-ota-update)
7. [Pre-Push Checklist](#7-pre-push-checklist)
8. [Common Mistakes & Troubleshooting](#8-common-mistakes--troubleshooting)
9. [Channel Architecture](#9-channel-architecture)
10. [Runtime Version & When You Cannot Use OTA](#10-runtime-version--when-you-cannot-use-ota)

---

## 1. Overview

**Expo OTA (Over-the-Air) Updates** via `expo-updates` allow you to push JavaScript/asset changes to devices **without going through the Play Store or App Store**. The installed native binary checks an Expo-managed URL on startup, downloads a new bundle if one is available, and swaps it in.

This is the **primary deployment mechanism** for GasTech Mobile. Native rebuilds are only needed when native code, permissions, or the `runtimeVersion` changes.

---

## 2. How the Update System Works

### Update Check Flow

```
App Launched
     │
     ▼
index.ts ──► registerRootComponent(App)
     │
     ▼
App.tsx ──► renders providers + <AppNavigator>
     │
     ▼
AppNavigator.jsx ──► NavigationContainer (initial route = "Splash")
     │
     ▼
SplashScreen.jsx ──► checks session/DB while visible
     │
     ▼
expo-updates (native layer) ──► checks update URL on load
                                (checkAutomatically: "ON_LOAD")
     │
     ├── Update available ──► downloads in background
     │                        app applies on NEXT launch
     └── No update ──────────► continues normally
```

### Update Check Config (`app.config.js`)

```js
updates: {
  url: selectedVariant.updatesUrl,   // Expo update endpoint
  checkAutomatically: "ON_LOAD",     // check every cold start
  fallbackToCacheTimeout: 0,         // don't wait — load cached immediately
}
```

`checkAutomatically: "ON_LOAD"` means the app checks for a new bundle **every time it is opened cold**. With `fallbackToCacheTimeout: 0`, the app always loads the cached JS immediately (no spinner), and if a newer bundle is found, it is applied on the **next** cold open.

---

## 3. Key Files & Their Responsibilities

| File | Role in Update System |
|---|---|
| `app.config.js` | **Central update config.** Selects project ID & update URL based on variant. Exports the `expo.updates.url` and `expo.extra.eas.projectId`. |
| `app.json` | Static fallback config (kept in sync with `app.config.js` for tooling). Contains hardcoded production values. **Always override with `app.config.js`.** |
| `eas.json` | Defines EAS build profiles (`development`, `preview`, `production`) and their **channels**. Each channel maps to a separate update stream. |
| `babel.config.js` | Resolves which `.env` file to inject at **bundle time** via `react-native-dotenv`. Also uses `APP_VARIANT` or git branch to switch env files. |
| `index.ts` | Entry point. Calls `registerRootComponent`. No update logic. |
| `App.tsx` | Mounts context providers. No update logic. |
| `package.json` | Contains `"expo-updates": "~29.0.16"` — the client-side update library. |
| `src/screens/SplashScreen.jsx` | First screen shown. Handles DB init and session check, **not** the update check (update check happens in the native `expo-updates` layer before JS runs). |
| `src/navigation/AppNavigator.jsx` | Wires the root navigator starting at `Splash`. Also manages background sync intervals. |

### `app.config.js` Deep Dive

```js
const APP_VARIANTS = {
  production: {
    name: "GasTechMobile",
    slug: "GasTechMobile",
    updatesUrl: "https://u.expo.dev/af65ddf8-bf52-4856-9eff-cd08773a7bab",
    projectId: "af65ddf8-bf52-4856-9eff-cd08773a7bab",
  },
  stage: {
    name: "GasTechMobileStage",
    slug: "GasTechMobileStage",
    updatesUrl: "https://u.expo.dev/2f94bcdc-e805-4cfb-a4d1-b9e15c833662",
    projectId: "2f94bcdc-e805-4cfb-a4d1-b9e15c833662",
  },
};
```

The variant is resolved by this priority:
1. `APP_VARIANT` environment variable (explicit override)
2. Current git branch name (contains `stage`/`staging` → stage; `main`/`master`/`prod` → production)
3. Defaults to `production`

This means `eas update` will log which project it targets before bundling — **always read this log line to confirm the target.**

---

## 4. Environment Setup — Staging vs Production

The project uses **two separate Expo projects** (two EAS project IDs) and **two `.env` files**:

| Environment | Expo Project | Odoo Backend | `.env` file |
|---|---|---|---|
| **Production** | `af65ddf8-...` (GasTechMobile) | `gas-tech.odoo.com` | `.env.production` |
| **Staging** | `2f94bcdc-...` (GasTechMobileStage) | `gas-tech-stage-*.dev.odoo.com` | `.env.stage` |

### How env files are loaded

`babel.config.js` uses `react-native-dotenv` (module name `@env`) and picks the env file at **bundle compile time**:

```js
// babel.config.js resolution order:
// 1. APP_VARIANT env var
// 2. git branch
// 3. fallback to .env.production
// 4. if preferred file missing → falls back to .env
```

> **Important:** `.env` values are **baked into the JS bundle at build time** by Babel. They are NOT read at runtime. Changing a `.env` file without re-bundling has no effect on a device.

### Current `.env` (root)

```
# Active: Production
ODOO_URL=https://gas-tech.odoo.com/jsonrpc
ODOO_DB=gas-tech-app-30121977
ODOO_API_KEY=ee33984e...
UID=2

# Staging (commented out):
# ODOO_URL=https://gas-tech-stage-30973708.dev.odoo.com/jsonrpc
# ODOO_DB=gas-tech-stage-30973708
# ODOO_API_KEY=bdc8870f...
# UID=10
```

---

## 5. The Env Caching Issue — Critical

### What happened

When switching between staging and production environments, the **Babel cache** and **Metro bundler cache** retained the previously compiled env values. This caused:

- An update pushed with `APP_VARIANT=stage` would still embed production Odoo URLs (or vice versa)
- The `babel.config.js` log line `[babel] Using env file: ...` would show the wrong file if the cache wasn't cleared
- Devices would hit the wrong backend after receiving an OTA update

### Root Cause

`babel.config.js` calls `api.cache(true)` — this caches the Babel config output. When the `APP_VARIANT` changes, Babel doesn't automatically invalidate the cache, so the old env values persist.

Additionally, Metro (the React Native bundler) has its own transform cache independent of Babel.

### The Fix — Always clear caches when switching environments

```bash
# Option 1: Clear Metro cache only (fastest)
npx expo start --clear

# Option 2: Clear both Babel + Metro cache before eas update
npx expo export --clear  # or just rely on --clear flag in eas update

# Option 3 (recommended for eas update): set APP_VARIANT explicitly
# Windows PowerShell:
$env:APP_VARIANT="stage"; eas update --branch staging --message "Fix: ..."
$env:APP_VARIANT="production"; eas update --branch preview --message "Fix: ..."

# Windows CMD:
set APP_VARIANT=stage && eas update --branch staging --message "..."
set APP_VARIANT=production && eas update --branch preview --message "..."
```

### How to verify the correct env was picked

Before running `eas update`, run:

```bash
# Check which env file babel would pick RIGHT NOW
node -e "
const { execSync } = require('child_process');
const branch = execSync('git rev-parse --abbrev-ref HEAD').toString().trim().toLowerCase();
const variant = process.env.APP_VARIANT || '';
console.log('APP_VARIANT:', variant || '(not set)');
console.log('Git branch:', branch);
console.log('Will use:', variant.includes('stage') ? '.env.stage' : branch.includes('stage') ? '.env.stage' : '.env.production');
"
```

Also watch the console output during bundling:
```
[babel] Using env file: .env.production   ← confirm this line!
[app.config] Using production config: GasTechMobile (af65ddf8-...)  ← confirm this too!
```

### Permanent Prevention

To prevent accidental cache use, create separate `.env.production` and `.env.stage` files (do not rely on commenting/uncommenting `.env`):

```
.env.production   ← production Odoo credentials
.env.stage        ← staging Odoo credentials  
.env              ← fallback (keep as production or leave empty)
```

Then Babel auto-picks the right file via `APP_VARIANT` or git branch with zero manual editing.

---

## 6. Step-by-Step: Pushing an OTA Update

### Prerequisites

```bash
npm install -g eas-cli          # install EAS CLI globally
eas login                       # authenticate with your Expo account
```

### Step 1 — Confirm your git branch

```bash
git branch --show-current
# Should be: main / master (→ production)  or  stage/staging (→ staging)
```

### Step 2 — Set APP_VARIANT explicitly (strongly recommended)

```powershell
# For STAGING update:
$env:APP_VARIANT = "stage"

# For PRODUCTION update:
$env:APP_VARIANT = "production"
```

### Step 3 — Verify the config resolves correctly

```bash
node app.config.js
# Watch for: [app.config] Using production config: GasTechMobile (af65ddf8-...)
# or:         [app.config] Using stage config: GasTechMobileStage (2f94bcdc-...)
```

### Step 4 — Push the update

```bash
# Push to STAGING (branch = staging, targets GasTechMobileStage project):
eas update --branch staging --message "feat: your change description"

# Push to PRODUCTION (branch = preview, targets GasTechMobile project):
eas update --branch preview --message "feat: your change description"
```

> **Note:** EAS Update "branches" are independent from git branches. In this project:
> - EAS branch `staging` → staging Expo project (`2f94bcdc-...`)
> - EAS branch `preview` → production Expo project (`af65ddf8-...`)

### Step 5 — Verify on Expo Dashboard

1. Go to [expo.dev](https://expo.dev) → your project → **Updates** tab
2. Confirm the new update appears in the correct branch
3. Check the bundle metadata (runtime version must match installed binary)

### Step 6 — Test on device

1. Kill the app completely (remove from recents)
2. Open the app — it will check for the update on cold start
3. **First open:** app may download update silently (you won't see it yet)
4. **Second open:** the new JS bundle runs
5. Verify the expected changes are visible

---

## 7. Pre-Push Checklist

Go through this checklist **before every `eas update`** command:

### Environment
- [ ] `APP_VARIANT` is explicitly set in the current shell session
- [ ] The git branch matches the intended target environment
- [ ] Confirm `[app.config] Using <variant> config` log line matches your intent
- [ ] Confirm `[babel] Using env file: <file>` log line shows the correct env file
- [ ] The correct `.env.production` / `.env.stage` file exists and has valid credentials

### Code
- [ ] All intended code changes are saved
- [ ] No hardcoded URLs or credentials in source files
- [ ] No debug `console.log` statements with sensitive data
- [ ] Translation keys updated if UI text changed (check `translations/` folder)
- [ ] No `TODO` / `FIXME` left in changed files

### Native Compatibility
- [ ] No new native packages added (if yes, a full native build is required first)
- [ ] `runtimeVersion` in `app.config.js` is unchanged (still `"1.0.0"`)
- [ ] No changes to `android/` directory (gradle, manifests, native modules)
- [ ] No new Expo config plugins added to `plugins[]` in `app.config.js`
- [ ] No new Android permissions added

### EAS / Expo
- [ ] EAS CLI is up to date (`eas --version` ≥ 18.5.0 per `eas.json`)
- [ ] You are logged into the correct Expo account (`eas whoami`)
- [ ] The EAS branch name is correct for the target environment
- [ ] You have a descriptive `--message` for the update (helpful for rollback)

### Post-Push
- [ ] Update visible on [expo.dev](https://expo.dev) dashboard
- [ ] Tested on a physical device (two cold starts)
- [ ] Staging tested first before pushing to production

---

## 8. Common Mistakes & Troubleshooting

### ❌ "Wrong backend after update" (env caching)
**Cause:** Babel cache held old env values.  
**Fix:** `$env:APP_VARIANT = "production"; eas update --branch preview ...`  
Always set `APP_VARIANT` explicitly. Never rely on git branch alone for env selection when doing manual pushes.

### ❌ Update not appearing on device
**Cause:** Device still has cached bundle, or `runtimeVersion` mismatch.  
**Fix:** Kill app fully → reopen twice. If still failing, check runtime version matches in Expo dashboard.

### ❌ "Runtime version mismatch"
**Cause:** You changed `runtimeVersion` in `app.config.js` or added native modules without a new binary build.  
**Fix:** Build and distribute a new APK via `eas build --profile preview`, then push the OTA update.

### ❌ Update pushed to wrong EAS branch
**Cause:** Used wrong `--branch` flag.  
**Fix:** Re-push with the correct branch name. Old update in wrong branch won't affect other branch devices.  
Go to Expo dashboard → Updates → delete the incorrect update if needed.

### ❌ "Project not found" error during eas update
**Cause:** `APP_VARIANT` mismatch causing wrong `projectId` to be embedded.  
**Fix:** Ensure `APP_VARIANT` is set correctly before running `eas update`.

### ❌ Babel cache using old `.env` after switching
**Symptoms:** `[babel] Using env file: .env.stage` but production credentials are loading.  
**Fix:** Delete `.expo/` and `node_modules/.cache/` folders, then re-run.
```powershell
Remove-Item -Recurse -Force .expo
Remove-Item -Recurse -Force node_modules\.cache
```

---

## 9. Channel Architecture

```
EAS Build Profiles (eas.json)          EAS Update Branches
─────────────────────────────          ──────────────────────
development  ── channel: development   (internal, dev client)
preview      ── channel: preview   ──► branch: preview    (→ production OTA)
production   ── channel: preview   ──► branch: preview    (auto-increment build)
                                       branch: staging    (→ staging OTA)
```

**Installed apps receive updates from the EAS branch that matches their build channel.**  
A device built with `eas build --profile preview` is on channel `preview` and will receive updates pushed to `--branch preview`.

A staging device must be built with a custom profile using `channel: staging` to receive staging-only OTA updates.

---

## 10. Runtime Version & When You Cannot Use OTA

`runtimeVersion: "1.0.0"` is set in `app.config.js`.

OTA updates **only work** when the new bundle's `runtimeVersion` matches the installed binary. If you change `runtimeVersion`, all existing installs will **not receive** that OTA update.

### You MUST do a full native build (not just eas update) when:

| Change | Reason |
|---|---|
| New npm package with native code (e.g., new expo module) | Changes native binary |
| Changes to `android/` directory (gradle, manifests, jar files) | Changes native binary |
| New entry in `plugins[]` in `app.config.js` | Config plugin modifies native layer |
| New Android `permissions` added | Embedded in AndroidManifest.xml |
| `runtimeVersion` bumped | Existing installs reject the bundle |
| `expo` SDK version upgrade | New SDK may require new binary |
| Changes to `newArchEnabled`, `edgeToEdgeEnabled` | React Native arch settings |

### You CAN use OTA (eas update) for:

- UI changes, layout fixes, style updates
- Business logic, service layer, API call changes
- Screen additions/removals (if no new native deps)
- Translation file changes
- Bug fixes in JavaScript
- New screens using existing native components
- Asset changes (images, fonts already bundled)
- Navigation structure changes

---

## Quick Reference Commands

```powershell
# Authenticate
eas login
eas whoami

# Push OTA to STAGING
$env:APP_VARIANT = "stage"
eas update --branch staging --message "feat: description of change"

# Push OTA to PRODUCTION  
$env:APP_VARIANT = "production"
eas update --branch preview --message "feat: description of change"

# List recent updates
eas update:list

# Rollback (re-publish an older update)
eas update:republish --branch preview --group <update-group-id>

# Full native build (when OTA is not enough)
eas build --profile preview --platform android

# Clear caches (when env switching issues occur)
Remove-Item -Recurse -Force .expo
Remove-Item -Recurse -Force node_modules\.cache
npx expo start --clear
```

---

## Project IDs Reference

| Project | Expo Project ID | EAS Update URL |
|---|---|---|
| **Production** (GasTechMobile) | `af65ddf8-bf52-4856-9eff-cd08773a7bab` | `https://u.expo.dev/af65ddf8-...` |
| **Staging** (GasTechMobileStage) | `2f94bcdc-e805-4cfb-a4d1-b9e15c833662` | `https://u.expo.dev/2f94bcdc-...` |

---

*This document covers the OTA update lifecycle for `gastech-mobile-v1` as of Expo SDK 54 / expo-updates 29.x. Review after any major SDK upgrade.*
