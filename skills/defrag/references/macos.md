# macOS cleanup avenues

**Health first:** capacity, memory pressure, storage SMART (when available), battery, and load — see [`health.md`](health.md).

Fingerprint first:

```bash
sw_vers
uname -m
df -H /
```

Branch on `ProductVersion` major. Commands below work across modern macOS; version notes call out differences.

| Major | Marketing |
|-------|-----------|
| 12 | Monterey |
| 13 | Ventura |
| 14 | Sonoma |
| 15 | Sequoia |
| 16+ | Follow system naming; prefer same tools |

SIP (`csrutil status`) protects `/System`. Never attempt to “clean” SIP paths.

---

## Storage overview

```bash
df -H /
# User library bulk (depth-limited)
du -hd1 ~/Library 2>/dev/null | sort -h | tail -n 30
du -hd1 ~/Library/Caches 2>/dev/null | sort -h | tail -n 20
```

GUI: **Apple menu → System Settings → General → Storage** (Ventura+) or **About This Mac → Storage** (older). Prefer native tools for Photos/iOS backups when sizes are unclear.

Purgeable space is not always freeable via CLI; macOS may reclaim automatically when needed.

---

## Homebrew (most common third-party manager)

Detect: `command -v brew` and `brew --prefix` (`/opt/homebrew` on Apple Silicon, `/usr/local` on Intel).

```bash
brew --prefix
du -sh "$(brew --cache)" 2>/dev/null
brew autoremove -n          # dry-run unused deps
brew cleanup -n             # dry-run
brew cleanup -s             # clear cache (after approval)
brew autoremove             # after approval
```

| Avenue | Risk |
|--------|------|
| `brew cleanup -s` | low |
| `brew autoremove` | medium |
| Uninstall formulae/casks | medium — only if user lists them |

Also:

```bash
# Old downloads
du -sh ~/Library/Caches/Homebrew 2>/dev/null
```

---

## Xcode and Apple developer tools

Only if Xcode or CLT present (`xcode-select -p`).

```bash
# DerivedData — often multi-GB
du -sh ~/Library/Developer/Xcode/DerivedData 2>/dev/null
# Archives
du -sh ~/Library/Developer/Xcode/Archives 2>/dev/null
# Device support symbols
du -sh ~/Library/Developer/Xcode/iOS\ DeviceSupport 2>/dev/null
du -sh ~/Library/Developer/Xcode/watchOS\ DeviceSupport 2>/dev/null
# Simulators
xcrun simctl list 2>/dev/null
du -sh ~/Library/Developer/CoreSimulator 2>/dev/null
```

Prune (after approval):

```bash
# DerivedData (low–medium; rebuild cost)
rm -rf ~/Library/Developer/Xcode/DerivedData/*
# Unavailable simulators
xcrun simctl delete unavailable
# Specific runtime (high impact for iOS work)
xcrun simctl runtime list
# xcrun simctl runtime delete <id>   # confirm first
```

Archives and DeviceSupport: delete only after user confirms they do not need old device symbols or App Store archives.

Command Line Tools leftover after Xcode install: do not remove CLT casually (`xcode-select --install` needed later).

---

## Time Machine local snapshots

Can consume large space on laptops:

```bash
tmutil listlocalsnapshots / 2>/dev/null
# Thin local snapshots (medium–high; reduces local restore points)
# sudo tmutil thinlocalsnapshots / <bytes> 4
```

Only with approval. Do not disable Time Machine backups without request.

---

## System and user caches

| Path | Risk | Notes |
|------|------|-------|
| `~/Library/Caches` | low–medium | App-specific; do not wipe entire folder blindly — target large subdirs |
| `/Library/Caches` | medium | May need sudo; leave Apple system caches if unsure |
| `~/Library/Logs` | low | Old logs |
| `/private/var/log` | medium | Prefer unified logging; do not gut system logs on production Macs |
| `~/Library/Logs/DiagnosticReports` | low | Crash reports |

```bash
du -sh ~/Library/Caches/* 2>/dev/null | sort -h | tail -n 25
```

### Mail, Messages, iOS backups

```bash
du -sh ~/Library/Messages 2>/dev/null
du -sh ~/Library/Mail 2>/dev/null
du -sh ~/Library/Application\ Support/MobileSync/Backup 2>/dev/null
```

iOS device backups are **high value** — list sizes; delete only with explicit OK.

---

## Docker Desktop / Colima / OrbStack / Podman on Mac

```bash
docker system df 2>/dev/null
# Docker Desktop: disk image can grow large; prune inside engine first
docker system prune -f
docker builder prune -af    # after approval
```

Docker Desktop settings UI can reclaim disk; CLI prune does not always shrink the raw disk image fully until Docker Desktop “Clean / Purge data” or compact.

Colima:

```bash
colima status 2>/dev/null
# colima delete only if user abandons the VM
```

OrbStack: use its disk cleanup UI/CLI if present; treat VMs as high risk.

---

## Mac App Store / system updates leftovers

```bash
# Software Update catalogs rarely need manual clean
# macOS installer apps left in /Applications (Install macOS *.app) — large; remove after upgrade if user OK
ls -d /Applications/Install\ macOS*.app 2>/dev/null
```

---

## Language toolchains on Mac

Same as `devtools.md`, with Mac path notes:

| Tool | Common cache paths |
|------|--------------------|
| npm/pnpm/yarn | `~/.npm`, `~/Library/Caches/pnpm`, `~/.yarn/cache` |
| CocoaPods | `~/Library/Caches/CocoaPods` |
| Gradle | `~/.gradle/caches` |
| Composer | `~/.composer/cache` |
| pip | `~/Library/Caches/pip` |
| Hugging Face / ML | `~/.cache/huggingface` |

```bash
du -sh ~/Library/Caches/CocoaPods ~/.npm ~/.gradle/caches 2>/dev/null
pod cache clean --all 2>/dev/null   # after approval if CocoaPods used
```

---

## Rosetta, universal binaries, arch notes

- On **Apple Silicon**, Intel Homebrew under `/usr/local` may coexist with `/opt/homebrew` — clean each prefix separately.
- Do not remove Rosetta (`softwareupdate --install-rosetta`) as cleanup.

---

## Version-specific signals

### Monterey (12) / Ventura (13)

- Storage UI under System Settings evolves; CLI tools above remain valid.
- `tmutil` local snapshots still relevant on laptops.

### Sonoma (14) / Sequoia (15)+

- Continue using Homebrew, Xcode, Docker paths above.
- Staging and purgeable space more aggressive — re-check `df` after large deletes; OS may report free space with lag.
- Prefer not disabling new security features for “speed”.

---

## Safe default Mac sequence

1. `brew cleanup -n` → `brew cleanup -s` + `brew autoremove` if approved  
2. Large `~/Library/Caches/*` subdirs (named, not full wipe)  
3. Xcode DerivedData if developer machine  
4. `docker system prune` if Docker present  
5. Devtool caches (`devtools.md`)  
6. Simulators / TM snapshots / iOS backups only with explicit OK  

---

## Avoid

- Deleting `~/Library/Application Support` wholesale  
- Touching `/System`, `/usr` (except via package managers)  
- Third-party “Mac cleaner” installers  
- Emptying Trash without user awareness if they use it as holding area  
