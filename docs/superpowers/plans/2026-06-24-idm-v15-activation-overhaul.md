# IDM Activation Script v1.5 — Activation Overhaul Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the broken activation/freeze flow on IDM 6.43 by restoring working downloads, expanding registry cleanup, blocking all validation endpoints after downloads, writing realistic registration only after servers are blocked, and confirming CLSID locks actually applied.

**Architecture:** Pure script edits to `IAS.cmd` and `README.md` only. No new files, no new dependencies. Seven isolated tasks committed separately.

**Tech Stack:** Windows Batch Script, PowerShell 5+

## Global Constraints

- Windows 7/8/8.1/10/11 and Server equivalents must remain supported
- No new network destinations beyond what IDM already contacts
- No new registry keys beyond IDM's own hive
- No startup tasks, scheduled tasks, or firewall rules added
- All changes are plain-text batch/PowerShell — no binaries
- Script must continue to require explicit admin elevation
- No data leaves the device; hosts modifications redirect to 127.0.0.1 only

---

### Task 1: Bump version to 1.5 and add README changelog

**Files:**
- Modify: `IAS.cmd` line 1
- Modify: `README.md` — Changelog section

**Interfaces:**
- Produces: `iasver=1.5` shown in CMD window title bar

- [ ] **Step 1: Verify current version string**

```bash
grep -n "iasver" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd" | head -2
```
Expected: `1:@set iasver=1.4`

- [ ] **Step 2: Update version string in IAS.cmd line 1**

Change:
```batch
@set iasver=1.4
```
To:
```batch
@set iasver=1.5
```

- [ ] **Step 3: Add v1.5 changelog entry to README.md**

In `README.md`, immediately above the existing `## v1.4` block in the `# Changelog` section, insert:

```markdown
## v1.5
* Fixed download step: reverted to original IDM image URLs; added `unblock_servers` to temporarily lift hosts blocking before each download run so downloads succeed on repeated script runs.
* Expanded `delete_queue` to also wipe `MData` and `regDT` registry values that IDM 6.43 uses to cross-check serial validity — without clearing these, IDM flags the serial as fake even when a correct value is written.
* Rewrote `register_IDM` to generate a realistic first/last name pair and a gmail.com address instead of random numbers and a suspicious `@tonec.com` email.
* Moved `register_IDM` call to after `block_servers` — IDM has no reachable server to phone home to when the serial is written to registry.
* Expanded `block_servers` with four additional domains IDM 6.43 uses for serial validation: `secure.registeridm.com`, `update.internetdownloadmanager.com`, `spider.tonec.com`, `cdn.internetdownloadmanager.com`.
* Added `validate_locks` subroutine that confirms at least one CLSID key was locked before proceeding — exits with an actionable warning if 0 keys were locked.
```

- [ ] **Step 4: Verify changelog position**

```bash
grep -n "## v1\." "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/README.md"
```
Expected: `## v1.5` line number is lower (appears earlier) than `## v1.4`

- [ ] **Step 5: Commit**

```bash
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" add IAS.cmd README.md
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" commit -m "chore: bump version to 1.5 and add changelog"
```

---

### Task 2: Expand delete_queue with MData and regDT

**Files:**
- Modify: `IAS.cmd` lines 444–473 (`:delete_queue` section)

**Interfaces:**
- Produces: `MData` and `regDT` deleted from both `HKCU\Software\DownloadManager` and `HKU\%_sid%\Software\DownloadManager` before any registration is written

- [ ] **Step 1: Verify MData and regDT are not yet present**

```bash
grep -n "MData\|regDT" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: no output

- [ ] **Step 2: Add MData and regDT to the HKCU block (lines 444–458)**

Find the line:
```batch
""HKCU\Software\DownloadManager" "/v" "Serial""
```
(currently line 448). Insert two lines immediately after it:
```batch
""HKCU\Software\DownloadManager" "/v" "MData""
""HKCU\Software\DownloadManager" "/v" "regDT""
```

The HKCU block should now read:
```batch
for %%# in (
""HKCU\Software\DownloadManager" "/v" "FName""
""HKCU\Software\DownloadManager" "/v" "LName""
""HKCU\Software\DownloadManager" "/v" "Email""
""HKCU\Software\DownloadManager" "/v" "Serial""
""HKCU\Software\DownloadManager" "/v" "MData""
""HKCU\Software\DownloadManager" "/v" "regDT""
""HKCU\Software\DownloadManager" "/v" "scansk""
""HKCU\Software\DownloadManager" "/v" "tvfrdt""
""HKCU\Software\DownloadManager" "/v" "radxcnt""
""HKCU\Software\DownloadManager" "/v" "LstCheck""
""HKCU\Software\DownloadManager" "/v" "ptrk_scdt""
""HKCU\Software\DownloadManager" "/v" "LastCheckQU""
"%HKLM%"
) do for /f "tokens=* delims=" %%A in ("%%~#") do (
```

- [ ] **Step 3: Add MData and regDT to the HKU block (lines 460–473)**

Find the line:
```batch
""HKU\%_sid%\Software\DownloadManager" "/v" "Serial""
```
(currently line 464). Insert two lines immediately after it:
```batch
""HKU\%_sid%\Software\DownloadManager" "/v" "MData""
""HKU\%_sid%\Software\DownloadManager" "/v" "regDT""
```

The HKU block should now read:
```batch
if not %HKCUsync%==1 for %%# in (
""HKU\%_sid%\Software\DownloadManager" "/v" "FName""
""HKU\%_sid%\Software\DownloadManager" "/v" "LName""
""HKU\%_sid%\Software\DownloadManager" "/v" "Email""
""HKU\%_sid%\Software\DownloadManager" "/v" "Serial""
""HKU\%_sid%\Software\DownloadManager" "/v" "MData""
""HKU\%_sid%\Software\DownloadManager" "/v" "regDT""
""HKU\%_sid%\Software\DownloadManager" "/v" "scansk""
""HKU\%_sid%\Software\DownloadManager" "/v" "tvfrdt""
""HKU\%_sid%\Software\DownloadManager" "/v" "radxcnt""
""HKU\%_sid%\Software\DownloadManager" "/v" "LstCheck""
""HKU\%_sid%\Software\DownloadManager" "/v" "ptrk_scdt""
""HKU\%_sid%\Software\DownloadManager" "/v" "LastCheckQU""
) do for /f "tokens=* delims=" %%A in ("%%~#") do (
```

- [ ] **Step 4: Verify both additions**

```bash
grep -n "MData\|regDT" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: exactly 4 lines — HKCU/MData, HKCU/regDT, HKU/MData, HKU/regDT

- [ ] **Step 5: Commit**

```bash
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" add IAS.cmd
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" commit -m "fix: add MData and regDT to delete_queue for IDM 6.43"
```

---

### Task 3: Revert download URLs to original IDM images

**Files:**
- Modify: `IAS.cmd` lines 662–667 (`:download_files` section)

**Interfaces:**
- Consumes: `%IDMan%` (IDM executable path), `%SystemRoot%\Temp` (save directory)
- Produces: `_fileexist=1` when any of the 3 downloads completes within 20 seconds per attempt

- [ ] **Step 1: Verify current broken URLs**

```bash
grep -n "set link=" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: 3 lines pointing to `raw.githubusercontent.com`

- [ ] **Step 2: Replace all 3 URLs**

Replace the 3 URL lines (662–667):
```batch
set link=https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.cmd
call :download
set link=https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.ps1
call :download
set link=https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/README.md
call :download
```
With:
```batch
set link=https://www.internetdownloadmanager.com/images/idm_box_min.png
call :download
set link=https://www.internetdownloadmanager.com/register/IDMlib/images/idman_logos.png
call :download
set link=https://www.internetdownloadmanager.com/pictures/idm_about.png
call :download
```

- [ ] **Step 3: Verify new URLs**

```bash
grep -n "set link=" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: 3 lines pointing to `www.internetdownloadmanager.com`

```bash
grep -n "raw.githubusercontent.com" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: no output (no remaining raw GitHub URLs in script body)

- [ ] **Step 4: Commit**

```bash
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" add IAS.cmd
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" commit -m "fix: revert download_files to original IDM image URLs"
```

---

### Task 4: Add :validate_locks subroutine and call it in :_activate

**Files:**
- Modify: `IAS.cmd` — insert `:validate_locks` subroutine before line 712 (`::===...`); add `call :validate_locks` at line 573 (after second regscan)

**Interfaces:**
- Consumes: `HKCU:\Software\Classes\WOW6432Node\CLSID` (or `CLSID` on x86) — queries for access-denied keys
- Produces: `_lockcount` (integer count of confirmed locked CLSID keys); exits to `:done` with error if `_lockcount` is 0

- [ ] **Step 1: Verify validate_locks not yet present**

```bash
grep -n "validate_locks\|_lockcount" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: no output

- [ ] **Step 2: Add the subroutine before :block_servers**

In `IAS.cmd`, find the separator line immediately before `:block_servers` (line 712):
```batch
::========================================================================================================================================

:block_servers
```
Insert the new subroutine before that separator block:

```batch
:validate_locks

echo:
echo Validating registry key locks...

set _lockcount=0
set "_vtemp=%SystemRoot%\Temp\_ias_vlc"
%psc% "$p=if($env:PROCESSOR_ARCHITECTURE -eq 'x86'){'HKCU:\Software\Classes\CLSID'}else{'HKCU:\Software\Classes\WOW6432Node\CLSID'}; $e=@(); Get-ChildItem $p -ErrorAction SilentlyContinue -ErrorVariable +e | Out-Null; [io.file]::WriteAllText('!_vtemp!',($e.Count).ToString())" %nul2%

if exist "!_vtemp!" (
set /p _lockcount=<"!_vtemp!"
del /f /q "!_vtemp!"
)

if %_lockcount% EQU 0 (
echo:
%eline%
echo Warning: No CLSID keys were confirmed locked.
echo The activation may not persist after IDM restarts.
echo Re-run this script. If the problem persists, reinstall IDM cleanly first.
echo:
goto done
)

echo !_lockcount! CLSID key(s) confirmed locked.

exit /b

::========================================================================================================================================

:block_servers
```

- [ ] **Step 3: Add the call in :_activate after the second regscan**

Find the second regscan line in `:_activate` (line 572) followed by `call :block_servers` (line 574):
```batch
%psc% "$sid = '%_sid%'; $HKCUsync = %HKCUsync%; $lockKey = 1; $deleteKey = $null; $f=[io.file]::ReadAllText('!_batp!') -split ':regscan\:.*';iex ($f[1])"

call :block_servers
```
Replace with:
```batch
%psc% "$sid = '%_sid%'; $HKCUsync = %HKCUsync%; $lockKey = 1; $deleteKey = $null; $f=[io.file]::ReadAllText('!_batp!') -split ':regscan\:.*';iex ($f[1])"

call :validate_locks

call :block_servers
```

- [ ] **Step 4: Verify subroutine and call are in place**

```bash
grep -n "validate_locks\|_lockcount\|_vtemp" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: `:validate_locks` label, `call :validate_locks`, `_lockcount`, `_vtemp` — all present

- [ ] **Step 5: Commit**

```bash
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" add IAS.cmd
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" commit -m "fix: add validate_locks subroutine to confirm CLSID keys are locked"
```

---

### Task 5: Add :unblock_servers subroutine and hosts pre-check before downloads

**Files:**
- Modify: `IAS.cmd` — insert `:unblock_servers` subroutine before `:validate_locks`; add pre-check block before `call :download_files` (line 563)

**Interfaces:**
- Consumes: `%SystemRoot%\System32\drivers\etc\hosts`, `# Block IDM Activation` marker string
- Produces: hosts file with IDM block section removed (truncated at the marker); `_hosts_was_blocked` flag (informational, not used in conditional logic — block_servers re-adds entries unconditionally after downloads)

- [ ] **Step 1: Verify unblock_servers not yet present**

```bash
grep -n "unblock_servers\|_hosts_was_blocked" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: no output

- [ ] **Step 2: Add :unblock_servers subroutine before :validate_locks**

Find the `:validate_locks` label (added in Task 4). Insert the new subroutine immediately before it:

```batch
:unblock_servers
    echo.
    echo Temporarily removing IDM server block for downloads...
    set "hosts_file=%SystemRoot%\System32\drivers\etc\hosts"
    %psc% "$f='%hosts_file%'; $c=[io.file]::ReadAllText($f); $i=$c.IndexOf('# Block IDM Activation'); if($i -ge 0){[io.file]::WriteAllText($f,$c.Substring(0,$i).TrimEnd()+[char]13+[char]10)}" %nul%
    echo IDM server block temporarily removed.
goto :eof

```

The PowerShell reads the entire hosts file, finds the index of `# Block IDM Activation`, and truncates the file to just before that marker (trimming trailing whitespace, then adding a single CRLF). All entries above the marker are preserved. Does not touch `hosts.bak`.

- [ ] **Step 3: Add hosts pre-check before call :download_files in :_activate**

Find the current line 563 in `:_activate`:
```batch
call :download_files
if not defined _fileexist (
```
Replace it with:
```batch
set _hosts_was_blocked=
findstr /c:"# Block IDM Activation" "%SystemRoot%\System32\drivers\etc\hosts" >nul && (
    set _hosts_was_blocked=1
    call :unblock_servers
)

call :download_files
if not defined _fileexist (
```

Note: `block_servers` (called after downloads) re-adds all entries unconditionally on every run where the marker is absent. Since `unblock_servers` removes the marker, `block_servers` will always re-add the full block after downloads complete. The `_hosts_was_blocked` flag is available for diagnostic purposes only.

- [ ] **Step 4: Verify pre-check and subroutine are in place**

```bash
grep -n "unblock_servers\|_hosts_was_blocked" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: minimum 4 lines — `findstr` check, `_hosts_was_blocked=1`, `call :unblock_servers`, `:unblock_servers` label

- [ ] **Step 5: Commit**

```bash
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" add IAS.cmd
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" commit -m "fix: add unblock_servers subroutine and hosts pre-check before downloads"
```

---

### Task 6: Expand block_servers with 4 additional IDM 6.43 domains

**Files:**
- Modify: `IAS.cmd` line 745 (`:block_servers` echo block, after `star.tonec.com`)

**Interfaces:**
- Produces: 14 total `127.0.0.1` entries in the hosts file block (10 existing + 4 new)

- [ ] **Step 1: Verify current last entry in the echo block**

```bash
grep -n "127.0.0.1" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: 10 entries, last one being `echo 127.0.0.1 star.tonec.com`

- [ ] **Step 2: Add 4 new domains after star.tonec.com**

Find line 745:
```batch
        echo 127.0.0.1 star.tonec.com
    ) >> "%hosts_file%"
```
Replace with:
```batch
        echo 127.0.0.1 star.tonec.com
        echo 127.0.0.1 secure.registeridm.com
        echo 127.0.0.1 update.internetdownloadmanager.com
        echo 127.0.0.1 spider.tonec.com
        echo 127.0.0.1 cdn.internetdownloadmanager.com
    ) >> "%hosts_file%"
```

- [ ] **Step 3: Verify 14 total entries**

```bash
grep -n "127.0.0.1" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: 14 lines, last 4 being the new additions ending with `cdn.internetdownloadmanager.com`

- [ ] **Step 4: Commit**

```bash
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" add IAS.cmd
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" commit -m "fix: expand block_servers with 4 additional IDM 6.43 validation domains"
```

---

### Task 7: Reorder :_activate and rewrite register_IDM with realistic registration

**Files:**
- Modify: `IAS.cmd` lines 559–574 (`:_activate` flow — move `register_IDM` call); lines 628–651 (`:register_IDM` subroutine — full rewrite)

**Interfaces:**
- Consumes: `%frz%` (0 = activation, 1 = freeze trial), `%_sid%`, `%HKCUsync%`
- Produces: `HKCU\Software\DownloadManager` written with realistic FName (e.g. `James`), LName (e.g. `Smith`), Email (`james.smith@gmail.com`), Serial (`ABCDE-FGHIJ-KLMNO-PQRST`) — written only when `frz=0`, only after servers are blocked

- [ ] **Step 1: Verify current position of register_IDM call**

```bash
grep -n "register_IDM\|download_files\|validate_locks\|block_servers" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: `register_IDM` call line number is LOWER than `download_files` call line number (i.e., currently before downloads)

- [ ] **Step 2: Remove register_IDM from its current position**

In `:_activate`, remove the line (currently line 561):
```batch
if %frz%==0 call :register_IDM
```
And the blank line after it. The flow between the first regscan and `call :download_files` becomes:
```batch
%psc% "$sid = '%_sid%'; $HKCUsync = %HKCUsync%; $lockKey = 1; $deleteKey = $null; $toggle = 1; $f=[io.file]::ReadAllText('!_batp!') -split ':regscan\:.*';iex ($f[1])"

call :download_files
```

- [ ] **Step 3: Add register_IDM call after block_servers**

Find (after Task 4 and Task 5 edits) the block in `:_activate`:
```batch
call :block_servers

echo:
echo %line%
```
Replace with:
```batch
call :block_servers

if %frz%==0 call :register_IDM

echo:
echo %line%
```

- [ ] **Step 4: Verify new call position**

```bash
grep -n "register_IDM\|block_servers\|validate_locks" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: `call :validate_locks` < `call :block_servers` < `call :register_IDM` (ascending line numbers)

- [ ] **Step 5: Rewrite :register_IDM subroutine**

Replace the entire `:register_IDM` subroutine (lines 628–651) with:

```batch
:register_IDM

echo:
echo Applying registration details...
echo:

set fname=
set lname=
set email=
set key=

for /f "delims=" %%a in ('%psc% "$fn=@('James','John','Robert','Michael','William','David','Richard','Joseph','Thomas','Charles','Mary','Patricia','Jennifer','Linda','Barbara'); $ln=@('Smith','Johnson','Williams','Brown','Jones','Garcia','Miller','Davis','Wilson','Taylor','Anderson','Thomas','Jackson','White','Harris'); $f=$fn|Get-Random; $l=$ln|Get-Random; $e=$f.ToLower()+'.'+$l.ToLower()+'@gmail.com'; $k=-join((Get-Random -Count 20 -InputObject ([char[]]('ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789')))); $k=$k.Substring(0,5)+'-'+$k.Substring(5,5)+'-'+$k.Substring(10,5)+'-'+$k.Substring(15,5); Write-Output ($f+'|'+$l+'|'+$e+'|'+$k)" %nul6%') do (
for /f "tokens=1-4 delims=|" %%A in ("%%a") do (
set "fname=%%A"
set "lname=%%B"
set "email=%%C"
set "key=%%D"
)
)

set "reg=HKCU\SOFTWARE\DownloadManager /v FName /t REG_SZ /d "%fname%"" & call :_rcont
set "reg=HKCU\SOFTWARE\DownloadManager /v LName /t REG_SZ /d "%lname%"" & call :_rcont
set "reg=HKCU\SOFTWARE\DownloadManager /v Email /t REG_SZ /d "%email%"" & call :_rcont
set "reg=HKCU\SOFTWARE\DownloadManager /v Serial /t REG_SZ /d "%key%"" & call :_rcont

if not %HKCUsync%==1 (
set "reg=HKU\%_sid%\SOFTWARE\DownloadManager /v FName /t REG_SZ /d "%fname%"" & call :_rcont
set "reg=HKU\%_sid%\SOFTWARE\DownloadManager /v LName /t REG_SZ /d "%lname%"" & call :_rcont
set "reg=HKU\%_sid%\SOFTWARE\DownloadManager /v Email /t REG_SZ /d "%email%"" & call :_rcont
set "reg=HKU\%_sid%\SOFTWARE\DownloadManager /v Serial /t REG_SZ /d "%key%"" & call :_rcont
)
exit /b
```

The PowerShell generates all four values in one call and outputs them pipe-delimited (`James|Smith|james.smith@gmail.com|ABCDE-FGHIJ-KLMNO-PQRST`). The inner `for /f` with `tokens=1-4 delims=|` splits them into `%%A`–`%%D`. The serial format (`XXXXX-XXXXX-XXXXX-XXXXX`, uppercase alphanumeric) is unchanged from the original — that is IDM's expected serial format.

- [ ] **Step 6: Verify no @tonec.com remains in register_IDM**

```bash
grep -n "tonec.com\|@gmail\|Get-Random" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd"
```
Expected: `@tonec.com` appears only inside `:block_servers` (the domain to block), NOT in `:register_IDM`; `@gmail.com` appears inside `:register_IDM`

- [ ] **Step 7: Verify complete :_activate flow order**

```bash
grep -n "delete_queue\|add_key\|regscan\|download_files\|validate_locks\|block_servers\|register_IDM" "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script/IAS.cmd" | grep -v "^[0-9]*::.*register_IDM\|:register_IDM\b\|:validate_locks\b\|:block_servers\b\|:delete_queue\b\|:add_key\b\|:download_files\b"
```

Verify ascending line number order:
1. `call :delete_queue`
2. `call :add_key`
3. first regscan `%psc%` (contains `$toggle = 1`)
4. `call :download_files`
5. second regscan `%psc%` (no `$toggle`)
6. `call :validate_locks`
7. `call :block_servers`
8. `if %frz%==0 call :register_IDM`

- [ ] **Step 8: Commit**

```bash
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" add IAS.cmd
git -C "/mnt/c/Users/KING_/Documents/WebDev/IDM_Final/IDM-Activation-Script" commit -m "fix: move register_IDM after block_servers and rewrite with realistic registration"
```
