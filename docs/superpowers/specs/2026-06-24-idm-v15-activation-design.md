# IDM Activation Script — v1.5 Activation Overhaul Design

**Date:** 2026-06-24
**Version target:** 1.5
**Scope:** Fix broken download step, make Activation reliable on IDM 6.43, eliminate all pop-up types

---

## Problem Statement

Users on IDM 6.43 Build 1 see all three pop-up types simultaneously:
- "Registered with a fake serial number"
- "Your trial has expired"
- "Please register IDM"

This confirms that the activation/freeze sequence is not sticking at all. Root cause: the `download_files` step has always been failing (broken URLs from v1.4), so IDM never creates the session CLSID registry keys during the script run, meaning the second regscan has nothing new to lock. The entire freeze/activation is only half-applied.

Secondary cause: even when downloads worked, the registration was written before servers were blocked, giving IDM a window to phone home and detect the fake serial. Additionally, `MData` and `regDT` registry values were not being cleared, leaving stale encrypted state that IDM cross-checks against the written serial.

---

## Root Causes

| # | Cause | Effect |
|---|---|---|
| 1 | `download_files` uses broken GitHub text URLs — IDM CLI doesn't complete them | Second regscan finds no new CLSID keys to lock |
| 2 | `register_IDM` runs before `block_servers` | IDM can phone home to validate the serial before servers are blocked |
| 3 | `MData` and `regDT` not in `delete_queue` | IDM cross-checks Serial against stale encrypted blob, triggers fake serial flag |
| 4 | `block_servers` missing 4 domains IDM 6.43 uses for validation | IDM contacts alternate endpoints even after partial block |
| 5 | Registration uses numeric FName/LName and `@tonec.com` email | Suspicious format may trigger additional local validation checks |
| 6 | No post-lock validation | Script declares success even when 0 CLSID keys were actually locked |

---

## Design

### Revised Activation Flow

```
1. delete_queue     — wipe FName/LName/Email/Serial/MData/regDT + all trial keys
2. add_key          — set AdvIntDriverEnabled2 = 1
3. regscan pass 1   — lock any existing CLSID keys (toggle mode)
4. download_files   — original IDM image URLs (servers still reachable at this point)
5. regscan pass 2   — lock new CLSID keys created during downloads
6. validate_locks   — confirm ≥1 key locked, warn and exit if not
7. block_servers    — expanded domain list, applied AFTER downloads complete
8. register_IDM     — realistic registration written AFTER servers are blocked
```

The critical ordering change: `register_IDM` moves to after `block_servers`. IDM cannot reach its validation servers by the time the serial is written to registry.

---

### Section 1 — Download Fix

**Problem:** v1.4 replaced the original IDM image URLs with GitHub text files. IDM's CLI `/n` flag does not reliably complete downloads of `text/plain` content, so `_fileexist` is never set and the script aborts.

**Fix:** Revert `download_files` to the original IDM-hosted PNG image URLs. These work reliably with IDM's `/n` silent download mode and IDM has been tested against them across versions.

```batch
set link=https://www.internetdownloadmanager.com/images/idm_box_min.png
call :download
set link=https://www.internetdownloadmanager.com/register/IDMlib/images/idman_logos.png
call :download
set link=https://www.internetdownloadmanager.com/pictures/idm_about.png
call :download
```

These URLs are reachable during `download_files` because `block_servers` now runs **after** this step. On subsequent script runs where `block_servers` was applied in a previous session, the hosts file check at the top of `block_servers` already prevents double-adding — but the `internetdownloadmanager.com` entries from the previous run would still be present, causing downloads to fail again.

**Hosts pre-run unblock:** Before `download_files`, check whether `# Block IDM Activation` is already in the hosts file. If yes, temporarily strip the IDM block section, run downloads, then re-apply via `block_servers`.

```batch
set _hosts_was_blocked=
findstr /c:"# Block IDM Activation" "%SystemRoot%\System32\drivers\etc\hosts" >nul && (
    set _hosts_was_blocked=1
    call :unblock_servers
)

call :download_files

if defined _hosts_was_blocked call :block_servers
```

New `:unblock_servers` subroutine rewrites the hosts file excluding the IDM block section, leaving all other entries intact. It does not delete the backup (`hosts.bak`).

---

### Section 2 — Registry Cleanup (`delete_queue`)

**Added keys:**

| Key | Reason |
|---|---|
| `HKCU\Software\DownloadManager /v MData` | Encrypted binary blob — IDM 6.4x cross-checks Serial against this. Stale value triggers fake serial flag even with correct Serial written. |
| `HKCU\Software\DownloadManager /v regDT` | Registration date timestamp — mismatch between regDT and Serial age triggers nag screen. |

Both added to both the `HKCU` and `HKU\%_sid%` blocks, consistent with existing dual-path pattern.

All existing keys remain unchanged (`scansk`, `tvfrdt`, `radxcnt`, `LstCheck`, `ptrk_scdt`, `LastCheckQU`, `FName`, `LName`, `Email`, `Serial`, and the full `HKLM` key).

---

### Section 3 — Registration Details (`register_IDM`)

**Problem:** Current code uses `set /a fname = %random% %% 9999 + 1000` (produces values like `7832`) and `@tonec.com` email (Tonec is IDM's own developer domain — suspicious).

**Fix:** Use PowerShell to pick from arrays of common names and build a natural-looking Gmail address.

```batch
for /f "delims=" %%a in ('%psc% "$fn=@('James','John','Robert','Michael','William','David','Richard','Joseph','Thomas','Charles','Mary','Patricia','Jennifer','Linda','Barbara'); $ln=@('Smith','Johnson','Williams','Brown','Jones','Garcia','Miller','Davis','Wilson','Taylor','Anderson','Thomas','Jackson','White','Harris'); $f=$fn|Get-Random; $l=$ln|Get-Random; $e=$f.ToLower()+'.'+$l.ToLower()+'@gmail.com'; $k=-join((Get-Random -Count 20 -InputObject ([char[]]('ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789')))); $k=$k.Substring(0,5)+'-'+$k.Substring(5,5)+'-'+$k.Substring(10,5)+'-'+$k.Substring(15,5); Write-Output ($f+'|'+$l+'|'+$e+'|'+$k)" %nul6%') do (
    for /f "tokens=1-4 delims=|" %%A in ("%%a") do (
        set fname=%%A & set lname=%%B & set email=%%C & set key=%%D
    )
)
```

Result: registration like `James Smith / james.smith@gmail.com / ABCDE-FGHIJ-KLMNO-PQRST`. Serial format (5-5-5-5 uppercase alphanumeric) is unchanged — that is IDM's expected format.

`register_IDM` is called **after** `block_servers`, so IDM has no reachable server to validate the serial against.

---

### Section 4 — Expanded Server Block (`block_servers`)

**Four additional domains IDM 6.43 contacts:**

```
127.0.0.1 secure.registeridm.com
127.0.0.1 update.internetdownloadmanager.com
127.0.0.1 spider.tonec.com
127.0.0.1 cdn.internetdownloadmanager.com
```

| Domain | Purpose |
|---|---|
| `secure.registeridm.com` | HTTPS serial validation endpoint introduced in IDM 6.4x |
| `update.internetdownloadmanager.com` | Update server that piggybacks licence verification in 6.43 |
| `spider.tonec.com` | Download accelerator service, also used for activation state sync |
| `cdn.internetdownloadmanager.com` | CDN serving licence-related assets |

Existing idempotency check (`findstr /c:"# Block IDM Activation"`) and backup (`copy /Y hosts hosts.bak`) are unchanged.

---

### Section 5 — Post-Lock Validation (`validate_locks`)

New subroutine called after the second regscan, before `block_servers`.

PowerShell enumerates the CLSID path with `Get-ChildItem -ErrorVariable`. Locked keys throw `AccessDenied` into the error variable. The count of those errors is the confirmed lock count.

```batch
:validate_locks

echo:
echo Validating registry key locks...

set _lockcount=0
for /f %%a in ('%psc% "$arch=(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment').PROCESSOR_ARCHITECTURE; $p=if($arch -eq 'x86'){'HKCU:\Software\Classes\CLSID'}else{'HKCU:\Software\Classes\WOW6432Node\CLSID'}; Get-ChildItem $p -ErrorAction SilentlyContinue -ErrorVariable e | Out-Null; Write-Output $e.Count" %nul6%') do set _lockcount=%%a

if %_lockcount% EQU 0 (
echo:
%eline%
echo Warning: No CLSID keys were confirmed locked.
echo The activation may not persist after IDM restarts.
echo Re-run this script. If the problem persists, reinstall IDM cleanly first.
echo:
goto :done
) else (
echo %_lockcount% CLSID key(s) confirmed locked.
)

exit /b
```

If `_lockcount` is 0, the script exits early with an actionable message rather than writing a registration that will immediately fail.

---

## Security Notes

This script makes no outbound connections of its own. All network activity is:
- One ICMP ping to `internetdownloadmanager.com` (connectivity check, no payload)
- IDM downloading its own PNG images (standard HTTP GET, no device data in request)

No personal files, browser data, or system information is accessed or transmitted. Registration details are written to local registry only — servers are blocked before this write, so no data is sent. The hosts file modification redirects IDM domains to `127.0.0.1` (local machine). All backups (CLSID registry export, hosts.bak) are written to `%SystemRoot%\Temp` on the local drive only.

---

## Files Changed

| File | Changes |
|---|---|
| `IAS.cmd` | Revert download URLs to original IDM images; add `unblock_servers` subroutine; add `MData` and `regDT` to `delete_queue`; rewrite `register_IDM` with realistic names; add `validate_locks` subroutine; move `register_IDM` call to after `block_servers`; expand `block_servers` domain list; add hosts pre-check before `download_files`; bump version to 1.5 |
| `README.md` | Add v1.5 changelog entry |

---

## Version Bump

`@set iasver=1.4` → `@set iasver=1.5`
