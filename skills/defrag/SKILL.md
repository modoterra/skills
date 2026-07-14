---
name: defrag
description: >
  OS-agnostic system hygiene and health pass: discover the host OS, distro, and
  version; assess health signals (disk capacity/SMART, memory pressure, load,
  firmware/RAM config such as XMP-DOCP-EXPO, thermals, failed services); map
  cleanup avenues; measure reclaimable space; prune safely with approval. Named
  after classic defrag tools but does not defragment disks. Use for free disk
  space, clean up machine, system health check, why is my PC slow, prune caches,
  docker system prune, brew cleanup, apt clean, Storage Sense, or /defrag.
compatibility: >
  Designed for Agent Skills-compatible coding agents with shell access on the
  host being cleaned. Prefer non-destructive measurement first; escalate only
  with explicit user approval for destructive or privileged operations.
metadata:
  author: Modoterra
  version: "1.1.0"
---

# Defrag

Assess host **health**, then reclaim space and reduce clutter by **discovering where you run**, **reading platform health signals**, **enumerating cleanup avenues**, **measuring impact**, and **pruning with consent**.

This skill is **not** filesystem defragmentation. It is host hygiene **plus** situational awareness: capacity, integrity, memory, firmware misconfig (e.g. RAM not at rated speed), thermals, services, and reclaim paths (caches, logs, packages, containers, toolchains).

## Principles

1. **Discover before acting.** Identify OS family, version, distro/edition, package managers, runtimes, and containers. Never assume.
2. **Health before (or with) cleanup.** Capacity, SMART, memory, load, and firmware signals explain *why* the machine feels bad and which cleanups matter.
3. **Measure before pruning.** Report sizes and risk for each avenue. Prefer dry-run / list modes.
4. **Consent for destructive work.** Caches and package downloads are low risk; volumes, user documents, Time Machine, Windows.old, and system components need explicit approval.
5. **Least privilege.** Prefer user-scope probes and cleanups. Use `sudo` / Administrator only when required and approved.
6. **Idempotent and reversible where possible.** Prefer official package-manager and OS cleanup tools over ad-hoc `rm -rf`.
7. **Do not brick the machine.** Never delete the running kernel, active WSL distros, SIP-protected paths, or the only copy of user data.
8. **Do not flash firmware.** Report BIOS/UEFI recommendations; the user changes firmware settings.

## When to run

- User asks to free disk space, clean the machine, prune caches, or runs `/defrag`.
- User asks whether the system is healthy, why it is slow, or for a health check.
- Host is low on disk; builds or package installs fail with space errors.
- After heavy Docker / CI / Xcode / Visual Studio / package-manager use.

## Workflow

### Phase 0 — Scope

Clarify if not stated:

| Scope | Meaning |
|-------|---------|
| **health-only** | Fingerprint + health signals; no deletes |
| **quick** | Health summary + safe caches only |
| **standard** (default) | Health + quick cleanups + logs, unused packages, dangling container prune, dev caches |
| **deep** | Standard + old kernels/snapshots/component store/simulators — confirm each high-impact item |
| **report-only** | Discover, health, and measure reclaim; never delete |

Default to **standard** with a written plan before Phase 4. If the user only mentions slowness or health, default to **health-only** then offer cleanup for findings that are capacity-related.

### Phase 1 — Discover the host

Run a **platform fingerprint**. Collect facts into a short inventory; surface a summary to the user.

#### Universal probes

```bash
uname -srm 2>/dev/null || true
hostname 2>/dev/null || true
whoami 2>/dev/null || true
echo "HOME=$HOME USER=$USER SHELL=$SHELL"
df -h 2>/dev/null || df -H 2>/dev/null || true
```

#### Classify OS family

| Signal | Family |
|--------|--------|
| `uname -s` → `Linux` | Linux |
| `uname -s` → `Darwin` | macOS |
| `uname -s` → `MINGW*` / `MSYS*` / `CYGWIN*` or `$OS=Windows_NT` | Windows (via Git Bash / MSYS) |
| PowerShell `$IsWindows` / `Win32_OperatingSystem` | Windows native |
| WSL: `/proc/version` contains `Microsoft` or `WSL` | Linux **guest** on Windows — health and clean both layers carefully |

#### Linux fingerprint

```bash
cat /etc/os-release 2>/dev/null
cat /etc/lsb-release /etc/debian_version /etc/redhat-release /etc/alpine-release 2>/dev/null
ps -p 1 -o comm= 2>/dev/null
systemd-detect-virt 2>/dev/null
[ -f /proc/version ] && head -c 200 /proc/version
uname -r
nproc 2>/dev/null
free -h 2>/dev/null
```

Read [`references/linux.md`](references/linux.md) after identifying the family (`ID=`, `ID_LIKE=`, major version).

#### macOS fingerprint

```bash
sw_vers
uname -m
system_profiler SPSoftwareDataType 2>/dev/null | head -n 40
df -H /
command -v brew; brew --prefix 2>/dev/null
xcode-select -p 2>/dev/null
sysctl hw.memsize hw.ncpu 2>/dev/null
```

Read [`references/macos.md`](references/macos.md). Branch on major version from `ProductVersion`.

#### Windows fingerprint

```powershell
Get-CimInstance Win32_OperatingSystem |
  Select-Object Caption, Version, BuildNumber, OSArchitecture, FreePhysicalMemory, TotalVisibleMemorySize
Get-PSDrive -PSProvider FileSystem
Get-Command winget, choco, scoop, docker -ErrorAction SilentlyContinue
```

From Git Bash / WSL, prefer `powershell.exe -NoProfile -Command "..."`.

Read [`references/windows.md`](references/windows.md). Branch on build (Win10 19041+, Win11 22000+, Server).

#### Toolchain probe (all OSes)

```bash
command -v docker podman nerdctl containerd 2>/dev/null
command -v npm pnpm yarn bun 2>/dev/null
command -v pip pip3 uv poetry conda mamba 2>/dev/null
command -v cargo rustup go 2>/dev/null
command -v gem bundle php composer java mvn gradle 2>/dev/null
command -v flatpak snap brew 2>/dev/null
```

Read [`references/devtools.md`](references/devtools.md) when those tools exist.

### Phase 2 — Health assessment

**Always run** (even when the user only wants cleanup). Full signal catalog, thresholds, and commands: [`references/health.md`](references/health.md).

Assess domains and assign **ok / watch / warn / crit** per domain:

| Domain | What “unhealthy” often looks like |
|--------|-----------------------------------|
| **Disk capacity** | Free &lt; ~10–15% (or &lt; few GiB) on `/`, `C:\`, or Data volume; inode exhaustion |
| **Disk integrity** | SMART failing; pending/reallocated sectors; I/O errors; degraded RAID; volume HealthStatus ≠ Healthy |
| **Memory** | Low available RAM; heavy swap/pagefile; OOM kills; resource-exhaustion events |
| **CPU / load / I/O** | Load ≫ cores; high iowait; thermal throttling |
| **Firmware / platform** | OS sees far less RAM than installed DIMMs; RAM **configured** speed ≪ **rated** speed (XMP/DOCP/EXPO off); single-channel when dual expected; battery worn |
| **Thermal / power** | Throttle; battery service condition; not charging |
| **Services / software** | systemd `degraded`; failed units; broken packages; long-pending reboot |
| **Containers / virt** | Docker root full; WSL VHD bloated against full host disk |
| **Network** (light) | No default route / DNS when connectivity expected |
| **Update hygiene** (info) | Huge backlog of updates — report only unless asked to patch |

#### Minimum probe set (every run)

1. **Capacity** — `df` / `Get-Volume` / Data volume; on Linux also `df -i`.
2. **Memory** — `free -h` / `memory_pressure`+`vm_stat` / Win32 OS memory fields.
3. **Load** — `uptime` + core count, or Windows processor load sample.
4. **Top pressure sources** — if capacity or memory is warn+, sample largest dirs or top processes (depth-limited; do not scan the entire disk blindly).

#### Extended probes (when privilege/tools allow or symptoms warrant)

- SMART / `Get-PhysicalDisk` health  
- `dmidecode -t memory` or `Win32_PhysicalMemory` (**Speed** vs **ConfiguredClockSpeed**)  
- systemd `--failed` / automatic services not running  
- `docker system df`  
- Laptop battery capacity ratio  
- Kernel/storage errors in dmesg or Event Log (recent, filtered)  

#### Firmware and “unused capability” avenues

These are **not** prune targets; list them under **Platform recommendations**:

- Enable **XMP** (Intel), **EXPO** / **DOCP** (AMD/board vendors) so RAM runs at kit rating.  
- Reseat/investigate DIMMs if half of installed memory is missing.  
- Enable VT-x/AMD-V when VMs are needed but CPU flags are absent.  
- Battery replacement when design capacity ratio is poor.  
- Disk replacement when SMART is failing (backup first).  

Never change BIOS/UEFI from the agent.

#### Health gate

- If **disk capacity** is **crit**, prioritize cleanup plan before deep scans that write heavily.  
- If **disk integrity** is **crit**, say so first; do not pretend cache cleanup fixes a dying drive.  
- If **memory** is **crit** with OOM, identify hogs; cleanup only helps when disk-backed swap is starving for free space.

Produce the health table (see output template) before or alongside the reclaim table.

### Phase 3 — Map cleanup avenues and measure

Using the fingerprint **and** health findings, build a **reclaim table**:

| Avenue | Path / tool | Est. size | Risk | Command (dry-run if any) | Privilege |
|--------|-------------|-----------|------|--------------------------|-----------|
| … | … | … | low/med/high | … | user/root |

Prioritize avenues that address **warn/crit capacity** domains (Docker on full `/var`, apt archives on full `/`, etc.).

**Risk guide:**

| Risk | Examples | Approval |
|------|----------|----------|
| **low** | package download caches, thumbnail caches, empty trash (user asked), dangling docker images | May batch under standard after showing plan |
| **medium** | journal vacuum, unused packages, docker unused images, old snap revisions, brew cleanup | Explicit OK for the batch |
| **high** | named docker volumes, old kernels, Windows.old, local TM snapshots, Xcode simulators, DISM ResetBase, user project `node_modules` | Per-item confirmation |

Measurement helpers:

```bash
du -sh PATH 2>/dev/null
du -h -d 1 ~/.cache 2>/dev/null | sort -h | tail -n 20
```

Prefer first-class size commands in platform references over raw `du` when available.

Present health summary + reclaim table. **Stop for approval** before Phase 4 unless the user already authorized a scope (or **health-only** / **report-only**).

### Phase 4 — Prune

Execute approved avenues in order:

1. Low-risk user caches  
2. Package-manager caches  
3. Logs / journal / crash dumps  
4. Container / VM residue (dangling first; volumes last and only if approved)  
5. Dev-tool version managers only if approved  
6. Deep OS items only if approved  

After each major group, re-check free space and note reclaimed amount. Re-run a short capacity probe for the final report.

### Phase 5 — Report

Deliver:

1. **Environment** — OS, version, arch, notable tools  
2. **Health assessment** — overall + per-domain table with evidence  
3. **Platform recommendations** — firmware/RAM profile, hardware, non-prune fixes  
4. **Actions taken** — prune table and space freed (if any)  
5. **Skipped / declined**  
6. **Follow-ups** — reboot, BIOS visit, backup, SMART replacement, empty Recycle Bin UI  

## Hard prohibitions

Unless the user **explicitly and specifically** requests the item:

- Do not delete contents of `~/Documents`, `~/Desktop`, project source trees, or cloud-sync folders.
- Do not remove **named** Docker/Podman volumes, databases, or VM disks without naming them and getting OK.
- Do not run `rm -rf /`, or recursive deletes on `/usr`, `/System`, `/Windows`, or SIP paths.
- Do not disable security tools, firewalls, or backup agents as “cleanup”.
- Do not compact WSL VHDs or delete Windows.old without clear warnings.
- Do not wipe full browser profiles — cache-only if at all.
- Do not install third-party “cleaner” or “optimizer” GUI apps.
- Do not flash or reconfigure BIOS/UEFI; do not force offline `fsck`/`chkdsk` without approval.
- Do not start long SMART self-tests without approval.

## Privilege and safety patterns

| Situation | Behavior |
|-----------|----------|
| Command needs root and user has not approved elevation | Skip; list as “needs sudo” |
| Dry-run available | Show dry-run/list first for medium+ risk |
| Uncertain command for this OS version | Prefer official docs / `--help` over guessing |
| Multiple package managers | Clean each independently |
| WSL | Linux guest + Windows host layers; VHDX may not shrink until compacted |
| Immutable / atomic desktops | `rpm-ostree`, toolbox/distrobox, flatpak — not classic host dnf remove |
| SMART / dmidecode unavailable | Report “not probed”; do not invent healthy SMART |

## Output template

```markdown
## Defrag report

**Host:** <os> <version> (<arch>) — <distro/edition>
**Scope:** health-only | report-only | quick | standard | deep
**Disk before → after:** <free> → <free> on <mount>   # omit after if no prune

### Health assessment
**Overall:** healthy | degraded | unhealthy

| Domain | Status | Evidence | Follow-up |
|--------|--------|----------|-----------|
| Disk capacity | … | … | … |
| Disk integrity | … | … | … |
| Memory | … | … | … |
| CPU / load | … | … | … |
| Firmware / platform | … | … | … |
| Services | … | … | … |
| … | … | … | … |

### Platform recommendations (non-prune)
- …

### Cleanup avenues
| Avenue | Size | Risk | Status |
|--------|------|------|--------|
| … | … | low/med/high | done / skipped / needs approval |

### Notes
- …
```

## Reference map

After Phase 1 classification, load what applies:

| Topic | Reference |
|-------|-----------|
| Health signals, thresholds, firmware/RAM, SMART, load | [`references/health.md`](references/health.md) |
| Linux cleanup avenues | [`references/linux.md`](references/linux.md) |
| macOS cleanup avenues | [`references/macos.md`](references/macos.md) |
| Windows cleanup avenues | [`references/windows.md`](references/windows.md) |
| Dev tools / language managers | [`references/devtools.md`](references/devtools.md) |

Load **health.md** every run. Load OS cleanup + devtools references when planning or executing prune.
