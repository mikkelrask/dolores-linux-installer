# Dolores Linux ISO

Dolores is an Archiso profile for building a bootable Dolores Linux installer.
The live ISO boots into an Archiso environment and includes `install-dolores`,
which partitions a target disk, installs the base system, configures Mango,
sets up `opendoas`, and writes Dolores OS metadata.

## Build Host Requirements

Install the tools needed to build and test the ISO:

```sh
sudo pacman -S archiso qemu-system-x86 qemu-img qemu-ui-gtk edk2-ovmf
```

If you use `doas` on the host, replace `sudo` with `doas`.

The profile uses the local `pacman.conf` declared in `profiledef.sh`. Arch
packages are pulled from explicit Arch mirrors, and the CachyOS repo is enabled
for packages such as `mango-wm` and `librewolf-bin`.

## Build

From the repository root:

```sh
mkdir -p work out
sudo mkarchiso -v -w work -o out .
```

The ISO is written to `out/` with a name like:

```text
out/dolores-2026.05.22-x86_64.iso
```

After a failed build or after changing package lists, boot files, or initramfs
configuration, rebuild clean:

```sh
sudo rm -rf work out
mkdir -p work out
sudo mkarchiso -v -w work -o out .
```

## Quick Checks

Check the installer script:

```sh
bash -n airootfs/usr/local/bin/install-dolores
shellcheck airootfs/usr/local/bin/install-dolores
```

Check that bootloader entries point at the Zen kernel used by the ISO:

```sh
rg 'vmlinuz-linux|initramfs-linux' efiboot syslinux grub
```

Expected paths use:

```text
vmlinuz-linux-zen
initramfs-linux-zen.img
```

After a successful build, check that the live initramfs contains Archiso hooks:

```sh
lsinitcpio -l work/iso/arch/boot/x86_64/initramfs-linux-zen.img | grep archiso
```

If this prints nothing, the ISO may boot into an emergency shell because it
cannot locate and mount the live root image.

## Test In QEMU

Create a sparse test disk:

```sh
qemu-img create -f qcow2 /tmp/dolores-test.qcow2 40G
```

This does not immediately use 40 GB on the host. The file grows as the VM writes
data. Check actual usage with:

```sh
du -h /tmp/dolores-test.qcow2
qemu-img info /tmp/dolores-test.qcow2
```

### BIOS Boot

```sh
qemu-system-x86_64 \
  -enable-kvm \
  -m 4096 \
  -cpu host \
  -smp 4 \
  -drive file=/tmp/dolores-test.qcow2,format=qcow2,if=virtio \
  -cdrom out/dolores-*.iso \
  -boot d \
  -display gtk
```

### UEFI Boot

Use a writable copy of the OVMF vars file:

```sh
cp /usr/share/edk2/x64/OVMF_VARS.4m.fd /tmp/dolores_OVMF_VARS.fd
```

Then boot:

```sh
qemu-system-x86_64 \
  -machine q35,accel=kvm \
  -m 4096 \
  -cpu host \
  -smp 4 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/x64/OVMF_CODE.4m.fd \
  -drive if=pflash,format=raw,file=/tmp/dolores_OVMF_VARS.fd \
  -drive file=/tmp/dolores-test.qcow2,format=qcow2,if=virtio \
  -drive file=out/dolores-*.iso,format=raw,media=cdrom,readonly=on \
  -boot order=d,menu=on \
  -display gtk
```

If OVMF opens its firmware menu instead of booting the ISO, choose the QEMU
DVD-ROM entry from the boot manager.

## Run The Installer

Inside the live ISO:

```sh
install-dolores
```

The installer must run as root. The live Archiso shell is already root; if you
are not root, switch to a root shell first:

```sh
su -
install-dolores
```

The installer will erase the selected target disk.

The installed disk is prepared for legacy BIOS boot with GRUB. When the live
ISO is booted in UEFI mode, the installer also installs systemd-boot on the EFI
System Partition.

## Reset Test State

Remove the test disk to start the installer from a blank drive:

```sh
rm -f /tmp/dolores-test.qcow2
qemu-img create -f qcow2 /tmp/dolores-test.qcow2 40G
```

Remove stale UEFI firmware variables:

```sh
rm -f /tmp/dolores_OVMF_VARS.fd
cp /usr/share/edk2/x64/OVMF_VARS.4m.fd /tmp/dolores_OVMF_VARS.fd
```

## Troubleshooting

If `mkarchiso` complains about missing boot packages, make sure
`packages.x86_64` includes the packages required by the enabled boot modes in
`profiledef.sh`.

If package downloads fail with 404s, remove `work/` and rebuild. If signatures
fail, refresh the host keyrings and remove bad cached packages:

```sh
sudo pacman -Sy archlinux-keyring cachyos-keyring
sudo pacman-key --populate archlinux cachyos
sudo pacman-key --refresh-keys
sudo pacman -Scc
```

If the boot menu immediately redraws after selecting the install entry, the boot
entry probably points at the wrong kernel or initramfs path.

If a VM stops at `Booting from Hard Disk...`, it is usually booting the installed
disk with legacy BIOS but the disk has no BIOS bootloader. Rebuild the ISO with
the current installer and reinstall, or boot the existing install media and
install GRUB to the target disk from a chroot.

If boot drops into an emergency shell with a failed mount of the real root, the
live initramfs probably lacks Archiso hooks. Ensure `packages.x86_64` includes
`mkinitcpio-archiso` and that the preset in `airootfs/etc/mkinitcpio.d/` matches
the kernel package.

## Mango Keybindings

After install, Dolores boots into Mango (a wlroots Wayland compositor). The
config lives at `~/.config/mango/config.conf` (stowed from dotfiles). Here are
the essentials:

| Action | Binding |
|--------|---------|
| Terminal | `Alt+Enter` |
| Browser (LibreWolf) | `Alt+W` |
| Launcher | `Alt+Space` |
| Run command | `Alt+D` |
| Switch tag | `Alt+1` – `Alt+9` |
| Move window to tag | `Super+Alt+1` – `Super+Alt+9` |
| Focus window | `Super+h` / `j` / `k` / `l` or `Super+Shift` + arrows |
| Swap window | `Super+Shift` + `h` / `j` / `k` / `l` or arrows |
| Toggle floating | `Alt+V` |
| Toggle maximize | `Alt+A` |
| Toggle fullscreen | `Alt+F` |
| Toggle overview | `Alt+Tab` |
| Cycle layout | `Super+N` |
| Set tile layout | `Super+T` |
| Toggle gaps | `Super+A` |
| Kill focused client | `Alt+Shift+Q` |
| Reload config | `Ctrl+Shift+R` |
| Quit Mango | `Ctrl+Shift+X` |

Media keys (`XF86Audio*`) control volume via PulseAudio/PipeWire by default.
