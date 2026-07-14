# Windows cleanup avenues

**Health first:** volume free space, `Get-PhysicalDisk` health, memory/pagefile, RAM Speed vs ConfiguredClockSpeed, failed auto services — see [`health.md`](health.md).

Fingerprint with PowerShell (native or `powershell.exe` / `pwsh` from WSL/Git Bash):

```powershell
Get-CimInstance Win32_OperatingSystem |
  Select-Object Caption, Version, BuildNumber, OSArchitecture,
                @{N='FreeGB';E={[math]::Round($_.FreePhysicalMemory/1MB,1)}}
Get-PSDrive -PSProvider FileSystem |
  Select-Object Name, @{N='UsedGB';E={[math]::Round(($_.Used/1GB),2)}},
                @{N='FreeGB';E={[math]::Round(($_.Free/1GB),2)}}
$PSVersionTable.PSVersion
```

| Build | OS |
|-------|-----|
| 19041–19045 | Windows 10 20H1–22H2 |
| 22000+ | Windows 11 |
| Server 2019/2022/2025 | Server — be more conservative on production roles |

Use **Administrator** PowerShell only when required (DISM, some cleanmgr profiles).

---

## Built-in storage tools

### Storage Sense (Win10 1709+ / Win11)

```powershell
# Policy / status varies; GUI: Settings → System → Storage
# Enable temporary file cleanup via Settings when interactive
```

Prefer guiding interactive Storage Sense for temp files and Recycle Bin when the session is user-attended.

### Disk Cleanup (`cleanmgr`)

```powershell
cleanmgr /d C:
# sageset/sagerun for scripted profiles (medium complexity; document numbers used)
# cleanmgr /sageset:1
# cleanmgr /sagerun:1
```

### Clear temporary directories (user + system)

```powershell
$env:TEMP
Get-ChildItem $env:TEMP -ErrorAction SilentlyContinue |
  Measure-Object -Property Length -Sum
# User temp
Remove-Item -Path "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue
# System temp needs Admin
# Remove-Item -Path "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue
```

Skip files in use (errors expected). **Low risk** for true temp data.

### Recycle Bin

```powershell
Clear-RecycleBin -Force -ErrorAction SilentlyContinue
```

Confirm if user may still need trashed files.

---

## Windows Update and component store

### SoftwareDistribution / Delivery Optimization

```powershell
# Delivery Optimization cache
du-equivalent:
Get-ChildItem "$env:SystemRoot\SoftwareDistribution\Download" -Recurse -ErrorAction SilentlyContinue |
  Measure-Object Length -Sum
# Stop bits/wuauserv before aggressive deletes (Admin) — prefer DISM and Disk Cleanup categories
```

Prefer:

```powershell
# Component store analysis (Admin)
Dism.exe /Online /Cleanup-Image /AnalyzeComponentStore
# Safe cleanup
Dism.exe /Online /Cleanup-Image /StartComponentCleanup
# Aggressive (HIGH — cannot uninstall superseded updates easily)
# Dism.exe /Online /Cleanup-Image /StartComponentCleanup /ResetBase
```

`/ResetBase` is **high risk** — only with explicit approval on non-critical machines.

### Windows.old

Present after major upgrades; multi-GB to tens of GB.

```powershell
Test-Path C:\Windows.old
# Remove via Disk Cleanup “Previous Windows installation(s)” or:
# Admin: automated cleanup scheduled by Windows after 10 days often
```

Irreversible rollback loss — **high**, confirm.

---

## WinGet / Chocolatey / Scoop

```powershell
Get-Command winget, choco, scoop -ErrorAction SilentlyContinue

# WinGet cache / old installers (paths vary by version)
winget --info
# winget source reset if corrupted — not a space tool
# Installer cache often under:
# $env:LOCALAPPDATA\Temp\WinGet
# $env:ProgramData\Microsoft\WinGet\Packages (do not bulk-delete packages dir)

# Chocolatey
choco list -l
# Cache:
# C:\ProgramData\chocolatey\lib  (installed pkgs — not a cache)
# C:\Users\<user>\AppData\Local\Temp\chocolatey
# choco has limited official “cache clear”; remove Temp\chocolatey with care

# Scoop
scoop cache show
scoop cache rm *
scoop cleanup *           # old versions — after approval
```

---

## Developer and container stacks on Windows

### Docker Desktop

```powershell
docker system df
docker system prune -f
# docker image prune -a -f   # after approval
```

Disk image under WSL2 backend: prune inside engine first; then use Docker Desktop **Troubleshoot → Clean / Purge data** only if user accepts full data loss inside Docker.

### WSL2 disk reclaim

Linux-side cleanup alone may not shrink `ext4.vhdx`.

```powershell
wsl -l -v
# In distro: clean packages/docker first
# Then shutdown and compact (HIGH impact, needs Admin often):

wsl --shutdown
# Optimize-VHD requires Hyper-V module:
# Optimize-VHD -Path $env:LOCALAPPDATA\Packages\...\LocalState\ext4.vhdx -Mode Full
# Or: diskpart → select vdisk file=... → attach readonly → compact vdisk → detach
```

Document the VHDX path from:

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Packages" -Recurse -Filter ext4.vhdx -ErrorAction SilentlyContinue |
  Select-Object FullName, @{N='GB';E={[math]::Round($_.Length/1GB,2)}}
# Also: %USERPROFILE%\AppData\Local\wsl\...
```

### Visual Studio / Build Tools

```powershell
# Installer cache and old MSIs can be huge
# "%ProgramData%\Microsoft\VisualStudio\Packages"
# VS Installer → Install cleanup
# Component cache:
du-like measure of:
# $env:LOCALAPPDATA\Microsoft\VisualStudio
# $env:TEMP\VSLogs
```

Use **Visual Studio Installer** to remove unused workloads rather than deleting folders by hand.

### NuGet / MSBuild / .NET

See also `devtools.md`:

```powershell
dotnet nuget locals all --list
dotnet nuget locals all --clear    # after approval
```

### npm / pnpm on Windows

```powershell
npm cache ls
npm cache clean --force
pnpm store path
pnpm store prune
```

Paths under `%LOCALAPPDATA%` and `%APPDATA%`.

---

## Hibernation, pagefile, System Restore

| Item | Command / note | Risk |
|------|----------------|------|
| Hibernation (`hiberfil.sys`) | `powercfg /hibernate off` | medium — loses hibernate |
| Pagefile | Do **not** remove casually | **high** stability |
| System Restore / shadow copies | `vssadmin list shadowstorage` | medium–high |
| | `vssadmin Resize ShadowStorage` / delete shadows | confirm |

```powershell
vssadmin list shadowstorage
# powercfg /hibernate off    # only if user wants disk over hibernate
```

---

## Event logs and dumps

```powershell
Get-ChildItem C:\Windows\Logs, C:\Windows\Minidump -ErrorAction SilentlyContinue
# Clear a log (medium — loses diagnostics):
# wevtutil cl Application
# Prefer sizing report over mass-clear on servers
```

Crash dumps: `C:\Windows\MEMORY.DMP`, `C:\Windows\Minidump` — low risk to delete old dumps if not debugging.

---

## Browsers and user profile caches

| Path | Notes |
|------|-------|
| `%LOCALAPPDATA%\Temp` | primary temp |
| `%LOCALAPPDATA%\Microsoft\Windows\INetCache` | IE/Edge legacy cache |
| Edge/Chrome caches under `%LOCALAPPDATA%\Microsoft\Edge` / `Google\Chrome` | clear via browser or cache subdirs only |
| `%LOCALAPPDATA%\Pip\Cache` | Python |
| `%USERPROFILE%\.cache` | some Unix-y tools on Windows |

Do not delete entire `%USERPROFILE%` or `NTUSER.DAT` sidecars.

---

## Windows 10 vs 11 notes

### Windows 10

- Storage Sense + Disk Cleanup still primary GUIs.
- `Dism /AnalyzeComponentStore` critical after years of updates.
- Feature updates leave `Windows.old` frequently.

### Windows 11

- Storage settings redesigned; same DISM/cleanmgr backend tools remain valid.
- Widgets/Team residual packages rare; do not remove system AppX packages as “cleanup” without expertise.

### Windows Server

- Skip user-oriented browser cache focus.
- Prefer DISM, WinSxS analysis, log sizing, unused roles/features via Server Manager.
- Never `ResetBase` on production without change window.

---

## Safe default Windows sequence

1. Measure free space on system drive  
2. User `%TEMP%` clear + Recycle Bin (if approved)  
3. `docker system prune` / devtool caches if present  
4. WinGet/Scoop/Chocolatey caches  
5. `Dism /AnalyzeComponentStore` → `StartComponentCleanup` (Admin, approved)  
6. Storage Sense / cleanmgr for Windows Update leftovers  
7. Windows.old / WSL compact / hibernate only with explicit OK  

---

## Avoid

- Registry cleaners  
- Third-party “optimizer” suites  
- Deleting `C:\Windows\WinSxS` manually  
- Stopping security services to delete files  
- Bulk-removing Appx packages for “bloat” without a known list and restore plan  
