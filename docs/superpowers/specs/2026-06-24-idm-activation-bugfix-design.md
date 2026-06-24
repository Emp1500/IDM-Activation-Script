# IDM Activation Script — v1.4 Bug Fix Design

**Date:** 2026-06-24  
**Version target:** 1.4  
**Scope:** Bug fixes only — no new features

---

## Problem Statement

Users who run the Freeze Trial option still receive random IDM activation pop-ups during normal use. The root cause is that the freeze trial process does not complete correctly due to a sequencing bug and blocked download URLs.

---

## Bugs Fixed

### Bug 1 (Critical) — `block_servers` called before `download_files`

**File:** `IAS.cmd` — `:_activate` section

**Current order:**
1. `delete_queue`
2. `add_key`
3. `block_servers` ← blocks `internetdownloadmanager.com` in hosts file
4. First `regscan` (locks existing CLSID keys)
5. `download_files` ← fails: IDM can't reach `internetdownloadmanager.com` (now blocked)
6. Second `regscan` ← never runs / runs on no new keys

**Effect:** The second regscan, which is responsible for locking the trial-related registry keys IDM creates *during* downloads, never runs successfully. The freeze is only partially applied, leaving IDM able to show activation pop-ups.

**Fix:** Move `call :block_servers` to after the second `regscan`:

```
1. delete_queue
2. add_key
3. First regscan (lockKey=1, toggle=1)
4. [register_IDM if frz=0]
5. download_files
6. Second regscan (lockKey=1)
7. block_servers  ← moved here
```

---

### Bug 2 (Critical) — Download URLs point to blocked servers

**File:** `IAS.cmd` — `:download_files` section

**Current URLs:**
```
https://www.internetdownloadmanager.com/images/idm_box_min.png
https://www.internetdownloadmanager.com/register/IDMlib/images/idman_logos.png
https://www.internetdownloadmanager.com/pictures/idm_about.png
```

**Effect:** Even if Bug 1 is fixed, on *subsequent runs* the hosts file already blocks `internetdownloadmanager.com`, so downloads fail again. The actual download destination doesn't matter — IDM creates its registry keys when downloading *any* file. These URLs just need to be reachable and stable.

**Fix:** Replace with neutral URLs from `raw.githubusercontent.com` (already trusted by `IAS.ps1`):

```
https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.cmd
https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.ps1
https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/README.md
```

The downloaded temp file path and cleanup logic stay unchanged.

---

### Bug 3 (Medium) — `$DownloadURL2` undefined in `IAS.ps1`

**File:** `IAS.ps1`

**Current code:**
```powershell
$DownloadURL = 'https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.cmd'

try {
    $response = Invoke-WebRequest -Uri $DownloadURL -UseBasicParsing
}
catch {
    $response = Invoke-WebRequest -Uri $DownloadURL2 -UseBasicParsing  # $DownloadURL2 never defined
}
```

**Effect:** If the primary URL fails, the catch block also throws (undefined variable), giving the user a confusing error with no fallback.

**Fix:** Define `$DownloadURL2` as a jsDelivr CDN mirror of the same file:

```powershell
$DownloadURL = 'https://raw.githubusercontent.com/lstprjct/IDM-Activation-Script/main/IAS.cmd'
$DownloadURL2 = 'https://cdn.jsdelivr.net/gh/lstprjct/IDM-Activation-Script@main/IAS.cmd'
```

---

### Bug 4 (Minor) — Version string not updated in `IAS.cmd`

**File:** `IAS.cmd` — line 1

**Current:** `@set iasver=1.2`  
**Fix:** `@set iasver=1.4`

---

## Safety Notes

All changes are to plain-text batch/PowerShell files. No new system access, no new network destinations beyond GitHub and jsDelivr (both already implicitly trusted). The script continues to:
- Back up the hosts file before modifying it
- Back up CLSID registry keys before touching them
- Clean up all temp files after use
- Request admin elevation explicitly

---

## Files Changed

| File | Changes |
|------|---------|
| `IAS.cmd` | Reorder `block_servers` call; replace download URLs; bump version to 1.4 |
| `IAS.ps1` | Define `$DownloadURL2` fallback |
| `README.md` | Add v1.4 changelog entry |
