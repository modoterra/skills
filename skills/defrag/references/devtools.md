# Cross-platform developer tool caches

Probe with `command -v` / `Get-Command` before cleaning. Only touch tools that exist. Prefer official cache commands over deleting directories by hand.

Sizes: run the **list/path** command first, then prune after approval when medium risk or higher.

---

## Containers (all OSes with Docker CLI)

```bash
docker system df
docker system prune -f                  # dangling + stopped — low–medium
docker builder prune -af                # build cache — medium
docker image prune -a -f                # unused images — medium; confirm
docker volume ls
# docker volume prune -f                # HIGH — unused volumes only; still confirm
```

Podman: same shape with `podman` instead of `docker`.

---

## JavaScript / Node

| Tool | Path / inspect | Prune |
|------|----------------|-------|
| npm | `npm cache ls` / `npm config get cache` | `npm cache clean --force` |
| pnpm | `pnpm store path` | `pnpm store prune` |
| yarn classic | `yarn cache dir` | `yarn cache clean` |
| yarn berry | `.yarn/cache` in project / global | follow Yarn 2+ docs; project-local |
| bun | `bun pm cache` | `bun pm cache rm` |
| nvm | `~/.nvm/versions` | remove old Node versions user does not need |
| fnm | `fnm list` | `fnm uninstall <ver>` |
| n | versions under prefix | selective |
| corepack | rarely large | leave unless broken |

**Project `node_modules`:** not global cleanup — only remove in abandoned projects with user OK.

```bash
du -sh ~/.npm "$(pnpm store path 2>/dev/null)" ~/.yarn/cache ~/.bun/install/cache 2>/dev/null
```

---

## Python

| Tool | Inspect | Prune |
|------|---------|-------|
| pip | `pip cache dir` | `pip cache purge` |
| pip3 | same | `pip3 cache purge` |
| uv | `uv cache dir` | `uv cache clean` |
| poetry | `poetry cache list` | `poetry cache clear --all pypi` (confirm) |
| conda/mamba | `conda clean -a -d` dry-run | `conda clean -a -y` after approval |
| pyenv | `~/.pyenv/versions` | uninstall old Pythons |
| ruff / pre-commit | `~/.cache/pre-commit`, ruff cache | selective delete |

```bash
pip cache dir 2>/dev/null; pip cache info 2>/dev/null
du -sh ~/.cache/pip ~/Library/Caches/pip 2>/dev/null
```

---

## Rust

```bash
rustup show
# Toolchains
# rustup toolchain list
# rustup toolchain uninstall <unused>
du -sh ~/.cargo/registry ~/.cargo/git target 2>/dev/null
cargo cache -e 2>/dev/null || true   # if cargo-cache installed
# Without cargo-cache: cleaning registry is medium risk (redownload)
```

`target/` directories inside projects: largest wins often live in workspaces — clean per-project with `cargo clean` only when user wants rebuild cost.

---

## Go

```bash
go env GOPATH GOMODCACHE
du -sh "$(go env GOMODCACHE)" "$(go env GOPATH)/pkg" 2>/dev/null
go clean -modcache     # after approval — redownload modules
go clean -cache
```

---

## Java / JVM

```bash
du -sh ~/.m2/repository ~/.gradle/caches 2>/dev/null
# Gradle
rm -rf ~/.gradle/caches/build-cache-*   # optional partial; or:
# ./gradlew --stop; clear caches with user OK
# Maven: delete ~/.m2/repository only if willing to redownload everything
```

SDKMAN:

```bash
command -v sdk && sdk list java
# sdk uninstall java <ver>
```

---

## PHP / Composer

```bash
composer clear-cache 2>/dev/null
du -sh ~/.composer/cache 2>/dev/null
```

---

## Ruby

```bash
gem sources
du -sh ~/.gem 2>/dev/null
# rbenv/rvm versions
rbenv versions 2>/dev/null
# gem cleanup
gem cleanup
```

---

## .NET

```bash
dotnet nuget locals all --list
dotnet nuget locals all --clear
# HTTP cache only:
dotnet nuget locals http-cache --clear
```

---

## Mobile / cross

| Stack | Paths / commands |
|-------|------------------|
| Android SDK | `$ANDROID_HOME` / `~/Library/Android/sdk` — remove unused system images via sdkmanager |
| Gradle (Android) | `~/.gradle/caches` |
| CocoaPods | `pod cache clean --all` |
| Flutter | `flutter pub cache clean`; old SDKs under install dir |
| Electron | electron download caches under `~/.cache/electron` / `%LOCALAPPDATA%` |

```bash
du -sh ~/.android/cache ~/Library/Android/sdk/system-images 2>/dev/null
du -sh ~/.cache/electron 2>/dev/null
```

---

## ML / data science caches (can be huge)

| Path | Notes |
|------|-------|
| `~/.cache/huggingface` | models — **confirm** before delete |
| `~/.cache/torch` | |
| `~/.cache/whisper` | |
| `~/.cache/pip` | already under Python |
| Docker volumes for Jupyter | treat as volumes — high risk |

```bash
du -sh ~/.cache/huggingface ~/.cache/torch 2>/dev/null
```

---

## Editors and LSP

| Tool | Cache-ish paths |
|------|-----------------|
| VS Code / Cursor | Cached extensions rarely worth deleting; `CachedData` under user data dir |
| JetBrains | `~/Library/Caches/JetBrains` or `%LOCALAPPDATA%\JetBrains` — invalidate via IDE |
| ccache / sccache | `ccache -C` / sccache zero stats + clear |

Do not delete IDE **config** directories as cleanup.

---

## Git

```bash
# Optional maintenance on huge repos (user project only)
git -C PATH maintenance run
git -C PATH gc --aggressive   # expensive; optional
# Global: stale credentials not a size issue
```

Untracked build artifacts: respect `.gitignore`; do not `git clean -fdx` without explicit OK.

---

## Multipass / Vagrant / VirtualBox / QEMU user images

```bash
multipass list 2>/dev/null
vagrant global-status 2>/dev/null
VBoxManage list hdds 2>/dev/null
du -sh ~/.vagrant.d ~/VirtualBox\ VMs 2>/dev/null
```

Delete VMs only with named approval.

---

## Suggested order (dev machine)

1. Container builder + dangling prune  
2. Language package caches (npm/pip/cargo/go/nuget)  
3. Version managers’ abandoned runtimes  
4. Gradle/Maven/CocoaPods  
5. IDE caches if still tight  
6. ML model caches and VM disks last (confirm)  

---

## Risk summary

| Action | Risk |
|--------|------|
| `npm cache clean`, `pip cache purge`, `go clean -cache` | low |
| `go clean -modcache`, `docker image prune -a`, `cargo` registry wipe | medium |
| Docker **volumes**, conda envs in use, HF models, VM disks | **high** |
| `git clean -fdx`, deleting project `node_modules` without scope | **high** |
