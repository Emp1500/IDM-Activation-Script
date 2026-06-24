# IDM Activation Script v1.4 Bug Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix four bugs in IAS.cmd and IAS.ps1 that cause IDM activation pop-ups to appear during normal use after running Freeze Trial.

**Architecture:** Pure script edits — no new files, no new dependencies. Changes are isolated to three files: `IAS.cmd` (version bump, URL swap, reorder one call), `IAS.ps1` (add one variable), `README.md` (changelog entry).

**Tech Stack:** Windows Batch Script, PowerShell 5+

## Global Constraints

- Windows 7/8/8.1/10/11 and Server equivalents must remain supported
- No new network destinations beyond `raw.githubusercontent.com` and `cdn.jsdelivr.net`
- No new registry keys, no new hosts entries beyond what already exists
- All changes must be readable plain text — no binaries, no obfuscation
- Script must continue to require explicit admin elevation

---

### Task 1: Bump version string and fix IAS.ps1 fallback URL

**Files:**
- Modify: `IAS.cmd` line 1
- Modify: `IAS.ps1` line 7 (after `$DownloadURL` definition)

**Interfaces:**
- Produces: `iasver=1.4` visible in CMD window title; `$DownloadURL2` available in catch block

- [ ] **Step 1: Verify current state of version string**

Run:
```bash
grep -n "iasver" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd" | head -3
```
Expected output contains: `1:@set iasver=1.2`

- [ ] **Step 2: Update version string in IAS.cmd**

In `IAS.cmd` line 1, change:
```batch
@set iasver=1.2
```
To:
```batch
@set iasver=1.4
```

- [ ] **Step 3: Verify current state of IAS.ps1**

Run:
```bash
grep -n "DownloadURL" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.ps1"
```
Expected output: only one line containing `$DownloadURL = '...'`, no `$DownloadURL2`

- [ ] **Step 4: Add $DownloadURL2 to IAS.ps1**

In `IAS.ps1`, after the line:
```powershell
$DownloadURL = 'https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.cmd'
```
Add immediately below it:
```powershell
$DownloadURL2 = 'https://cdn.jsdelivr.net/gh/lstprjct/IDM-Activation-Script@main/IAS.cmd'
```

So that block now reads:
```powershell
$DownloadURL = 'https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.cmd'
$DownloadURL2 = 'https://cdn.jsdelivr.net/gh/lstprjct/IDM-Activation-Script@main/IAS.cmd'
```

- [ ] **Step 5: Verify both changes**

Run:
```bash
grep -n "iasver\|DownloadURL" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd" | head -2
grep -n "DownloadURL" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.ps1"
```
Expected:
```
1:@set iasver=1.4
...
7:$DownloadURL = 'https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.cmd'
8:$DownloadURL2 = 'https://cdn.jsdelivr.net/gh/lstprjct/IDM-Activation-Script@main/IAS.cmd'
```

- [ ] **Step 6: Commit**

```bash
git add IAS.cmd IAS.ps1
git commit -m "fix: bump version to 1.4 and add IAS.ps1 fallback URL"
```

---

### Task 2: Replace blocked download URLs in IAS.cmd

**Files:**
- Modify: `IAS.cmd` — `:download_files` section (approx lines 660–668)

**Interfaces:**
- Consumes: `%IDMan%` (IDM executable path, resolved earlier in script)
- Produces: `_fileexist` flag set to `1` if any download succeeds; `%SystemRoot%\Temp\temp.png` temp file

**Context:** The `:download` function saves every URL as `temp.png` via IDM's `/f temp.png` flag regardless of the URL's actual file type. The URL content doesn't matter — IDM creates registry keys on any download. The three GitHub raw URLs are reachable, stable, and won't appear in any IDM server block list.

- [ ] **Step 1: Verify current download URLs**

Run:
```bash
grep -n "internetdownloadmanager.com" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected — three lines like:
```
663:set link=https://www.internetdownloadmanager.com/images/idm_box_min.png
665:set link=https://www.internetdownloadmanager.com/register/IDMlib/images/idman_logos.png
667:set link=https://www.internetdownloadmanager.com/pictures/idm_about.png
```

- [ ] **Step 2: Replace the three download URLs**

In `IAS.cmd`, in the `:download_files` section, replace:
```batch
set link=https://www.internetdownloadmanager.com/images/idm_box_min.png
call :download
set link=https://www.internetdownloadmanager.com/register/IDMlib/images/idman_logos.png
call :download
set link=https://www.internetdownloadmanager.com/pictures/idm_about.png
call :download
```
With:
```batch
set link=https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.cmd
call :download
set link=https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.ps1
call :download
set link=https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/README.md
call :download
```

- [ ] **Step 3: Verify no remaining IDM download URLs**

Run:
```bash
grep -n "internetdownloadmanager.com" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: output contains only the internet connectivity check lines (in `:_activate`, around line 527–535), NOT the download URLs. The download section should now reference `raw.githubusercontent.com`.

- [ ] **Step 4: Verify new URLs are in place**

Run:
```bash
grep -n "raw.githubusercontent.com" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: exactly 3 lines, all inside the `:download_files` section.

- [ ] **Step 5: Commit**

```bash
git add IAS.cmd
git commit -m "fix: replace blocked IDM download URLs with neutral GitHub raw URLs"
```

---

### Task 3: Move block_servers call to after download_files (critical fix)

**Files:**
- Modify: `IAS.cmd` — `:_activate` section (approx lines 556–588)

**Context:** `block_servers` adds `internetdownloadmanager.com` (and others) to the Windows hosts file. The internet connectivity check at the top of `:_activate` pings `internetdownloadmanager.com` — this check runs BEFORE `block_servers`, so it is unaffected by this reorder. Only `download_files` is affected, and it now uses GitHub URLs (Task 2), so moving `block_servers` here ensures server blocking happens after all registry key locking is complete.

- [ ] **Step 1: Verify current position of block_servers call**

Run:
```bash
grep -n "block_servers\|download_files\|regscan" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected — `block_servers` appears BEFORE `download_files`:
```
559:call :block_servers
561:%psc% "...regscan..." (first regscan)
563:if %frz%==0 call :register_IDM
565:call :download_files
574:%psc% "...regscan..." (second regscan)
```

- [ ] **Step 2: Remove block_servers from its current position**

In `IAS.cmd` `:_activate` section, remove the line:
```batch
call :block_servers
```
(and its surrounding blank line) from between `call :add_key` and the first `%psc%` regscan call.

- [ ] **Step 3: Add block_servers after the second regscan**

In `IAS.cmd` `:_activate` section, after the second regscan `%psc%` line and before the success message `echo:` / `echo %line%`, add:
```batch
call :block_servers
```

The full `:_activate` section from `call :delete_queue` through the success message should now read:

```batch
call :delete_queue
call :add_key

%psc% "$sid = '%_sid%'; $HKCUsync = %HKCUsync%; $lockKey = 1; $deleteKey = $null; $toggle = 1; $f=[io.file]::ReadAllText('!_batp!') -split ':regscan\:.*';iex ($f[1])"

if %frz%==0 call :register_IDM

call :download_files
if not defined _fileexist (
%eline%
echo Error: Unable to download files with IDM.
echo:
echo Help: %mas%IAS-Help#troubleshoot
goto :done
)

%psc% "$sid = '%_sid%'; $HKCUsync = %HKCUsync%; $lockKey = 1; $deleteKey = $null; $f=[io.file]::ReadAllText('!_batp!') -split ':regscan\:.*';iex ($f[1])"

call :block_servers

echo:
echo %line%
echo:
if %frz%==0 (
call :_color %Green% "The IDM Activation process has been completed."
echo:
call :_color %Gray% "If the fake serial screen appears, use the Freeze Trial option instead."
) else (
call :_color %Green% "The IDM 30 days trial period is successfully freezed for Lifetime."
echo:
call :_color %Gray% "If IDM is showing a popup to register, reinstall IDM."
)
```

- [ ] **Step 4: Verify new order**

Run:
```bash
grep -n "block_servers\|download_files\|regscan" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected — `block_servers` now appears AFTER both regscan calls and AFTER `download_files`:
```
(first regscan line)
call :download_files
(second regscan line)
call :block_servers
```

- [ ] **Step 5: Commit**

```bash
git add IAS.cmd
git commit -m "fix: move block_servers to after download_files so registry locking completes first"
```

---

### Task 4: Update README changelog

**Files:**
- Modify: `README.md` — Changelog section

- [ ] **Step 1: Add v1.4 changelog entry**

In `README.md`, in the `# Changelog` section, insert a new `## v1.4` block immediately above the existing `## v1.3` block:

```markdown
## v1.4
* Fixed critical bug: `block_servers` was called before `download_files`, preventing the second registry key lock pass from running. Freeze Trial now fully locks all trial keys, eliminating random activation pop-ups.
* Replaced IDM-specific download URLs in the freeze/activation flow with neutral GitHub raw URLs. The previous URLs pointed to `internetdownloadmanager.com`, which the script itself blocks — causing download failures on all subsequent runs.
* Fixed undefined `$DownloadURL2` variable in `IAS.ps1`. If the primary download URL failed, the catch block also threw an error instead of falling back to the jsDelivr CDN mirror.
```

- [ ] **Step 2: Verify changelog section**

Run:
```bash
grep -n "## v1\." "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/README.md"
```
Expected:
```
## v1.4
## v1.3
## v1.2
...
```
(v1.4 appears first)

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add v1.4 changelog"
```
