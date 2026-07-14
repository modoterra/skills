# Linux cleanup avenues

Use after fingerprinting via `/etc/os-release` (`ID`, `ID_LIKE`, `VERSION_ID`) and `uname -r`.

**Health first:** run capacity, SMART, memory, load, firmware/RAM, and services probes from [`health.md`](health.md) before pruning. Cleanup priority should follow **warn/crit** capacity findings.

## Quick disk picture

```bash
df -hT
lsblk -f 2>/dev/null
# Largest top-level dirs (needs root for full /var accuracy)
sudo du -xhd1 / 2>/dev/null | sort -h | tail -n 20
du -xhd1 "$HOME" 2>/dev/null | sort -h | tail -n 20
```

Focus mounts that are actually full (`Use%` high on `/`, `/home`, `/var`, Docker data roots).

---

## Package managers by family

### Debian / Ubuntu / Pop!_OS / Mint / elementary (`ID` or `ID_LIKE` = debian, ubuntu)

| Avenue | Measure | Prune | Risk |
|--------|---------|-------|------|
| APT download cache | `du -sh /var/cache/apt/archives` | `sudo apt-get clean` | low |
| Unused deps | `apt-get -s autoremove` | `sudo apt-get autoremove --purge` | medium |
| Obsolete packages | `apt list --installed` + manual | selective `remove` | medium |
| Old kernels | `dpkg -l 'linux-image-*'` vs `uname -r` | `sudo apt autoremove --purge` (keeps current+one often) | **high** if manual purge wrong image |
| Snap | `snap list --all`; old revisions | `sudo snap list --all` then remove disabled: `sudo snap remove NAME --revision=N` | medium |
| Flatpak | `flatpak list`; `du -sh ~/.local/share/flatpak /var/lib/flatpak` | `flatpak uninstall --unused`; `flatpak uninstall --delete-data` only if OK | medium |

Ubuntu-specific notes:

- **16.04–18.04:** `apt-get` dominant; snap less central.
- **20.04+:** snap common for Chromium/Firefox; many disabled snap revisions accumulate.
- **22.04 / 24.04:** same; also check `needrestart`, leftover `linux-modules-*`.
- Prefer `apt` modern CLI when available; `apt-get clean/autoremove` remains reliable in scripts.

```bash
sudo apt-get update -qq
sudo apt-get -s autoremove   # dry simulation
sudo apt-get clean
sudo apt-get autoremove --purge -y   # only after approval
```

### Fedora / RHEL / CentOS Stream / Rocky / Alma (`fedora`, `rhel`, `centos`)

| Avenue | Measure | Prune | Risk |
|--------|---------|-------|------|
| DNF cache | `du -sh /var/cache/dnf` | `sudo dnf clean all` | low |
| Old kernels | `rpm -q kernel-core` / `dnf list --installed kernel*` | `sudo dnf remove <old>` keeping current | **high** |
| Unused deps | | `sudo dnf autoremove` | medium |
| Flatpak | same as above | | medium |
| Toolbox / Distrobox | `toolbox list` / `distrobox list` | remove unused containers only with OK | medium |

```bash
sudo dnf clean all
sudo dnf autoremove -y    # after approval
# Keep at least running kernel
uname -r
```

**Fedora Silverblue / Kinoite / CoreOS / atomic:** host is ostree. Use:

```bash
rpm-ostree status
rpm-ostree cleanup -bm     # after approval; leaves deployments managed
flatpak uninstall --unused
```

Do not treat layered packages like classic dnf on mutable root without reading `rpm-ostree` docs.

### Arch / Manjaro / Endeavour (`arch`, `manjaro`)

| Avenue | Measure | Prune | Risk |
|--------|---------|-------|------|
| Pacman cache | `du -sh /var/cache/pacman/pkg` | `sudo paccache -d` (dry) / `sudo paccache -r` keep last 3 | low–medium |
| Orphans | `pacman -Qtdq` | `sudo pacman -Rns $(pacman -Qtdq)` | medium |
| Yay/paru AUR cache | `du -sh ~/.cache/yay ~/.cache/paru` | tool-specific clean | low |
| Flatpak | same | | |

```bash
sudo pacman -Sc            # interactive; prefer paccache
command -v paccache && paccache -d && sudo paccache -r
```

### openSUSE Leap / Tumbleweed (`opensuse*`)

```bash
sudo zypper clean --all
sudo zypper packages --unneeded
# Atomic / MicroOS: transactional-update / reboot model — do not classic-remove casually
```

### Alpine (`alpine`)

```bash
du -sh /var/cache/apk
sudo apk cache clean
# Minimal system — little to reclaim beyond caches and docker
```

### Gentoo

```bash
# Distfiles and binpkgs can be huge
du -sh /var/cache/distfiles /var/cache/binpkgs
sudo eclean-dist -d
sudo eclean-pkg -d
```

### Nix / NixOS

```bash
nix-store --gc
nix-collect-garbage -d     # removes old generations — confirm
# NixOS: also nixos-rebuild generations
```

### Guix

```bash
guix gc
```

---

## Logs, journals, crashes

### systemd journal (most modern distros)

```bash
journalctl --disk-usage
# Vacuum by size or time (medium risk — loses old logs)
sudo journalctl --vacuum-size=200M
# or
sudo journalctl --vacuum-time=14d
```

### Classic logs

```bash
sudo du -sh /var/log
# Rotated logs: prefer logrotate; do not truncate active syslog blindly
sudo find /var/log -type f -name '*.gz' -mtime +30   # list candidates
# Core dumps
coredumpctl 2>/dev/null
sudo du -sh /var/lib/systemd/coredump /var/crash 2>/dev/null
```

Ubuntu `whoopsie` / crash reports: `/var/crash` — remove old `.crash` with care.

---

## Temporary and trash

```bash
du -sh /tmp /var/tmp "$HOME/.cache" "$HOME/.local/share/Trash" 2>/dev/null
# User trash
command -v trash-empty && trash-empty -v   # if trash-cli installed; else GUI empty
# Only delete /tmp files not in use; prefer reboot-aged files
```

Do not wipe `/tmp` wholesale on multi-user production hosts.

---

## Containers and VMs (Linux host)

### Docker

```bash
docker system df
docker system df -v
# Safe-ish: unused containers, networks, dangling images, build cache
docker system prune -f
# Broader images (medium)
docker image prune -a -f    # after approval — removes unused images
# Build cache only
docker builder prune -af
# Volumes (HIGH) — never without listing
docker volume ls
docker volume prune -f      # only unused; still confirm
```

Custom data-root: check `/etc/docker/daemon.json` → `data-root` (often `/var/lib/docker`).

### Podman / Buildah

```bash
podman system df
podman system prune -af     # after approval
podman volume prune
```

### containerd / nerdctl / k3s / microk8s

```bash
# k3s images
sudo k3s crictl images
sudo k3s crictl rmi --prune 2>/dev/null
# microk8s
microk8s ctr images ls
```

### LXC / LXD / Incus

```bash
lxc list 2>/dev/null || incus list 2>/dev/null
# Delete stopped unused instances only with OK
```

### libvirt / QEMU

```bash
virsh list --all
du -sh /var/lib/libvirt/images "$HOME/.local/share/libvirt/images" 2>/dev/null
```

---

## Desktop / user caches (GUI Linux)

| Path | Notes |
|------|-------|
| `~/.cache/` | Thumbnails, pip, mesa shaders, browsers subdirs — prune selectively |
| `~/.cache/thumbnails` | low risk |
| `~/.local/share/Trash` | empty trash |
| `~/.var/app/` | Flatpak app data — do not bulk delete |
| `~/.config/` | **not** a cache — leave alone |
| Browser caches under `~/.cache/mozilla`, `~/.cache/google-chrome`, etc. | low–medium; closes space but re-download |

KDE / GNOME:

```bash
du -sh ~/.cache/mesa_shader_cache ~/.cache/thumbnails 2>/dev/null
```

---

## Snaps and Flatpak deep clean

```bash
# Old snap revisions
snap list --all | awk '/disabled/{print $1, $3}'
# For each: sudo snap remove <name> --revision=<rev>

flatpak uninstall --unused -y
# Repair refs if needed
flatpak repair --user
sudo flatpak repair
```

---

## Old kernels (high impact)

Always keep the **running** kernel (`uname -r`) and preferably one previous.

```bash
uname -r
# Debian-family
dpkg -l 'linux-image-*' 'linux-headers-*' | grep ^ii
# Fedora-family
rpm -q kernel-core
```

Prefer `autoremove` over hand-crafted purge lists. Reboot after removals when package manager advises.

---

## WSL guest specifics

When `/proc/version` mentions Microsoft/WSL:

- Clean **inside** the distro with this file’s avenues.
- Free space may not return to Windows until VHDX compact — see `references/windows.md` WSL section.
- Avoid filling `$HOME` with Docker-in-WSL without checking `.wslconfig` and docker data-root.

---

## Immutable / gaming / appliance distros

| Distro | Prefer |
|--------|--------|
| Fedora Atomic | `rpm-ostree cleanup`, flatpak, toolbox |
| SteamOS / Steam Deck | Discover mode; limited host writes; flatpak + proton caches under home |
| ChromeOS Linux (Crostini) | Clean inside the container; host storage is managed by ChromeOS |
| OpenWrt / appliance | Rarely run this skill; space is flash — be extremely conservative |

---

## Priority order on a full Linux root

1. `docker system df` / container data if Docker is large  
2. APT/DNF/pacman caches  
3. journal vacuum  
4. Flatpak/snap unused  
5. `~/.cache` heavy hitters  
6. Old kernels / deep autoremove (confirm)  
7. Named volumes and VMs (confirm)
