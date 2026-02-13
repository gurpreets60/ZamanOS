# ZamanOS

An x86_64 UEFI live USB Linux appliance produced with Buildroot and delivered as a Weston kiosk.

## Mission Profile
- **Goal**: Boot a Buildroot-based image straight into Weston kiosk, then auto-launch the ZamanOS control center, which can spawn Cog, a minimal file explorer, and bench visualizations.
- **Disk policy**: Never auto-mount internal disks. User-initiated mounts must be read-only with `ro,nosuid,nodev,noexec`, NTFS via `ntfs3`, and encrypted volumes (BitLocker/LUKS) must be detected and labeled as encrypted without unlock support.
- **Benchmarks**: Provide a `benchctl` CLI wrapper around `fio` JSON runs, expose system inventory, and allow the GUI to export JSON/CSV/TXT results to removable media.
- **Non-goals (v1)**: No installer, package updates, SSH daemon, or full desktop environment.

## Repository Layout
```
buildroot/              # Upstream Buildroot checkout or extracted tarball (pinned release)
board/zamanos_x86_64/   # Custom board support files: kernel configs, rootfs overlays, post-image hooks
package/benchctl/       # benchctl package description, sources, patches
package/gui-app/        # Weston kiosk application package
package/mount-helper/   # Mount helper enforcing disk policy
configs/                # Buildroot defconfigs (zamanos_defconfig)
docs/                   # Product docs, architecture notes, benchmarks, troubleshooting
```

## Task Guide (commands + expected output)
Each task below should be documented in commit messages as you iterate. Capture the command output snippets so regressions are easy to spot.

### Task 1 – Pin Buildroot sources (submodule or tarball)
Goal: land Buildroot 2025.11.1 (latest stable as of January 20, 2026) so everyone builds against the same tree.

**Option A: git submodule**
```bash
git submodule add --name buildroot --branch 2025.11.1 https://gitlab.com/buildroot.org/buildroot.git buildroot
git -C buildroot checkout 2025.11.1
```
Expected output:
```
Cloning into 'buildroot'...
Submodule path 'buildroot': checked out '1a2b...'
Note: switching to '2025.11.1'.
```

**Option B: pinned tarball**
```bash
curl -LO https://buildroot.org/downloads/buildroot-2025.11.1.tar.xz
tar -xf buildroot-2025.11.1.tar.xz --strip-components=1 -C buildroot
```
Expected output:
```
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
100 13719  100 13719    0     0  2150k      0 --:--:-- --:--:-- --:--:-- 2150k
```

### Task 2 – Author `zamanos_defconfig`
Goal: seed Buildroot with the Weston kiosk profile and custom packages via `configs/zamanos_defconfig`.

1. Start from the upstream QEMU EFI defconfig and save it as our base:
   ```bash
   cd buildroot
   make qemu_x86_64_efi_defconfig
   make savedefconfig BR2_DEFCONFIG=../configs/zamanos_defconfig
   ```
   Expected output:
   ```
   BR2_DEFCONFIG='/home/user/projects/ZamanOS/../configs/zamanos_defconfig'
   ```
2. Edit `configs/zamanos_defconfig` and enable:
   - EFI grub2/OVMF boot
   - Weston + kiosk-shell launching `/usr/bin/zamanos-gui`
   - Packages: `benchctl`, `zamanos-gui`, `mount-helper`, `fio`, `ntfs-3g` (read-only), and kernel `CONFIG_NTFS3_FS`.
   - Rootfs overlays from `board/zamanos_x86_64/overlay`.

### Task 3 – Build the image
Goal: produce the bootable artifact that satisfies acceptance.

```bash
cd buildroot
make BR2_EXTERNAL=$(pwd)/.. zamanos_defconfig
make BR2_EXTERNAL=$(pwd)/..
```
Expected output snippets:
```
>>> benchctl  Building
>>> zamanos-gui  Building
>>> mount-helper  Building
>>>   Installing host-grub2
>>>   Finalizing root filesystem
```
Artifacts land under `output/images/` (ISO/ESP, rootfs tarballs, EFI images). Copy the ISO to a USB stick with `dd` or `bmaptool`.

## Acceptance
1. `cd buildroot && make BR2_EXTERNAL=$(pwd)/.. zamanos_defconfig` passes without errors using the pinned Buildroot release.
2. `cd buildroot && make BR2_EXTERNAL=$(pwd)/..` (or `make` inside `buildroot/` with the defconfig baked) produces a bootable UEFI image artifact (ISO + ESP binary) under `buildroot/output/images/`.
3. Booting the image drops into Weston kiosk mode with the control center full screen, and only user-initiated read-only mounts are possible.

## Next Steps
- Flesh out `configs/zamanos_defconfig` with the actual Buildroot options.
- Implement `package/benchctl`, `package/gui-app`, and `package/mount-helper` build recipes.
- Add board assets: kernel config fragments, overlays, post-build scripts, and bench export helpers.
