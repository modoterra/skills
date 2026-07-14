# System health signals

Use this reference in **every** defrag run after the host is fingerprinted. Goal: classify the machine as **healthy**, **degraded**, or **unhealthy** across independent domains, then connect findings to cleanup, configuration, or hardware follow-ups.

Signals are **indicators**, not diagnoses. Correlate at least two related signals before declaring root cause. Prefer read-only probes; escalate privilege only when needed (SMART, full `dmidecode`, some Windows WMI).

---

## Severity rubric

| Level | Meaning | Typical action |
|-------|---------|----------------|
| **ok** | Within normal bounds | Note and continue |
| **watch** | Elevated or suboptimal; not urgent | Mention in report; optional fix |
| **warn** | Likely impacting performance or headroom | Prioritize; propose remediation |
| **crit** | Imminent failure, data risk, or severe unusability | Lead the report; stop deep cleans that need free space if disk is the crisis |

Domain roll-up: overall status = worst domain severity (except purely informational firmware notes).

---

## Domain map

| Domain | Healthy patterns | Unhealthy patterns |
|--------|------------------|--------------------|
| **Disk capacity** | Free ≥20% on critical mounts; inodes plentiful | Free &lt;10–15%; inode exhaustion; root 100% |
| **Disk integrity** | SMART PASSED; no pending sectors; FS clean | SMART FAILING; reallocated/pending rising; remount-ro |
| **Memory** | Available headroom; little thrashing | Chronic swap; OOM kills; commit near limit |
| **CPU / load** | Load ≲ cores; low sustained iowait | Load ≫ cores; thermal throttle; stuck D-state |
| **Firmware / platform** | RAM at rated speed/capacity; virt as needed | Half RAM missing; DIMMs at JEDEC not XMP; empty slots unused by choice vs fault |
| **Thermal / power** | Temps in spec; battery ≥80% design (laptop) | Thermal throttle; battery &lt;60% design; AC not charging |
| **Services / software** | No failed units; time synced; pkgs consistent | Failed multi-user.target deps; broken packages; clock drift |
| **Containers / virt** | Docker disk headroom; VMs healthy | Docker root full; WSL VHD maxed; disk pressure from images |
| **Network** (light) | Default route + DNS resolve | No route; captive portal; flapping interface |
| **Security hygiene** (light) | Updates available but system boots; disk encrypted if policy | Months of pending updates; backup agent dead (report only) |

---

## 1. Disk capacity

Simplest and highest-signal health metric. Always collect.

### Thresholds (critical data mounts: `/`, `C:\`, `/System/Volumes/Data`, user home)

| Free space | Severity |
|------------|----------|
| ≥ 20% and ≥ 10 GiB | **ok** |
| 10–20% or &lt; 10 GiB on large disks | **watch** |
| 5–10% | **warn** |
| &lt; 5% or &lt; 1–2 GiB absolute | **crit** |

On multi-TB volumes, prefer **absolute free** as well as percent (5% of 4 TB is still huge; 5% of 128 GB is not).

### Linux

```bash
df -hT
df -ih                          # inode capacity — full inodes == writes fail with "No space left"
findmnt -A -o TARGET,SOURCE,FSTYPE,OPTIONS,AVAIL,USE%
# Read-only or errors mounts
findmnt -A -o TARGET,OPTIONS | grep -E 'ro,|errors='
```

| Signal | Unhealthy |
|--------|-----------|
| `Use%` ≥ 90–95% on `/` or `/var` | warn/crit |
| `IUse%` ≥ 90% | warn/crit (many small files: mail, containers, node_modules trees) |
| Mount option `ro` unexpectedly | crit |
| Separate `/var` or `/var/lib/docker` full while `/` free | containers/logs domain |

### macOS

```bash
df -H /
df -H /System/Volumes/Data 2>/dev/null
# APFS may show purgeable space — freeable by OS under pressure
diskutil info / | egrep -i 'Container|Volume Free|File System|SMART|Solid State'
```

| Signal | Notes |
|--------|-------|
| Low free on Data volume | Same thresholds; APFS purgeable is not guaranteed free for large installs |
| `diskutil apfs list` | Container capacity vs volume usage |

### Windows

```powershell
Get-PSDrive -PSProvider FileSystem |
  Select-Object Name,
    @{N='FreeGB';E={[math]::Round($_.Free/1GB,2)}},
    @{N='UsedGB';E={[math]::Round($_.Used/1GB,2)}},
    @{N='FreePct';E={
      $t=$_.Used+$_.Free; if($t -eq 0){0}else{[math]::Round(100*$_.Free/$t,1)}
    }}
Get-Volume | Select-Object DriveLetter, FileSystemLabel, FileSystem,
  @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
  @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,2)}},
  HealthStatus, OperationalStatus
```

| Signal | Unhealthy |
|--------|-----------|
| `HealthStatus` ≠ Healthy | warn/crit |
| System drive free &lt; 10–15% | warn; Windows Update and pagefile suffer |
| BitLocker / volume degraded | integrity domain |

**Remediation link:** low free space → cleanup avenues (package caches, Docker, logs). Capacity health drives **priority** of Phase 3 prune.

---

## 2. Disk integrity (SMART, filesystem, RAID)

Capacity can look fine while the disk is dying.

### Linux SMART

```bash
# List candidates
lsblk -d -o NAME,SIZE,MODEL,ROTA,TYPE
# smartmontools (install only if user approves; otherwise skip)
sudo smartctl -H /dev/nvme0n1 2>/dev/null
sudo smartctl -H /dev/sda 2>/dev/null
sudo smartctl -A /dev/nvme0n1 2>/dev/null | egrep -i 'Media|Percentage|Temperature|Error|Spare|Unsafe'
sudo smartctl -A /dev/sda 2>/dev/null | egrep -i 'Reallocated|Pending|Uncorrectable|Temperature|Power_On'
# NVMe without smartctl
sudo nvme smart-log /dev/nvme0n1 2>/dev/null
```

| Attribute / field | Concern |
|-------------------|---------|
| SMART overall **FAILED** | **crit** — backup and replace |
| Reallocated / pending / offline uncorrectable &gt; 0 and rising | **warn→crit** |
| NVMe available spare low / percentage used high | **warn** |
| Media errors, critical warning flags | **crit** |
| Temperature sustained above vendor limit | **warn** (thermal) |

`ROTA=1` → HDD (defrag of filesystem is still out of scope; fragmentation rarely primary on modern FS). `ROTA=0` → SSD/NVMe.

### Filesystem / kernel complaints

```bash
# Recent storage errors
dmesg -T 2>/dev/null | egrep -i 'I/O error|ext4|xfs|btrfs|nvme|Buffer I/O|reset|offline' | tail -n 40
journalctl -k -p err..alert --no-pager -n 50 2>/dev/null
# Btrfs
sudo btrfs device stats / 2>/dev/null
# mdadm
cat /proc/mdstat 2>/dev/null
sudo mdadm --detail /dev/md0 2>/dev/null
```

| Signal | Severity |
|--------|----------|
| I/O errors in dmesg | warn/crit |
| Btrfs write/read/flush/corruption errs | crit |
| md RAID `[U_]` degraded | crit |
| Remounted read-only | crit |

### macOS

```bash
diskutil info disk0 | egrep -i 'SMART|Solid State|Protocol|Device Location'
# system_profiler for storage
system_profiler SPStorageDataType SPNVMeDataType 2>/dev/null | head -n 80
# S.M.A.R.T. in Disk Utility is GUI; CLI limited without third-party
```

If SMART status is **Failing**, treat as **crit**.

### Windows

```powershell
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, HealthStatus, OperationalStatus, Size
Get-Disk | Select-Object Number, FriendlyName, HealthStatus, OperationalStatus, PartitionStyle
# Storage reliability counters (when available)
Get-StorageReliabilityCounter -PhysicalDisk (Get-PhysicalDisk) -ErrorAction SilentlyContinue |
  Format-List
```

| Signal | Severity |
|--------|----------|
| PhysicalDisk HealthStatus Warning/Unhealthy | warn/crit |
| chkdsk dirty bit | warn — schedule check |
| Frequent `disk` reset errors in Event Log | warn |

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2,3; StartTime=(Get-Date).AddDays(-7)} -ErrorAction SilentlyContinue |
  Where-Object { $_.ProviderName -match 'disk|ntfs|storahci|nvme|Disk' } |
  Select-Object -First 20 TimeCreated, ProviderName, Id, Message
```

**Remediation:** integrity issues are **not** fixed by cache prune. Report backup urgency; cleanup only to free space for backup tools if needed.

---

## 3. Memory pressure and capacity

### Linux

```bash
free -h
cat /proc/meminfo | egrep 'MemTotal|MemAvailable|SwapTotal|SwapFree|Dirty|Writeback|AnonPages'
# Pressure stall info (kernel 4.20+ with PSI)
cat /proc/pressure/memory 2>/dev/null
cat /proc/pressure/cpu 2>/dev/null
cat /proc/pressure/io 2>/dev/null
# OOM history
journalctl -k --no-pager 2>/dev/null | egrep -i 'Out of memory|Killed process|oom-kill' | tail -n 20
dmesg -T 2>/dev/null | egrep -i 'Out of memory|Killed process' | tail -n 10
# Who is huge
ps aux --sort=-%mem | head -n 15
```

| Signal | Severity |
|--------|----------|
| MemAvailable &lt; ~10% of MemTotal under idle/light load | warn |
| Swap used heavily + high `si`/`so` in `vmstat 1 5` | warn |
| Recent OOM killer events | warn/crit |
| PSI memory `some` avg10 chronically high | warn |
| No swap on memory-constrained hosts (optional watch) | watch |

```bash
vmstat 1 5
# High 'wa' = iowait; high 'si/so' = swap thrash
```

### macOS

```bash
# Memory pressure: green/yellow/red style via memory_pressure
memory_pressure 2>/dev/null | head -n 40
vm_stat
sysctl hw.memsize hw.physicalcpu hw.logicalcpu
# Top memory consumers
ps aux -m | head -n 15
```

| Signal | Severity |
|--------|----------|
| memory_pressure reports critical / pages heavily compressed with swap | warn |
| Swap file growing large on nearly full disk | warn (disk + memory) |

### Windows

```powershell
Get-CimInstance Win32_OperatingSystem | Select-Object `
  @{N='TotalGB';E={[math]::Round($_.TotalVisibleMemorySize/1MB,2)}}, `
  @{N='FreeGB';E={[math]::Round($_.FreePhysicalMemory/1MB,2)}}, `
  @{N='FreePct';E={[math]::Round(100*$_.FreePhysicalMemory/$_.TotalVisibleMemorySize,1)}}
Get-Counter '\Memory\Available MBytes','\Memory\Pages/sec','\Paging File(*)\% Usage' -ErrorAction SilentlyContinue
Get-CimInstance Win32_PageFileUsage | Select-Object Name, AllocatedBaseSize, CurrentUsage
```

| Signal | Severity |
|--------|----------|
| Available RAM chronically low + high pages/sec | warn |
| Page file usage near 100% | warn/crit |
| Resource-Exhaustion events in System log | warn |

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Microsoft-Windows-Resource-Exhaustion-Detector'; StartTime=(Get-Date).AddDays(-14)} -ErrorAction SilentlyContinue |
  Select-Object -First 10 TimeCreated, Id, Message
```

**Remediation link:** memory pressure → identify hogs, reduce browser/Docker limits, add RAM; not solved by `apt clean` unless disk-backed swap is starving for free space.

---

## 4. Firmware / platform configuration (BIOS–UEFI)

User intent includes **misconfiguration that leaves capacity unused** — e.g. XMP/DOCP/EXPO off so RAM runs below rated speed, or only half of installed memory visible.

### Memory population and speed (Linux)

```bash
sudo dmidecode -t memory 2>/dev/null
# Compact view:
sudo dmidecode -t memory 2>/dev/null | egrep -i 'Size:|Speed:|Configured|Manufacturer|Locator:|Form Factor|Type:|Maximum Capacity|Number Of Devices'
# Kernel view of what is actually online
free -h
lsmem 2>/dev/null
grep MemTotal /proc/meminfo
# SPD vs configured (when available)
sudo dmidecode -t memory 2>/dev/null | egrep -i 'Speed:|Configured Memory Speed|Configured Clock Speed'
```

| Signal | Interpretation |
|--------|----------------|
| Sum of populated DIMM **Size** ≫ OS **MemTotal** | Firmware map issue, integrated GPU carve-out (small delta OK), or hardware fault — **warn** if multi-GB missing |
| Empty slots while user expects upgrade headroom | **info** — “avenues to improve” not failure |
| `Speed:` (rated) e.g. 5600 MT/s but `Configured Memory Speed` 2133/2400/4800 lower | **XMP/DOCP/EXPO (or AMD EXPO / Intel XMP) likely disabled** — **watch/warn** for performance |
| Only one of two DIMMs detected | seating/fault — **warn** |
| ECC CE/UE counts rising (server) | **warn/crit** |

How to talk about profile names:

| Vendor ecosystem | Marketing name |
|------------------|----------------|
| Intel Z-series etc. | **XMP** (Extreme Memory Profile) |
| AMD (many boards) | **EXPO** or **DOCP** (Direct Over Clock Profile; ASUS-style) |
| JEDEC default | Safe baseline — often slower than kit rating |

**Never** change BIOS from this skill. Report: enter firmware setup, enable the memory profile, save/reboot, re-measure `Configured Memory Speed`.

### Memory (Windows)

```powershell
Get-CimInstance Win32_PhysicalMemory | Select-Object BankLabel, Manufacturer, PartNumber, `
  @{N='GB';E={$_.Capacity/1GB}}, Speed, ConfiguredClockSpeed, DeviceLocator
Get-CimInstance Win32_ComputerSystem | Select-Object `
  @{N='TotalPhysicalGB';E={[math]::Round($_.TotalPhysicalMemory/1GB,2)}}, `
  NumberOfProcessors, NumberOfLogicalProcessors
Get-CimInstance Win32_PhysicalMemoryArray | Select-Object `
  @{N='MaxGB';E={[math]::Round($_.MaxCapacity/1MB,2)}}, MemoryDevices
```

| Signal | Same as Linux |
|--------|----------------|
| `Speed` vs `ConfiguredClockSpeed` | Profile disabled if configured ≪ rated |
| Sum(Capacity) vs TotalPhysicalMemory | Missing RAM |
| MaxCapacity headroom | upgrade avenue |

### Memory (macOS)

```bash
sysctl hw.memsize
system_profiler SPMemoryDataType 2>/dev/null
```

Apple Silicon: RAM is package-on-package; no XMP. Report total and pressure only. Intel Macs with SODIMMs: compare profiler size/speed to expectation.

### Other firmware / platform signals

```bash
# Linux: virt capability (for developers)
egrep -c '(vmx|svm)' /proc/cpuinfo
# IOMMU / secure boot (informational)
bootctl status 2>/dev/null | head -n 40
mokutil --sb-state 2>/dev/null
# GPU BAR / resizable BAR — hard to assert from userspace; skip unless tooling present
# Laptop battery
upower -i $(upower -e 2>/dev/null | grep BAT) 2>/dev/null
cat /sys/class/power_supply/BAT*/{capacity,status,technology,cycle_count,energy_full,energy_full_design} 2>/dev/null
```

| Signal | Severity / note |
|--------|-----------------|
| Battery `energy_full` / `energy_full_design` &lt; 0.8 | **watch**; &lt; 0.6 **warn** — wear |
| Virtualization bits off when user needs VMs/KVM | **watch** — enable SVM/VT-x in firmware |
| Dual-channel expected but single DIMM | performance **watch** |
| PCIe ASPM / power issues | advanced; only if dmesg noise |

### Windows firmware extras

```powershell
Get-CimInstance Win32_BIOS | Select-Object Manufacturer, SMBIOSBIOSVersion, ReleaseDate
Get-CimInstance Win32_Processor | Select-Object Name, MaxClockSpeed, CurrentClockSpeed, NumberOfCores, NumberOfLogicalProcessors
# Battery
Get-CimInstance Win32_Battery | Select-Object EstimatedChargeRemaining, BatteryStatus, DesignCapacity, FullChargeCapacity
```

| Signal | Note |
|--------|------|
| CurrentClockSpeed ≪ MaxClockSpeed at idle | normal power saving; if under load + thermal, throttle |
| FullChargeCapacity / DesignCapacity low | battery wear |
| Very old BIOS date on new CPU issues | **info** — check vendor advisories |

### macOS battery

```bash
pmset -g batt
system_profiler SPPowerDataType 2>/dev/null | egrep -i 'Cycle|Condition|Maximum Capacity|Charge'
```

`Condition: Service Battery` → **warn**.

**Remediation:** firmware findings are **configuration/hardware** avenues, not prune targets. List them under **Platform recommendations**.

---

## 5. CPU load, I/O wait, thermal throttling

### Linux

```bash
nproc
uptime
# load vs cores: load1 / nproc
lscpu | egrep 'Model name|CPU\(s\)|Thread|Core|MHz|max MHz'
# Snapshot
top -b -n1 | head -n 20
# iowait and run queue
vmstat 1 5
# Per-disk
iostat -xz 1 3 2>/dev/null
# Thermal
sensors 2>/dev/null
cat /sys/class/thermal/thermal_zone*/temp 2>/dev/null
# Throttling (Intel)
grep -i thrott /sys/devices/system/cpu/cpu*/thermal_throttle/* 2>/dev/null
journalctl -k --no-pager 2>/dev/null | egrep -i 'thermal|throttl|mce|Machine check' | tail -n 20
```

| Signal | Severity |
|--------|----------|
| load1 sustained &gt; 1.0 × nproc | **watch**; ≫ 2× **warn** |
| `%wa` (iowait) high while load high | I/O bound — disk/NFS, not CPU |
| Thermal throttle counters increasing | **warn** |
| MCE / machine check | **crit** hardware |

Load average on Linux includes uninterruptible I/O waiters — high load + low CPU% ⇒ investigate disk/network storage.

### macOS

```bash
sysctl hw.ncpu hw.physicalcpu
uptime
top -l 2 -n 0 | head -n 20
pmset -g thermlog 2>/dev/null | tail -n 20
# powermetrics needs root — only if approved
```

### Windows

```powershell
Get-CimInstance Win32_Processor | Select-Object LoadPercentage, Name, NumberOfCores
Get-Counter '\Processor(_Total)\% Processor Time','\PhysicalDisk(_Total)\% Disk Time' -SampleInterval 1 -MaxSamples 3
```

| Signal | Severity |
|--------|----------|
| Sustained CPU 95–100% with no intentional workload | **watch** — find process |
| Disk time 100% + low free space | disk domain |

---

## 6. Services, packages, time, pending reboot

### Linux (systemd)

```bash
systemctl is-system-running 2>/dev/null          # running | degraded | maintenance
systemctl --failed --no-pager 2>/dev/null
# Disk-full often breaks services
systemctl list-units --state=failed --no-pager
# Package health
# Debian:
dpkg --audit 2>/dev/null
# Fedora:
sudo dnf check 2>/dev/null
# Time
timedatectl 2>/dev/null
# Pending reboot (Ubuntu)
test -f /var/run/reboot-required && cat /var/run/reboot-required.pkgs 2>/dev/null
# Needs restart (Debian needrestart)
command -v needrestart >/dev/null && sudo needrestart -b 2>/dev/null | head
```

| Signal | Severity |
|--------|----------|
| `is-system-running` = **degraded** | **warn** |
| Multiple failed units | **warn** |
| `dpkg --audit` broken packages | **warn** |
| NTP not synchronized | **watch** (certs, logs, make) |
| reboot-required after security updates | **watch** |

### macOS

```bash
# Launchd failed jobs harder; check system logs if user reports issues
log show --predicate 'eventMessage contains "error"' --last 1h 2>/dev/null | tail -n 5
sntp -d time.apple.com 2>/dev/null | head -n 5
```

Prefer softwareupdate inventory only if user wants update hygiene:

```bash
softwareupdate -l 2>/dev/null | head -n 40
```

### Windows

```powershell
# Failed services
Get-Service | Where-Object { $_.Status -ne 'Running' -and $_.StartType -eq 'Automatic' } |
  Select-Object Name, Status, StartType
# Component store corruption (expensive — deep only)
# DISM /Online /Cleanup-Image /CheckHealth
# Pending reboot
Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending'
Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired'
# Time
w32tm /query /status 2>$null
```

| Signal | Severity |
|--------|----------|
| Auto services stopped (especially bits, wuauserv when updating) | context-dependent |
| Reboot pending long term | **watch** |
| DISM CheckHealth corrupt | **warn** |

---

## 7. Containers, VMs, developer disk hogs (health angle)

```bash
docker system df 2>/dev/null
# Warn if reclaimable build cache + unused images is large fraction of disk
df -h $(docker info -f '{{.DockerRootDir}}' 2>/dev/null) 2>/dev/null
```

| Signal | Severity |
|--------|----------|
| Docker root filesystem ≥ 90% | **warn** — prune avenues |
| WSL VHD large + host disk low | **warn** both layers |
| Multipass/libvirt images filling home | **watch** |

Connect directly to cleanup Phase 2 avenues.

---

## 8. Network (lightweight)

Only quick checks — not a full network skill.

```bash
# Linux/macOS
ip route 2>/dev/null || route -n 2>/dev/null
ping -c 1 -W 2 1.1.1.1 2>/dev/null || ping -c 1 1.1.1.1 2>/dev/null
getent hosts example.com 2>/dev/null || dscacheutil -q host -a name example.com 2>/dev/null
```

```powershell
Get-NetIPConfiguration | Select-Object InterfaceAlias, IPv4Address, IPv4DefaultGateway, DNSServer
Test-Connection -ComputerName 1.1.1.1 -Count 1 -Quiet
```

| Signal | Severity |
|--------|----------|
| No default route | **warn** if user expects connectivity |
| Ping OK, DNS fail | **warn** DNS |
| Offline intentional | **ok** if known |

---

## 9. Security / update hygiene (informational)

Report, do not “harden” without request.

| Check | Command hints | Note |
|-------|---------------|------|
| Pending updates many months | `apt list --upgradable`, `dnf check-update`, `softwareupdate -l`, `winget upgrade` | watch |
| Firewall off | `ufw status`, `firewall-cmd`, Windows Firewall profile | info |
| Disk encryption | `lsblk -o NAME,FSTYPE`, BitLocker status, FileVault `fdesetup status` | policy context |
| Backup agent failed | Time Machine `tmutil`, VSS, restic timers | warn if found failed |

---

## 10. Cross-check recipes (common unhealthy stories)

| Story | Signals that co-occur | Likely direction |
|-------|----------------------|------------------|
| “System feels slow” | load ≫ cores **or** high iowait **or** memory pressure + swap | CPU vs disk vs RAM — do not guess from load alone |
| “No space left” but `df` shows free | **inode** full; or quota; or wrong mount; Docker disk-quota | `df -i`, `df` on docker root |
| “Only 8 GB of 16 GB RAM” | dmidecode one DIMM empty/faulty; UMA GPU carve small | firmware/hardware |
| “RAM slower than advertised” | ConfiguredClockSpeed &lt; SPD Speed | enable XMP/DOCP/EXPO |
| “Disk full after Docker week” | `docker system df` large; `/var/lib/docker` | prune |
| “Random kills / freezes” | OOM logs; SMART pending; thermal | memory or disk integrity |
| “Windows Update fails” | C: &lt; 10 GB free; component store issues | free space + DISM |
| “Laptop dies fast” | battery capacity ratio low; high discharge apps | battery + software |
| “Degraded” boot message | `systemctl --failed` | fix units / disk space |

---

## Probe budget (what to always run)

**Always (all OS):** capacity (`df` / volumes), memory summary, load/uptime or CPU snapshot, top few disk/memory consumers if stressed.

**When tools/privilege exist:** SMART health, dmidecode/Win32_PhysicalMemory speed vs configured, systemd failed units / Windows auto services, Docker df, battery (laptops).

**Deep / on suspicion only:** full SMART attributes, DISM CheckHealth, powermetrics, long Event Log digs, RAID detail.

Skip probes that hang (flaky NFS `df`, sleeping disks) — set timeouts when possible (`timeout 10 df -h`).

---

## Output fragment (embed in defrag report)

```markdown
### Health assessment
**Overall:** healthy | degraded | unhealthy

| Domain | Status | Evidence | Follow-up |
|--------|--------|----------|-----------|
| Disk capacity | warn | `/` 94% used (12G free) | prioritize cache/docker prune |
| Disk integrity | ok | SMART PASSED nvme0n1 | — |
| Memory | ok | 18/32 GiB available | — |
| Firmware / RAM | watch | Configured 4800 MT/s, rated 6000; enable EXPO/XMP in UEFI | user BIOS |
| CPU / load | ok | load1 0.4 on 16 threads | — |
| Services | warn | 2 failed units: … | investigate |
| Battery | watch | 72% of design capacity | — |

### Platform recommendations (non-prune)
- Enable memory XMP/EXPO profile so RAM runs at rated speed.
- …
```

---

## What health checks must not do

- Do not flash BIOS/UEFI or change firmware settings.
- Do not force offline `chkdsk` / `fsck` without explicit approval and backup awareness.
- Do not start long SMART tests (`-t long`) without approval (I/O load).
- Do not disable security features to “improve health scores.”
- Do not treat a single yellow metric as catastrophe — correlate.
