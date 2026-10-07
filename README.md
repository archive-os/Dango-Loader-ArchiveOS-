# Dango Loader 0.3.5

A customizable Debian-family live ISO installer with animated pages, a sidebar step list, light/dark/system appearance and independent distribution branding. The default accent is #49AFE8.

## Preview and build

```sh
./run.sh --demo
./run.sh --demo --language tr
./build.py
sudo ./install.sh
```

Demo mode uses synthetic disks and installation events. Building creates `dango-loader_0.3.5_all.deb` and `SHA256SUMS`; it does not install anything. `install.sh` installs into the current system/chroot, so run it inside the live image when preparing an ISO. Installation is enabled in the shipped `/etc/dango-loader/installer.json` profile. Set its trusted SquashFS path and distribution-specific account/bootloader settings before release.

## Minimum requirements and password policy

Installation requires at least 32 GB total installation space (32,000,000,000 bytes, about 29.80 GiB), including boot partitions, and 4 GiB installed RAM. Boot/alignment space is deducted from this total when validating root partitions, rather than added to the minimum disk size. The Windows allocation slider uses whole GiB, so its minimum is 30 GiB. Checks run in the UI and privileged backend, not just as informational text. RAM is measured using MemTotal; up to 256 MiB of firmware/kernel reservation is allowed so a VM configured with 4 GiB is not incorrectly rejected.

The user/sudo password is required. “Use a reliable password” is checked by default and enforces at least 8 characters. Unchecking it allows any nonempty password, including 1–3 characters. No extra complexity requirement is imposed. This sets the same password for user login and sudo; root remains locked. The translated explanation is shown beside the checkbox. The UI checks password matching; the privileged backend enforces the selected minimum length.

## Guided installation

Welcome → Location → Keyboard → Partitions → User → Summary → Installation → Finish.

The welcome language selector offers 82 entries, including English/Turkish and the available Calamares translation catalogs. The system locale selects the initial language. Main navigation and matching Calamares phrases use these catalogs; custom Dango descriptions without a translated counterpart currently fall back to English. This is not a claim of complete translation of every string into every language. Translation sources and licenses are in `translations/`.

The offline world map and system tzdata list stay synchronized. Map clicks choose the nearest timezone reference city, not exact legal timezone boundaries. Keyboard layouts and variants use the complete system XKB catalog. Selection also changes the current Plasma keyboard through notified KConfig settings; other X11 sessions use setxkbmap. Demo mode does not change the current keyboard. The chosen layout/variant is saved to the installed system.

Partitions first presents three categories:

- **Install alongside Windows:** select an eligible NTFS partition and drag the allocation bar or enter the space to allocate. Filesystem/minimum-size and no-action resize checks run before changes. BitLocker, dirty/hibernated NTFS and mounted target disks are refused. Disable Windows Fast Startup/hibernation before testing.
- **Erase disk and install:** select a physical disk, explicitly confirm erasure, and create a GPT layout with a boot partition (EFI for UEFI, bios_grub for BIOS) and root. Empty disks are detected even if they have no existing partitions. All existing data on the selected disk is removed when installation starts.
- **Use existing partitions:** select an unmounted root partition; UEFI also requires a separate preserved EFI partition. BIOS/GPT requires a separate unused bios_grub partition of at least 1 MiB, preserved during installation. BIOS/MBR requires at least a 1-MiB gap before its first partition for GRUB embedding.

Btrfs is selected by default for snapshot-friendly `@` and `@home` subvolumes with zstd compression. Ext4 is also available. This layout does not itself schedule snapshots. Advanced opens KDE Partition Manager; rescan after editing. The separate authorized scan obtains metadata that an ordinary desktop user cannot read. Existing Linux filesystem detection does not imply that an unmounted distribution's exact identity has been verified.

After disk configuration, a separate memory page offers **zram enabled by default (recommended)** for all methods, including Windows dual boot and erase. The target zram service uses 100% of RAM up to 4 GiB, otherwise 50%, capped at 8 GiB. It does not provide hibernation. Progress, slideshow and expandable terminal show the current installation operation.

## Branding and slides

Edit `branding/branding.json`, replace `branding/assets/logo.png`, and set a hex accent color. See `branding/README.md`. Installer identity remains the separate Dango skewer logo. Numbered PNG slides 1–10 are played in numeric order; missing numbers are skipped. English/Turkish and dark slide variants introduce ArchiveOS tools. Slides describe tools but do not install their packages.

## Extracted ISO integration

```sh
sudo ./integrate-iso.sh /path/to/extracted
```

This installs the local DEB into the specified image with dpkg (without running triggers). Dependencies must already be installed in the image; use `install.sh` within its chroot to obtain missing dependencies. It normalizes ownership of the privileged Dango code/config and their image ancestors, which is necessary when the SquashFS was extracted under a non-root owner. It disables only legacy `aos-live.desktop` and `welcome-live.desktop` startup entries, keeping timestamped backups under `/var/backups/dango-live-startup`. Other autostart applications remain untouched.

The package automatically launches Dango in live sessions only (`boot=casper`, `boot=live`, `root=live:` or standard live mount markers), creates a desktop shortcut, and uses a session lock to avoid duplicate instances. Normal installed sessions do not launch it. After successful bootloader setup in the installation target, the installer purges its own target package and shortcuts; the running live app remains available. `enable-live-autostart.sh` can restore the packaged startup entry from inside the image/chroot.

## Scope and verification

The backend supports Debian-family x86_64 UEFI/GPT and BIOS installation. BIOS erase mode creates GPT with a bios_grub partition. Existing BIOS targets may use GPT (with bios_grub) or MBR (with sufficient GRUB embedding space). BIOS Windows alongside supports basic primary MBR layouts with a free primary partition slot; extended/logical layouts use Advanced. Windows on GPT requires UEFI for matching dual boot. The trusted source image must contain its kernel, GRUB tools/modules and locale utilities for the selected firmware. Install grub-pc-bin and grub-efi-amd64-bin inside the live source; alongside mode also needs os-prober. Secure Boot enrollment, LUKS/LVM/RAID and hibernation are not implemented. Passwords travel through helper stdin, not command arguments or logs. Device identities and mount states are checked again before changes. Failure stops at the first error and unmounts the target; completed disk changes are not rolled back.

```sh
/usr/bin/python3 tests/test_dango.py
/usr/bin/python3 tests/test_engine.py
/usr/bin/python3 tests/test_live.py
/usr/bin/python3 tests/test_appearance.py
/usr/bin/python3 tests/test_new_installation.py
/usr/bin/python3 tests/test_erase_engine.py
/usr/bin/python3 tests/test_requirements_bios.py
```

These tests use temporary files, synthetic inventories and mocked disk commands. UI/translation, blank-disk detection, identity/mount guards, erasure command ordering, live cleanup and keyboard command construction are checked. Real disk formatting, BIOS/UEFI installed-system boot, Windows resizing and live session keyboard behavior still require disposable VM validation.

## Administrator startup
Normal startup requests administrator authorization immediately through the graphical Polkit dialog. The installer then runs as root and launches its protected helper directly, without another authorization prompt. Demo/help do not require root. With sudo already used (for example `sudo -E dango-loader` in a terminal), no startup prompt is requested. Keyboard settings are applied as the originating desktop user, not saved into root’s configuration.

## Read-only live installation source
Protected code/configuration still require root ownership and no group/other writes. The SquashFS source additionally permits ISO9660 or SquashFS media verified by the kernel mount table as read-only, regardless of archived uid/mode bits. The mount attachment ancestors must still be protected. Writable media, ordinary unprotected files, other filesystem types and ambiguous mount reports are refused. This handles ISO trees whose casper directory retains non-root ownership; no chmod/chown on the mounted CD is needed.

Disk detection uses the single authorized Detect disks button and displays detected physical disks on the storage page. Disk controls and descriptions are hidden on the zram screen; returning restores them.

GRUB configuration is checked using sh -n before destructive installation when present in the source image. A simple missing closing quote on GRUB_DISTRIBUTOR is repaired in the target with an etc/default/grub.dango-backup copy. Other invalid syntax is rejected. The configuration is parsed, never executed, during checking. Actual BIOS/UEFI boot still needs VM validation.

The root partition name is configurable through root_partition_name in installer.json (ArchiveOS: DangoRoot / ArchiveOS). Btrfs also uses this full filesystem label. Ext4/XFS label byte limits require the shorter distro bootloader_id (ArchiveOS); GPT partition names retain the full text. Installation slides expand into the available area with a 300-pixel minimum instead of the old 210-pixel fixed size. Open installation terminal displays a separate nonmodal read-only log window; closing/reopening preserves logs and does not stop installation.

Storage now shows the installation disk selector above the method categories. Erase uses this selected disk; alongside/manual root and EFI choices are filtered to it. Changing disks resets confirmation and NTFS limits; the backend rejects cross-disk or changed-disk plans. The keyboard typing test is removed. During installation, the step sidebar is hidden to give slides the full window width; progress and phase remain visible, and the sidebar returns on other pages.
