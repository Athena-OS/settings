# Athena OS Settings

This repository provides a collection of configuration files and settings to optimize Athena OS. By taking inspiration from [CachyOS](https://github.com/CachyOS/CachyOS-Settings), these settings are designed to enhance system performance, responsiveness, resource management, and wireless compatibility, with a focus on the needs of security researchers, penetration testers, CTF players and students.

## Core System Optimizations

### ⚙️ Udev Rules: Device Event Automation

Udev rules automatically apply system configurations upon device detection or state changes.

* **Audio Power Management** (`20-audio-pm.rules`): Manages `snd-hda-intel` power saving to mitigate audio crackling. Disables power saving when on AC power and restores it when switching to battery. Stateful - saves and restores the original value across plug/unplug cycles.
* **ZRAM Swap Optimization** (`30-zram.rules`): When ZRAM finishes initializing, sets `vm.swappiness=150` to strongly prefer anonymous page compression and disables Zswap to prevent double compression and ensure accurate ZRAM accounting.
* **Device Permissions** (`40-hpet-permissions.rules`): Sets `rtc0` and `hpet` device group to `audio` for proper access by timing-sensitive applications.
* **SATA Performance** (`50-sata.rules`): Configures SATA host link power management to `max_performance` - only applied on controllers that explicitly report LPM support, preventing issues on unsupported hardware.
* **I/O Scheduler Assignment** (`60-ioschedulers.rules`): Dynamically assigns optimal I/O schedulers based on drive type: `bfq` for HDDs, `mq-deadline` for SATA SSDs and eMMC, and `none` for NVMe SSDs.
* **HDD Performance Tuning** (`69-hdparm.rules`): Applies `hdparm -B 254 -S 0` to rotational ATA disks, setting near-maximum Advanced Power Management and disabling automatic spindown to prevent latency spikes.
* **NVIDIA Runtime Power Management** (`71-nvidia.rules`): Enables runtime PM (`power/control=auto`) on NVIDIA GPU driver bind and disables it (`power/control=on`) on unbind, reducing idle power draw and improving thermal behavior.
* **Wireless Regulatory Domain** (`85-iw-regulatory.rules`): Triggers the `iw-set-regdomain` service whenever a wireless device is added - ensuring USB WiFi adapters plugged in after boot also receive the correct regulatory domain, which is critical for full channel and TX power availability during wireless assessments.
* **CPU DMA Latency Access** (`99-cpu-dma-latency.rules`): Sets group ownership of `/dev/cpu_dma_latency` to `audio`, allowing timing-sensitive applications to request CPU latency targets without root.

### 🚀 Sysctl: Kernel Runtime Configuration

Sysctl parameters modify kernel behavior at runtime for system-wide performance and stability. Settings are applied via `usr/lib/sysctl.d/70-athena-settings.conf`, which loads after all Arch defaults to ensure correct override precedence.

* **Memory Management** (`vm.swappiness=100`): Pairs with the ZRAM udev rule which raises this to `150` once ZRAM is confirmed active. Ensures the kernel strongly prefers RAM-based swap over disk. Suitable because Athena always ships with ZRAM enabled.
* **VFS Cache Pressure** (`vm.vfs_cache_pressure=50`): Keeps inode and dentry caches in RAM longer, reducing syscall overhead for filesystem-heavy workloads such as compilers, package managers, and security tools that traverse large directory trees.
* **Dirty Page Limits** (`vm.dirty_bytes=268435456`, `vm.dirty_background_bytes=67108864`): Caps dirty page accumulation at fixed byte thresholds for predictable write-back behavior, preventing sudden I/O stalls from large buffer flushes.
* **Write-back Interval** (`vm.dirty_writeback_centisecs=1500`): Extends the kernel flusher wake-up interval to reduce unnecessary CPU wake-ups during idle periods.
* **Swap Readahead** (`vm.page-cluster=0`): Disables swap readahead clustering, reading exactly one page per fault. Optimal with ZRAM or NVMe swap where random access cost is negligible.
* **NMI Watchdog** (`kernel.nmi_watchdog=0`): Disables the NMI watchdog, freeing a hardware performance counter and reducing interrupt overhead. Beneficial for latency-sensitive workloads.
* **Unprivileged User Namespaces** (`kernel.unprivileged_userns_clone=1`): Allows normal users to create unprivileged namespaces, required for rootless containers (Podman), Flatpak sandboxing, and browser sandboxes.
* **Kernel Pointer Restriction** (`kernel.kptr_restrict=1`): Hides kernel pointers from unprivileged users while keeping them accessible to root. Improves security without breaking kernel-level debugging and exploit development workflows that require `/proc/kallsyms` access.
* **Network Receive Queue** (`net.core.netdev_max_backlog=4096`): Increases the network device input queue to reduce packet drops under heavy load - relevant for packet capture tools such as `wireshark`, `tcpdump`, and `airodump-ng`.
* **File Descriptor Limit** (`fs.file-max=2097152`): Raises the system-wide open file handle limit, preventing "too many open files" errors when running tools like `nmap`, `masscan`, fuzzers, or proxy interception tools that open large numbers of simultaneous connections.

### 🔧 Modprobe: Kernel Module Parameters

Modprobe configurations control module loading and behavior for hardware-specific optimizations. All files are placed under `usr/lib/modprobe.d/` so users can override any setting by dropping a file in `/etc/modprobe.d/` without package conflicts.

* **Module Blacklist** (`blacklist.conf`):
  * `iTCO_wdt` and `sp5100_tco` - blacklists Intel and AMD TCO watchdog timers to prevent spurious resets and reduce IRQ overhead.
  * `evbug` - blacklists the kernel input event debug module, which logs every keypress and mouse movement to the kernel ring buffer. On a security-focused system this module is functionally a keylogger and should never be loaded.
  * `pcspkr` and `snd_pcsp` - disables the PC speaker and its ALSA driver to prevent unwanted beeps and spurious audio devices.
  * `nouveau` - prevents the community NVIDIA driver from loading and conflicting with the `nvidia-open` kernel modules.
* **AMD GPU Driver Enforcement** (`amdgpu.conf`): Forces the `amdgpu` driver for older Southern Islands (GCN 1.0) and Sea Islands (GCN 2.x) AMD GPUs by enabling `si_support` and `cik_support` on the `amdgpu` module and disabling them on `radeon`. Without this, cards from the HD 7000 / R7 / R9 series default to the legacy `radeon` driver, losing access to modern Vulkan, compute (ROCm), and power management features. Has no effect on GCN 3+ hardware (RX 400 series onwards) which uses `amdgpu` automatically.
* **NVIDIA Driver Parameters** (`nvidia.conf`):
  * `NVreg_UsePageAttributeTable=1` - enables Page Attribute Table for faster CPU↔GPU memory access via write-combining.
  * `NVreg_InitializeSystemMemoryAllocations=0` - skips zeroing GPU memory buffers on allocation for faster launch times.
  * `NVreg_DynamicPowerManagement=0x02` - enables fine-grained runtime power management for mobile NVIDIA GPUs.
  * `NVreg_EnableS0ixPowerManagement=1` - enables S0ix modern standby support for proper suspend/resume on laptops with NVIDIA GPUs.

### 🔄 Modules: Kernel Module Loading

* **NT Sync** (`modules-load.d/ntsync.conf`): Loads the `ntsync` kernel module at boot. NT Sync implements Windows NT synchronization primitives directly in the kernel, dramatically improving performance and compatibility for Windows applications and games running under Wine or Proton. Particularly relevant for running Windows-only security tools, malware analysis binaries, and licensed software under Wine.

### ⏱️ Systemd: Service & System Management

Systemd unit and configuration files for streamlined boot, resource management, and service control.

* **Journal Log Limits** (`journald.conf.d/00-journal-size.conf`): Sets the `journald` size limit to `200M` - large enough to retain meaningful diagnostic history for driver issues, tool crashes, and kernel events relevant to security workflows.
* **Service Timeouts** (`system.conf.d/00-timeout.conf`): Sets `DefaultTimeoutStartSec=15s` and `DefaultTimeoutStopSec=10s`. Reduces the default 90-second timeouts so hung services fail fast and do not delay shutdown or boot.
* **File Descriptor Limits** (`system.conf.d/10-limits.conf`): Sets `DefaultLimitNOFILE=2048:2097152` for all system services, raising both soft and hard file descriptor limits.
* **Time Synchronization** (`timesyncd.conf.d/10-timesyncd.conf`): Configures `systemd-timesyncd` with Cloudflare (`time.cloudflare.com`) as the primary NTP server and Google plus the Arch pool as fallbacks.
* **ZRAM Generator** (`zram-generator.conf`): Configures ZRAM with `zstd` compression, `zram-size=ram` (dynamically allocated up to the size of physical RAM), and `swap-priority=100` to ensure ZRAM is always preferred over any optional disk swap partition the user may add. Setting the ZRAM size to the full RAM capacity provides maximum virtual memory headroom for memory-hungry pentesting workloads - running multiple simultaneous tools such as Burp Suite, Android emulators for mobile assessments, hashcat GPU cracking sessions, and several browser instances no longer risks OOM kills even under heavy load, since excess pages are compressed in RAM rather than triggering the OOM killer or stalling on slow disk swap.
* **Wireless Regulatory Domain Service** (`iw-set-regdomain.service` + `iw-set-regdomain.path`): A `.path` unit watches `/etc/localtime` for changes and triggers `iw-set-regdomain.service` to reapply the correct WiFi regulatory domain when the timezone changes. Works in conjunction with `85-iw-regulatory.rules` for complete regulatory domain management.
* **User Service Resource Delegation** (`user.conf.d/delegate.conf`): Delegates `cpu`, `cpuset`, `io`, `memory`, and `pids` cgroup controllers to user sessions, enabling proper per-user resource isolation and supporting rootless container workflows.
* **RTKit Log Level** (`system/rtkit-daemon.service.d/override.conf`): Suppresses verbose debug output from `rtkit-daemon` in the journal while keeping informational messages intact.

### 🧹 Tmpfiles: Temporary File & THP Management

* **Coredump Retention** (`coredump.conf`): Clears coredumps older than 3 days from `/var/lib/systemd/coredump`. On a pentesting system where crashes are common (fuzzing, exploit development, deliberate fault injection), coredumps can accumulate quickly - this prevents unbounded disk consumption while keeping recent dumps for analysis.
* **THP Defragmentation** (`thp.conf`): Sets `transparent_hugepage/defrag` to `defer+madvise`. Prevents the kernel from aggressively defragmenting RAM to form huge pages (which causes latency stalls), and instead only forms them when applications explicitly request it via `madvise`. Particularly beneficial for applications using tcmalloc such as Chrome and Electron-based security tools.
* **THP Shrinker** (`thp-shrinker.conf`): Sets `khugepaged/max_ptes_none=409` on kernel 6.12+. Splits huge pages where more than 80% of sub-pages are zero-filled, reducing memory waste from the `THP=always` policy while preserving the performance benefit for genuinely populated huge pages.

### 📡 Wireless Regulatory Domain

A complete regulatory domain management system ensuring WiFi adapters operate on the correct channels and power levels for the user's region.

* **`iw-set-regdomain`** (`usr/lib/iw-set-regdomain`): Shell script that determines the correct country code from the system timezone via `timedatectl` and `zone.tab`, then applies it with `iw reg set`. Supports a manual override file at `/etc/iw-regdomain` (format: `COUNTRY=XX`) for users who need to set a specific domain regardless of timezone. Gracefully handles UTC and GMT timezones with no associated country.


**Note for wireless assessments**: Without a correct regulatory domain the kernel falls back to the world domain (`00`) - the most restrictive common denominator across all countries - which silently disables channels 12 and 13 in the 2.4GHz band, restricts most 5GHz DFS channels, limits TX power well below what the hardware is capable of, and causes tools like `airodump-ng` and `wash` to silently miss networks entirely on restricted channels. A wireless pentester operating under `00` may walk away from an assessment believing certain networks do not exist when they are simply on channels their adapter could reach but the kernel is blocking. This script ensures the full legal channel range and TX power for the user's region are available from boot, and reapplies the correct domain automatically whenever a new wireless adapter is plugged in or the timezone changes, so the pentester never misses a network, a channel, or a beacon. Use `iw reg get` to verify your current domain.

* **Triggered by three independent events**:
  1. Boot - via `iw-set-regdomain.service`
  2. Timezone change - via `iw-set-regdomain.path` watching `/etc/localtime`
  3. WiFi device plug-in - via `85-iw-regulatory.rules` triggering the service on `ieee80211` device add


### 🖱️ Input

* **Touchpad Tapping** (`usr/share/X11/xorg.conf.d/20-touchpad.conf`): Enables tap-to-click for all libinput touchpads in X11. Has no effect under Wayland where touchpad settings are managed by the compositor.

### 🔒 Security Considerations

Several settings in this package involve deliberate trade-offs relevant to a security-focused distribution:

* **`kernel.kptr_restrict=1`** rather than `2`: Root retains access to `/proc/kallsyms` for kernel exploit development and kernel-level security research, while unprivileged users cannot read kernel pointers.
* **`evbug` blacklisted**: This module logs all input events to the kernel ring buffer and is functionally a keylogger. It is blacklisted unconditionally.
* **`nouveau` blacklisted**: Required to prevent conflicts with `nvidia-open` kernel modules. Arch Linux no longer ships the proprietary `nvidia` package.

## File Hierarchy

All distro-owned configuration files follow the correct XDG/systemd split to allow clean user overrides without package conflicts:

| Package ships in | User overrides in |
|---|---|
| `usr/lib/modprobe.d/` | `etc/modprobe.d/` |
| `usr/lib/sysctl.d/` | `etc/sysctl.d/` |
| `usr/lib/udev/rules.d/` | `etc/udev/rules.d/` |
| `usr/lib/systemd/system/` | `etc/systemd/system/` |
| `usr/lib/tmpfiles.d/` | `etc/tmpfiles.d/` |
| `usr/share/libalpm/hooks/` | `etc/pacman.d/hooks/` |
