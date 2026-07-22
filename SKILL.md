---
name: disk-audit
description: Scan and analyze disk space usage on Windows and macOS (with Linux references), then produce a categorized cleanup report — and, only when the user explicitly asks, execute cleanup with per-item confirmation. Use this skill whenever the user asks to "scan C drive", "analyze disk space", "find what's eating my storage", "clean up system drive", "磁盘空间分析", "清理 C 盘", "扫描磁盘占用", "找重复文件", "find duplicates", or wants a breakdown of where disk space went. Also trigger when user mentions disk full, low space warnings, or wants to know what can be safely deleted. The audit phase is always read-only. The cleanup phase requires explicit, per-item user confirmation and never proceeds on ambiguous intent.
---

# Disk Space Audit

A repeatable workflow for auditing disk space on Windows and macOS (with Linux references). The output is a categorized report that tells the user exactly where their bytes went and what's safe to clean. **Cleanup is a separate, explicitly-confirmed phase** — the audit itself never touches anything.

## Core principles

1. **Audit is read-only.** The scan-and-report phase never deletes, moves, or modifies files. Cleanup only begins when the user explicitly asks, and even then every item requires individual confirmation.
2. **Ambiguous intent → ask, don't assume.** If the user says "clean some stuff" without specifying what, or "delete the caches" without saying which, you MUST ask a focused clarifying question before acting. Never round up ambiguous approval into a broad deletion. When in doubt, ask one more question — not one fewer.
3. **Surface potential issues mid-cleanup.** If during a confirmed cleanup you notice something that looks wrong (a path resolves to more than expected, a cache is actively being written to, a folder belongs to a still-running app, a symlink points somewhere unexpected), STOP and ask before continuing — even if the user already said "yes" to that item. Confirmation is not a blank check; new information resets it.
4. **Value over vanity.** The goal is NOT to maximize "GB cleaned". Clearing 50GB of useful cache to show a big number is harmful. A cache that saves 30 minutes of download time is worth keeping. Every recommendation must weigh re-acquisition cost (time, bandwidth, availability) against space saved.
5. **Network environment awareness.** Many users (especially in China) have slow or unreliable internet. Re-downloading a 10GB cache can take hours. Always state the re-download cost when recommending cache deletion. If the user is on a slow connection, lean toward keeping caches.
6. **Speed matters.** Naive `Get-ChildItem -Recurse` / `du` on directories with millions of small files (caches, node_modules, uv cache) or NTFS junctions/reparse points will hang for tens of minutes or forever. Use MFT-reading tools on Windows, parallel Rust tools elsewhere.
7. **Investigate before judging.** Folders with cryptic names (e.g. `onetFile`, `Huorong`, `MineLogin`) must be identified via web search + content inspection before being labeled "safe to delete". Don't guess.
8. **Categorize, don't just list.** A flat list of 500 folders is useless. Group by: system vs user, cache vs data, active app vs uninstalled residue. Rank cleanup candidates by (size × safety).

## Workflow

### Phase 1 — Recon (2 min)

Get the lay of the land before any deep scan. Detect platform once, branch accordingly.

**Windows:**
```powershell
# Get-Volume can fail with CIM permission errors (HRESULT 0x80041031) on some systems;
# fall back to Get-CimInstance Win32_LogicalDisk which uses a different provider.
try {
  Get-Volume | Sort-Object DriveLetter |
    Select-Object DriveLetter, FileSystemLabel, FileSystem,
      @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
      @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,2)}},
      @{N='UsedGB';E={[math]::Round(($_.Size-$_.SizeRemaining)/1GB,2)}},
      @{N='PctFree';E={[math]::Round($_.SizeRemaining/$_.Size*100,1)}} |
    Format-Table -AutoSize
} catch {
  Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3" |
    Select-Object DeviceID, VolumeName,
      @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
      @{N='FreeGB';E={[math]::Round($_.FreeSpace/1GB,2)}},
      @{N='UsedGB';E={[math]::Round(($_.Size-$_.FreeSpace)/1GB,2)}},
      @{N='PctFree';E={if($_.Size -gt 0){[math]::Round($_.FreeSpace/$_.Size*100,1)}}} |
    Format-Table -AutoSize
}

# Root-level listing (fast, non-recursive)
Get-ChildItem C:\ -Directory -Force -ErrorAction SilentlyContinue |
  Select-Object Name, FullName, LastWriteTime
```

**macOS:**
```bash
# Volume overview — APFS containers may show multiple volumes sharing one pool
df -h / /System/Volumes/Data 2>/dev/null | sort -u

# Home directory top-level (fast, non-recursive)
ls -la ~/ | head -40

# /Applications and /Volumes (external/attached drives)
ls -la /Applications/ | wc -l; du -sh /Applications/ 2>/dev/null
ls /Volumes/
```

Record: total capacity, used, free, % free. Flag any drive > 85% full as urgent.

### Phase 2 — Deep scan (the critical choice)

**Windows — use WizTree (MFT direct read).** This is the single most important lesson: PowerShell `Get-ChildItem -Recurse` and even `dust` will hang on AppData (junction loops via `Application Data` → self) and on caches with millions of tiny files (uv cache: 124GB / 12万+ files). WizTree reads the NTFS Master File Table directly and scans a full 500GB drive in ~10 seconds.

Install if missing:
```powershell
winget install --id AntibodySoftware.WizTree -e --accept-source-agreements --accept-package-agreements --silent
```

Export all folders to CSV (silent, no GUI blocking). CSV is cached for reuse:
```powershell
$wiz = 'C:\Program Files\WizTree\WizTree64.exe'
# Cache CSV in a persistent location (not $env:TEMP — it gets cleared by disk cleanup tools).
$cacheDir = "$env:LOCALAPPDATA\disk-audit-cache"
New-Item $cacheDir -ItemType Directory -Force | Out-Null

function Invoke-WizTreeScan($drive, $forceRescan = $false) {
  $csv = Join-Path $cacheDir "wiztree_$($drive.Replace(':','')).csv"
  # Incremental cache: reuse CSV if < 24h old and non-trivial size.
  # User can say "重新扫" / "force rescan" to bypass.
  if (-not $forceRescan -and (Test-Path $csv)) {
    $age = (Get-Date) - (Get-Item $csv).LastWriteTime
    if ($age.TotalHours -lt 24 -and (Get-Item $csv).Length -gt 1000) {
      Write-Host "Reusing cached scan for $drive (age: $([math]::Round($age.TotalHours,1))h). Say 'force rescan' to override." -ForegroundColor DarkGray
      return $csv
    }
  }

  Remove-Item $csv -Force -ErrorAction SilentlyContinue

  # MFT direct read only works with admin rights (WizTree official docs:
  # "high speed NTFS scanning is only possible when running with admin rights").
  # Without admin, WizTree falls back to Windows API mode — functional but slower.
  $isAdmin = ([Security.Principal.WindowsPrincipal]::new(
    [Security.Principal.WindowsIdentity]::GetCurrent()
  )).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
  $adminFlag = if ($isAdmin) { '/admin=1' } else { '/admin=0' }
  if (-not $isAdmin) {
    Write-Host "Non-admin: WizTree will use Windows API mode (slower than MFT). " +
      "Run an admin shell for ~10s MFT-speed scanning." -ForegroundColor Yellow
  }

  $proc = Start-Process -FilePath $wiz `
    -ArgumentList @("`"$drive`"", "/export=`"$csv`"", $adminFlag, '/exportfolders=1', '/exportfiles=0') `
    -PassThru
  # CLI export auto-exits on completion — wait for it rather than polling file size
  # (polling races: CSV appears before it's fully written, causing truncated reads).
  $proc.WaitForExit(300000) | Out-Null
  if (-not $proc.HasExited) { try { $proc.Kill() } catch {} }
  return $csv
}

# Scan C: (and D:, E:, ... as needed)
$csvC = Invoke-WizTreeScan 'C:'
# $csvD = Invoke-WizTreeScan 'D:'
```

Parse the CSV to extract folders at any depth, sorted by size:
```powershell
$sr = [System.IO.StreamReader]::new($csv, [System.Text.Encoding]::UTF8)
$null = $sr.ReadLine(); $null = $sr.ReadLine()  # skip banner + header
$results = [System.Collections.Generic.List[PSObject]]::new()
while (-not $sr.EndOfStream) {
  $line = $sr.ReadLine()
  if ($line -notmatch '^"((?:[^"]|"")+)",(\d+),') { continue }
  $path = $Matches[1] -replace '""','"'
  $size = [long]$Matches[2]
  if (-not $path.EndsWith('\')) { continue }  # folders only
  $depth = ($path -replace '\\$','' -split '\\').Count - 1
  if ($depth -gt 4) { continue }  # top 4 levels for overview
  $results.Add([PSCustomObject]@{Path=$path; SizeGB=[math]::Round($size/1GB,2)})
}
$sr.Close()
$results | Sort-Object SizeGB -Descending | Select-Object -First 80 | Format-Table -AutoSize
```

To drill into a specific subtree (e.g. AppData\Local), filter by path prefix and bump depth:
```powershell
# inside the same loop, replace the depth filter:
if ($path -notlike "$env:LOCALAPPDATA\*") { continue }
if ($depth -ne 5) { continue }  # direct children of AppData\Local
```

**Windows fallback — when WizTree isn't available or user declines to install:**
Use PowerShell but **never recurse blindly** — AppData contains NTFS junctions (e.g. `Application Data` → self) that create infinite loops. The `$exclude` list below only filters top-level names; `Get-ChildItem -Recurse` will still descend into junctions nested deeper. Use the safe recursive function instead.
```powershell
# Safe recursive size function — checks ReparsePoint at EVERY level, not just the top.
function Get-SafeSize($path) {
  $sum = 0
  try {
    foreach ($item in Get-ChildItem $path -Force -ErrorAction SilentlyContinue) {
      if ($item.PSIsContainer) {
        if ($item.Attributes.ToString() -match 'ReparsePoint') { continue }
        $sum += Get-SafeSize $item.FullName
      } else {
        $sum += $item.Length
      }
    }
  } catch {}
  return $sum
}

$exclude = @('AppData','OneDrive','Application Data','Local Settings')  # junctions/cloud
Get-ChildItem "$env:USERPROFILE" -Directory -Force -ErrorAction SilentlyContinue |
  Where-Object { $exclude -notcontains $_.Name -and
                 -not $_.Attributes.ToString().Contains('ReparsePoint') } |
  ForEach-Object {
    [PSCustomObject]@{Name=$_.Name; SizeGB=[math]::Round((Get-SafeSize $_.FullName)/1GB,2)}
  } | Sort-Object SizeGB -Descending
```
For AppData, scan `Local`, `Roaming`, `LocalLow` separately, one level at a time, using the same `Get-SafeSize` function.

**macOS — use `disky` (APFS-aware) or `dust` as fallback.**
`disky` handles sparse files (OrbStack `data.img.raw` reports 8TB logical but 13GB physical — always use `--physical`).
```bash
brew install disky  # or: curl -sSfL https://raw.githubusercontent.com/bootandy/dust/refs/heads/master/install.sh | sh
disky scan /
disky dirs --physical | head -50
disky top --physical | head -50
```

macOS-specific scan targets — these are the usual heavy hitters, scan them explicitly:
```bash
# Xcode: DerivedData (build cache), iOS DeviceSupport, CoreSimulator (iOS simulators)
du -sh ~/Library/Developer/Xcode/DerivedData/ 2>/dev/null
du -sh ~/Library/Developer/Xcode/iOS\ DeviceSupport/ 2>/dev/null
du -sh ~/Library/Developer/CoreSimulator/ 2>/dev/null

# Caches: system + app caches (NOT all safe to delete — see anti-patterns table)
du -sh ~/Library/Caches/ 2>/dev/null
du -sh ~/Library/Caches/Homebrew/ 2>/dev/null

# Docker / OrbStack: sparse VHDX files — use disky --physical, not du
du -sh ~/Library/Containers/com.docker.docker/ 2>/dev/null
du -sh ~/.orbstack/ 2>/dev/null

# Application Support (app data — ask user before touching)
du -sh ~/Library/Application\ Support/ 2>/dev/null

# Logs
du -sh ~/Library/Logs/ 2>/dev/null
```

macOS never-touch paths (equivalent to Windows system dirs):
- `~/Documents`, `~/Desktop`, `~/Pictures`, `~/Movies`, `~/Music` — user data
- `~/Library/Mobile Documents` — iCloud Drive
- `~/Library/Keychains` — keychain data
- `~/Library/Mail` — email
- `~/Library/Messages` — iMessage history
- `~/.ssh`, `~/.gnupg`, `~/.aws` — credentials

**Linux — use `ncdu` or `dust` or `dua`.** All parallel, all fast on ext4/xfs.
```bash
# ncdu (interactive TUI)
ncdu /

# dust (one-shot, Rust)
dust -d 3 -s -p /

# dua (parallel, fastest on SSD)
dua aggregate /
```

### Phase 2.5 — Duplicate file detection (optional, on request)

Triggered when the user says "找重复文件" / "find duplicates" / "dedupe". Not run by default — it's I/O-heavy and most users don't need it every audit.

**Strategy: three-level filter to avoid hashing every file on disk.**

Level 1 — group by size (cheap, no I/O beyond stat):
Level 2 — same-size files: hash first 4KB only (eliminates most false positives);
Level 3 — same 4KB hash: full-file hash to confirm true duplicates.

Only files > 50 MB are considered — smaller duplicates aren't worth the I/O cost to find.

**Windows:**
```powershell
function Find-DuplicateFiles($root, $minSizeMB = 50) {
  # Level 1: collect files > minSize, group by length
  $files = [System.Collections.Generic.List[PSObject]]::new()
  Get-ChildItem $root -Recurse -File -Force -ErrorAction SilentlyContinue |
    Where-Object { $_.Length -gt ($minSizeMB * 1MB) } |
    ForEach-Object { $files.Add($_) }

  $sizeGroups = $files | Group-Object Length | Where-Object { $_.Count -gt 1 }

  $duplicates = [System.Collections.Generic.List[PSObject]]::new()
  foreach ($grp in $sizeGroups) {
    # Level 2: hash first 4KB
    $partialHashes = @{}
    foreach ($f in $grp.Group) {
      try {
        $stream = [System.IO.File]::OpenRead($f.FullName)
        $buffer = [byte[]]::new(4096)
        $null = $stream.Read($buffer, 0, 4096)
        $stream.Close()
        $hash = [System.Security.Cryptography.SHA256]::Create().ComputeHash($buffer)
        $key = [BitConverter]::ToString($hash) -replace '-',''
        if (-not $partialHashes.ContainsKey($key)) { $partialHashes[$key] = @() }
        $partialHashes[$key] += $f
      } catch {}
    }
    # Level 3: full hash for matching partials
    foreach ($entry in $partialHashes.Values) {
      if ($entry.Count -lt 2) { continue }
      $fullHashes = @{}
      foreach ($f in $entry) {
        try {
          $h = (Get-FileHash $f.FullName -Algorithm SHA256).Hash
          if (-not $fullHashes.ContainsKey($h)) { $fullHashes[$h] = @() }
          $fullHashes[$h] += $f
        } catch {}
      }
      foreach ($dup in $fullHashes.Values) {
        if ($dup.Count -lt 2) { continue }
        $duplicates.Add([PSCustomObject]@{
          Hash = $h
          SizeMB = [math]::Round($dup[0].Length / 1MB, 2)
          Count = $dup.Count
          WastedMB = [math]::Round(($dup.Count - 1) * $dup[0].Length / 1MB, 2)
          Paths = $dup.FullName
        })
      }
    }
  }
  return $duplicates | Sort-Object WastedMB -Descending
}

# Usage — scan specific directories (not entire C:\ — too slow):
# $dups = Find-DuplicateFiles 'D:\PROJECTALL'
# $dups | Select-Object WastedMB, Count, SizeMB, Paths | Format-Table -AutoSize
```

**macOS:**
```bash
# Three-level filter in bash: size → 4KB hash → full hash
# Only files > 50MB. Scans specified directories only.
find "$1" -type f -size +50M 2>/dev/null | while read -r f; do
  sz=$(stat -f%z "$f" 2>/dev/null)
  echo "$sz|$f"
done | sort -t'|' -k1,1n | awk -F'|' '
  { size=$1; path=$2; paths[size]=paths[size] " " path; count[size]++ }
  END { for (s in count) if (count[s]>1) print s, paths[s] }
' | while read -r sz paths; do
  for f in $paths; do
    h=$(dd if="$f" bs=4096 count=1 2>/dev/null | shasum -a 256 | cut -d' ' -f1)
    echo "$h|$f"
  done
done | sort -t'|' -k1,1 | awk -F'|' '
  { hash=$1; path=$2; paths[hash]=paths[hash] "\n  " path; count[hash]++ }
  END { for (h in count) if (count[h]>1) printf "Duplicates (%d files):\n%s\n\n", count[h], paths[h] }
'
```

Report duplicates as a separate section in the audit report:
```markdown
## Duplicate files (optional)
| Group | Files | Size each | Wasted | Paths |
List groups sorted by wasted space. Each group shows all duplicate paths.
Deletion of duplicates goes through Phase 6 confirmation — never auto-delete.
```

### Phase 3 — Investigate the unknowns

For any folder whose purpose isn't obvious from the name, do **both**:

1. **Web search** the folder name + "what is" / "can delete". Cite sources.
2. **List contents** (non-recursive first, then shallow recurse if needed):
   ```powershell
   Get-ChildItem 'C:\Huorong' -Force -ErrorAction SilentlyContinue |
     Select-Object Name, Mode, Length | Format-Table -AutoSize
   ```

Common Windows root-level residue patterns to flag:
- Empty folders from uninstalled software (leidian, AppStore, gurobi, MineLogin, ZHIPU) → safe to delete
- `$WinREAgent` → Windows Recovery Environment update temp; safe to delete **after** confirming no pending updates
- `onetFile\recv` → community-reported malware/abnormal component residue; flag for user attention, monitor recurrence
- `OneDriveTemp` → OneDrive sync temp; safe if OneDrive isn't actively syncing
- `hiberfil.sys`, `pagefile.sys`, `swapfile.sys` → system; don't delete directly, manage via `powercfg` / System settings
- `System Volume Information` → system restore points; manage via System Protection settings, never delete directly

Common macOS residue patterns to flag:
- `~/Library/Developer/Xcode/DerivedData/` → Xcode build cache; safe to delete, rebuilds on next build (10-30 min)
- `~/Library/Developer/Xcode/iOS DeviceSupport/` → iOS debug symbols; safe to delete for devices no longer connected, re-downloads from Apple on reconnect
- `~/Library/Developer/CoreSimulator/Devices/` → iOS simulator runtimes; use `xcrun simctl delete unavailable` to clean only obsolete ones
- `~/Library/Caches/Homebrew/` → Homebrew download cache; `brew cleanup -s` clears it safely
- `~/Library/Logs/` → app logs; safe to delete, apps recreate as needed
- `~/.Trash/` → macOS Trash; safe to empty, but check for intentionally-stashed files first
- `/private/var/folders/` → macOS temp; system manages, don't touch directly

### Phase 4 — Categorize and rank

Group every significant item (> 100 MB) into:

| Category | Examples | Default stance |
|----------|----------|----------------|
| **Cache (rebuildable)** | uv cache, npm cache, browser cache, pnpm store, pip cache, Temp, Zed node cache, Homebrew cache, Xcode DerivedData | Safe to clean — auto-rebuilds |
| **App data (active)** | WeChat, QQ, Tencent, Lark, WPS, Edge profile, Chrome profile, ~/Library/Application Support/* | Ask user — may contain user data |
| **App residue (uninstalled)** | empty leidian/, AppStore/, gurobi/, MineLogin/ (Win); orphaned ~/Library/Application Support/<uninstalled app> (macOS) | Safe to delete |
| **Dev tooling** | .cargo, .rustup, .conda, .m2, node_modules, ms-playwright, iOS DeviceSupport, CoreSimulator | Ask if still using the toolchain |
| **AI editor army** | .gemini, .cursor, .trae, .antigravity, .codex, .qoder, .kiro | Ask which ones still in use |
| **Mobile backup** | CrossDevice\<phone>\ (Win); ~/Library/Application Support/MobileSync/Backup (macOS) | Ask user — may be intentional |
| **System** | Windows, WinSxS, Recovery, System Volume Information (Win); /System, /private/var, /usr (macOS) | Don't touch directly; use `dism` / System Protection / `brew cleanup` |
| **Virtual disks** | Docker VHDX, WSL VHDX (Win); Docker/OrbStack sparse images (macOS) | Compact after pruning (Win: `optimize-vhd`; macOS: OrbStack auto-compacts) |
| **User files** | Documents, Desktop, Pictures, Videos (Win); ~/Documents, ~/Desktop, ~/Pictures, ~/Movies, ~/Music (macOS) | Never suggest deleting — user's data |

Rank cleanup candidates by `size × safety`:
- 🟢 **Green tier**: large + safe (caches, temp, residue) → list first
- 🟡 **Yellow tier**: large + needs command (Docker prune + compact, browser cache, app-internal cache) → list second with the exact command
- 🟠 **Orange tier**: needs user decision (AI tools still installed?, mobile backup, hibernation file, conda envs) → list third as questions
- 🔴 **Red tier**: do not touch (system, active app data user wants kept) → list last as "don't touch" with reason

### Phase 5 — Report

Use this exact structure (adapt the prose to the user's language — Chinese in, Chinese out):

```markdown
# [Drive letter] Disk Space Audit Report

## Overview
| Drive | Capacity | Used | Free | % Free |
(table from Phase 1)

## Where the bytes went — top-level breakdown
(table: region, size, % of used, notes)
Note: WizTree counts logical size; NTFS hard links (WinSxS) double-count, so
column sums exceed physical used. Trust the volume-level number for truth.

## Deep dive — [largest region, usually Users]
### AppData\Local — X GB
(table: directory, size, what it is, cleanable?)
### AppData\Roaming — X GB
(table)
### Non-AppData user dirs
(table)
### Root-level residue
(table with investigation conclusions)

## Cleanup recommendations (ranked)
Every item MUST have a "consequence if deleted" column — never list size alone.
State re-download / rebuild cost for caches. Items appearing in both Tier 1 and the anti-patterns table below are NOT contradictions: Tier 1 means "safe to execute when the user asks", anti-patterns means "don't proactively recommend without stating re-download cost". When the user asks for an anti-pattern item, state the cost first, then proceed on confirmation.

### 🟢 Tier 1 — Safe, high yield (~X GB recoverable)
(table: item, size, method, re-acquire cost, risk)
### 🟡 Tier 2 — Needs action but high yield (~X GB)
### 🟠 Tier 3 — Needs your decision
### 🔴 Do not touch (with reason)

## Recommended tools
(table: tool, purpose, install command, status)

## Summary
One paragraph: total recoverable, what clearing achieves (% free after).
```

### Phase 6 — Cleanup (ONLY when user explicitly asks)

The audit (Phases 1-5) is read-only and always safe to run. Cleanup is a separate phase that begins only when the user says something like "go ahead and clean item 1 and 3" or "删除 uv cache". It follows a strict confirmation protocol.

#### 6.1 Intent resolution — never skip this

Before any deletion, resolve intent to a concrete, unambiguous list of items:

- **Vague request → ask.** "Clean some stuff", "free up space", "你看着办" are NOT sufficient. Respond with a focused question: "I can clean these N items totaling X GB. Which ones do you want me to do? (list them)".
- **Scope creep → ask.** If the user approved "uv cache" but you're tempted to also clear "npm cache because it's the same kind of thing", DON'T. Only delete what was explicitly approved. Mention the adjacent item as a suggestion, don't act on it.
- **Ambiguous target → ask.** If "delete the Docker stuff" could mean images, containers, volumes, or VHDX compaction, list the sub-options and ask which.
- **One approval ≠ blanket approval.** "Yes clean uv cache" does not authorize cleaning npm cache next. Each item is its own decision.

#### 6.2 Pre-deletion checklist — run for EVERY item

For each item the user approved, before executing:

1. **Re-verify the path exists and matches** what was in the report. Caches move; apps get reinstalled; junctions redirect. Re-stat the path.
2. **Check the owning app isn't running.** Deleting a cache while the app writes to it can corrupt the app. On Windows: `Get-Process | Where-Object { $_.Path -like '*<app>*' }`. On macOS: `pgrep -fl <app>`.
3. **Check for symlinks/junctions at the target.** `Get-Item <path> | Select-Object Attributes, Target`. If it's a reparse point, resolving it might delete files elsewhere — STOP and report.
4. **Show the exact command** you're about to run, with the resolved absolute path, and the expected size freed. Get a final "yes" for that specific command.
5. **Prefer the app's own cleanup command** over manual deletion when one exists:
   - `uv cache clean` over `Remove-Item $env:LOCALAPPDATA\uv\cache`
   - `npm cache clean --force` over deleting `npm-cache`
   - `pnpm store prune` over deleting pnpm store
   - `docker system prune -a` (only if user confirms "all unused") over manual VHDX deletion
   - `brew cleanup -s` (macOS) over deleting `~/Library/Caches/Homebrew/`
   - `xcrun simctl delete unavailable` (macOS) over deleting all CoreSimulator devices
   - `tmutil deletelocalsnapshots <date>` (macOS) over deleting Time Machine snapshots manually
   - App's in-app "clear cache" button (tell the user to click it) over deleting app data folders
6. **Set time expectations for large deletions.** If the cache is >50GB or has millions of files, tell the user upfront: "this will take 20-40 minutes, no progress bar, I'll check disk space periodically to confirm it's working." This prevents the user from thinking it's hung.
7. **For Docker specifically — NEVER use `docker volume prune` or `docker system prune --volumes`** without per-volume confirmation. Volumes hold database data, project state, user uploads. List each volume with `docker volume inspect`, identify the owning project, and confirm per project. `docker builder prune` (build cache only) is the one safe prune.
8. **For VHDX compaction** (Docker/WSL): warn that `wsl --shutdown` will kill all running WSL sessions and Docker. Confirm the user has no active work in WSL/Docker containers.

#### 6.3 Mid-cleanup vigilance — stop on new information

Even after per-item confirmation, if during execution you discover:

- The path is 10× larger than the report said (something else got installed there)
- The path is a symlink/junction to a different location than expected
- The owning app is actually running (process check fails — app started since report)
- Deletion errors out with "file in use" or "access denied" in an unexpected way
- A subdirectory looks like user data, not cache (e.g. `uv\cache\` contains a folder named `my-project-backup`)

**STOP. Report what you found. Ask before continuing.** Confirmation is not a blank check; new information resets it. This is non-negotiable.

#### 6.4 Post-deletion verification

After each deletion:
- Confirm the path is gone (`Test-Path` returns false)
- Report actual space freed (`Get-Volume` before/after delta)
- If space freed << reported size, investigate (hard links, sparse files, VHDX not compacted)
- For VHDX: remind user that the `.vhdx` file won't shrink until `optimize-vhd` is run

#### 6.5 What NEVER to do in cleanup

- ❌ `rm -rf` / `Remove-Item -Recurse -Force` on a path you haven't re-verified in the last 60 seconds
- ❌ `docker volume prune` / `docker system prune --volumes` without per-volume confirmation
- ❌ `docker container prune` (stopped containers may be restarted; ask per container)
- ❌ Deleting `~/.cache/uv`, `~/.cache/huggingface`, `~/.cache/modelscope` without stating re-download cost (can be hours on slow networks)
- ❌ Deleting `node_modules` in a project the user is actively working on (breaks `npm run dev`)
- ❌ Deleting `~/Library/Caches` wholesale on macOS (some subfolders are IDE index caches that take 10+ min to rebuild)
- ❌ Running `powercfg /h off` without explaining that hibernation will be permanently disabled
- ❌ Batch-confirming ("I'll clean items 1-8, ok?") — each item gets its own confirmation unless the user explicitly says "all of tier 1"
- ❌ Proceeding when the user says "sure" to a question you didn't actually ask clearly

## Anti-patterns: what NOT to recommend deleting

These are often flagged as "cache, safe to delete" by naive cleanup tools. They are NOT — deleting them causes real harm (long rebuilds, lost state, multi-hour re-downloads). Always include a "consequence" column in your report and mark these explicitly as "keep unless desperate". **If the user explicitly asks to clean one of these, you may proceed — but only after stating the re-download/rebuild cost and getting confirmation for that specific item.**

| Item | Typical size | Why NOT to delete | Real impact of deletion |
|------|-------------|-------------------|------------------------|
| **uv cache** | 10-120 GB | Python package cache; rebuilding requires re-downloading all deps from PyPI | Every Python project redownloads deps; 30min-2hr on slow networks |
| **npm _cacache** | 5-20 GB | Downloaded packages cached locally | `npm install` redownloads everything; 30min-2hr in China |
| **pnpm store** | 1-10 GB | Hard-linked package store; all projects share it | All pnpm projects break until `pnpm install` re-runs |
| **Playwright browsers** | 2-4 GB | Browser binaries for automation testing | Redownload 2GB+ each time; 30min-1hr |
| **HuggingFace / ModelScope cache** | varies (GB-TB) | AI model weights | Redownload large models; hours to days |
| **Docker stopped containers** | <500 MB each | May restart anytime with `docker start` | Lose container state; need to recreate |
| **Docker volumes** | varies | Database data, project state, user uploads | **Data loss** — often irreplaceable |
| **JetBrains / VS Code index caches** | 1-5 GB | IDE indexing caches | IDE takes 5-10 min to re-index on next launch |
| **Xcode DerivedData** (macOS) | 10+ GB | Build cache | Next build takes 10-30 min longer |
| **iOS DeviceSupport** (macOS) | 2-3 GB | Required for device debugging | Redownload from Apple when connecting device |
| **Conda envs** | 1-10 GB each | Installed package environments | Must recreate env; `conda install` is slow |
| **.rustup toolchains** | 1-2 GB each | Rust compiler + stdlib | Must reinstall via `rustup`; 10-30 min |
| **WSL distro VHDX** | 5-50 GB | Entire Linux filesystem for that distro | **Data loss** — all files in the distro gone |
| **WeChat / QQ / Feishu data** | 5-50 GB | Chat history, sent files, user data | **Data loss** — chat history not recoverable |
| **OneDrive / iCloud local copies** | varies | User documents, even if cloud-backed | May trigger re-download; disruptive |

**The vanity trap**: Showing "Cleaned 50GB!" feels good, but if 40GB of that was useful cache, the user now has a slower machine and hours of re-downloads. The right question is "what is truly useless?" not "what can I delete to show a big number?". When unsure, keep.

## Guardrails

- **Audit phase: never delete anything.** The report is the deliverable. Cleanup is a separate, explicitly-confirmed phase.
- **Cleanup phase: per-item confirmation, always.** No batch approvals unless the user explicitly says "all of tier X". No rounding up ambiguous consent.
- **Ambiguous intent → ask one more question, not one fewer.** "Clean some stuff" is not authorization. "Delete uv cache and npm cache, nothing else" is.
- **Mid-cleanup surprise → stop and re-confirm.** New information (path changed, app running, symlink, unexpected size) resets prior approval.
- **Never run `rm -rf`, `Remove-Item -Recurse -Force`, `optimize-vhd`, `docker system prune`, `docker volume prune`, `powercfg /h off`, or any state-changing command without (a) re-verifying the target in the last 60 seconds and (b) showing the exact command and getting a final yes.**
- **Never use `docker volume prune` or `docker system prune --volumes`** without per-volume inspection and per-project confirmation. Volumes hold irreplaceable data.
- **Cite sources** for any claim about what a folder is or whether it's safe to delete. "I think it's..." is not acceptable — search or inspect.
- **Flag suspicious folders.** If something looks like malware residue (recurring empty folders with no owning app, names like `onetFile`), say so and point to community reports — don't just silently recommend deletion.
- **Respect user keep-lists.** If the user says "don't touch WeChat/QQ/Feishu", honor it without re-suggesting. Don't re-list kept items in later passes hoping for a different answer.
- **State re-download cost** for every cache you recommend deleting. "uv cache 124GB" alone is not enough — say "124GB; re-download takes ~2hr on a 100Mbps connection, longer on slow networks".
- **Distinguish logical vs physical size.** On Windows, WizTree reports logical size; NTFS hard links (especially WinSxS) inflate sums. Always reconcile against `Get-Volume` for the ground truth. On macOS APFS, sparse files make logical size wildly misleading — use `disky --physical` or `du -A`.

## Tool reference

| Platform | Tool | Why | Install |
|----------|------|-----|---------|
| Windows | **WizTree 4.31** | Reads NTFS MFT directly; scans 500GB in ~10s; handles millions of files without hanging | `winget install AntibodySoftware.WizTree` |
| Windows | dust | CLI du-replacement; good for single-directory quick checks; **will hang on AppData junctions and million-file caches** — use only on isolated subtrees | `winget install bootandy.dust` |
| Windows | PowerShell `Get-ChildItem` | Always available; use only with `-ErrorAction SilentlyContinue`, batched by directory, skipping reparse points | built-in |
| macOS | **disky** | APFS-aware (`--physical`), Rust, Trash-restorable cleanup, JSON for agents | `brew install disky` |
| macOS | GrandPerspective | GUI treemap | `brew install --cask grandperspective` |
| Linux | **ncdu** | Interactive TUI, the classic | `apt install ncdu` / `dnf install ncdu` |
| Linux | dua | Parallel Rust, fastest on SSD | `cargo install dua-cli` or install script |
| All | dust | Cross-platform, simple | see bootandy/dust GitHub |

## Known traps (learned the hard way)

- **AppData\Local contains `Application Data` junction → self-reference.** Any recursive scanner that follows junctions will loop forever. WizTree handles this; PowerShell `Get-ChildItem -Recurse` does not (even with `-Force`). Always filter `Attributes -match 'ReparsePoint'` or use WizTree.
- **uv cache can be 100+ GB of tiny files.** `Get-ChildItem -Recurse` on it takes 10+ minutes and may never finish. WizTree reads it via MFT in seconds. If forced to use PowerShell, stream with `[System.IO.Directory]::EnumerateFiles(..., AllDirectories)` and aggregate manually — still slow but won't OOM.
- **WizTree CLI export auto-exits on completion.** Use `$proc.WaitForExit(300000)` (5 min) — don't poll file size (CSV appears before it's fully written, causing truncated reads). The old "poll then Kill" approach caused incomplete CSVs on large drives. 120s timeout is too short for 500GB+ drives with millions of files; observed C: scan taking ~3 min in non-admin mode.
- **WizTree `/exportfiles=0` flag is unreliable** — files may still appear in CSV. Filter by `path.EndsWith('\')` to keep only folders.
- **Docker VHDX and WSL VHDX grow but don't shrink.** Even after `docker system prune`, the `docker_data.vhdx` file stays large. Must `wsl --shutdown` then `optimize-vhd -Path ... -Mode full` (admin) to compact. On Windows Home without Hyper-V tools, use `diskpart` → `compact vdisk`.
- **OneDrive files are often cloud placeholders (reparse points).** Recursive size scans report near-zero for them. Don't mistake OneDrive folder for "empty" — check `Attributes` for `ReparsePoint`/`Offline`.
- **CrossDevice / phone backup folders** can be huge (35GB observed) and are easy to miss because they're not under AppData. Always scan `C:\Users\<user>\` depth-1 fully.
- **Deleting millions of tiny files takes a long time with no progress output.** `uv cache clean` on 124GB / 5.4M files took ~30 minutes with zero stdout. Don't assume it's hung — monitor via `Get-Volume -DriveLetter C | Select SizeRemaining` every few minutes to confirm space is being released. If space isn't moving after 5+ minutes, then investigate. Consider telling the user upfront: "this will take 20-40 minutes, I'll check progress periodically."
- **Disk freed ≠ bytes deleted on NTFS.** NTFS cluster sizing, MFT overhead, and hard links mean `Get-Volume` before/after delta is the only reliable measure of actual reclamation. Never report "freed X GB" based on the tool's own delete count — always measure the volume delta. If the tool says "deleted 97GB" but volume only gained 23GB, report the 23GB and explain the gap.
- **macOS APFS sparse files report logical size, not physical.** OrbStack's `data.img.raw` can show 8TB logical but 13GB physical. Always use `disky --physical` or `du -A` on macOS — plain `du -h` will show the logical size and mislead you into thinking OrbStack/Docker is eating terabytes.
