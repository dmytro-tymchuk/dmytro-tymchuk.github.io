# Arch Linux Installation

Dmytro Tymchuk | October 26, 2026

## 1. Create the VM

Created an ARM64 VM in UTM:

- QEMU virtualization
- 2 CPU cores
- 4096 MiB RAM
- 32 GB virtual disk
- OpenGL acceleration disabled
- No shared directory

Used the instructor-approved ARM64 ISO:
`archboot-2026.10.26-02.07-7.3.0-1-aarch64-ARCH-latest-aarch64.iso`

Guide: https://wiki.archlinux.org/title/Installation_guide

## 2. Boot the live environment

Selected **Launch UEFI Archboot Arch Linux aarch64**.
After preparation finished, pressed **Ctrl+C** to open a shell
for manual installation.

Kept the default keyboard layout.

```bash
uname -m
```

Checks the architecture. Returned `aarch64`.

## 3. Verify UEFI boot

```bash
ls /sys/firmware/efi
```

Checks for the UEFI interface. The directory existed and
contained EFI entries, confirming UEFI boot.

## 4. Verify internet access

```bash
ping -c 3 archlinux.org
```

Tests DNS and network connectivity. All three packets returned
with 0% packet loss.

## 5. Check the clock

```bash
timedatectl
```

Shows time and synchronization status. Time zone was UTC;
NTP was inactive and the clock was not synchronized.

Next: Enable time synchronization, then identify the virtual disk.

## 6. Identify the virtual disk

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Listed storage devices. Identified `/dev/vda` as the empty
32 GB virtual disk. `/dev/sr0` was the ISO and `/dev/zram0`
was the live environment in RAM.

## 7. Partition the disk

Planned layout: GPT partition table, a 1 GiB EFI partition,
and the remaining space for an ext4 root partition.
Swap will use a file later.

```bash
fdisk /dev/vda
```

Opened the VM's virtual disk in the partition editor.

Inside fdisk, entered the following options:

| Input                  | Purpose                                                  |
| ---------------------- | -------------------------------------------------------- |
| `g`                    | Create a GPT partition table.                            |
| `n`, `1`, Enter, `+1G` | Create a 1 GiB partition using the default first sector. |
| `t`, `uefi`            | Set partition 1 to EFI System.                           |
| `n`, `2`, Enter, Enter | Create partition 2 using the remaining disk space.       |
| `p`                    | Display and review the planned partitions.               |

Confirmed `/dev/vda1` was 1 GiB (EFI System) and
`/dev/vda2` was approximately 31 GiB (Linux filesystem).

Entered `w` to save the partition table and exit fdisk.

## 8. Format the partitionscl

```bash
mkfs.fat -F 32 /dev/vda1
```

Formatted the EFI System partition as FAT32.

```bash
mkfs.ext4 /dev/vda2
```

Formatted the root partition as ext4.

## 9. Mount the partitions

```bash
mount /dev/vda2 /mnt
```

Mounted the root partition at the installation directory.

```bash
mount --mkdir /dev/vda1 /mnt/boot
```

Created the boot directory and mounted the EFI partition.

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
```

Verified that the ext4 root partition was mounted at `/mnt`
and the FAT32 EFI partition at `/mnt/boot`.

## 10. Check installation tools and repositories

```bash
command -v pacstrap arch-chroot genfstab
```

Confirmed all three manual installation tools were available.

```bash
cat /etc/pacman.conf
```

Checked the package manager configuration. The architecture
was `aarch64`, and the repositories were enabled.

```bash
grep -Ev '^[[:space:]]*(#|$)' /etc/pacman.d/mirrorlist
```

Displayed the active download mirror, excluding comments and
blank lines. The mirror pointed to Arch Linux ARM.

```bash
pacman -Si base linux-aarch64
```

Attempted to check the base system and ARM64 kernel packages.
The query failed because repository databases had not been downloaded.

```bash
pacman -Sy
```

Downloaded the repository package lists to resolve the issue.

```bash
pacman -Si base linux-aarch64
```

Repeated the query and confirmed both packages were available.
Selected `linux-aarch64` because the VM uses ARM64.

## 11. Install the base system

```bash
pacstrap -K /mnt base linux-aarch64 linux-firmware networkmanager sudo nano
```

Installed the base system, ARM64 kernel, firmware, NetworkManager,
sudo, and Nano into `/mnt` on the VM's virtual disk.
The `-K` option initialized a new package signing keyring.

## 12. Generate the filesystem table

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Generated the partition mount configuration using UUIDs and
appended it to the installed system's `/etc/fstab`.

```bash
cat /mnt/etc/fstab
```

Displayed the file to review the root and EFI mount entries.

## 13. Enter and configure the installed system

```bash
arch-chroot /mnt
```

Entered the installed system to configure it.

```bash
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```

Set the time zone to America/Chicago.

```bash
hwclock --systohc
```

Updated the VM's hardware clock from the system clock.

## 14. Configure language and keyboard

```bash
nano /etc/locale.gen
```

Initially, Nano failed because the installed system did not
recognize the live environment's `kmscon` terminal type.

```bash
clear
```

Attempted to clear the screen, but it failed for the same reason.

```bash
export TERM=xterm-256color
```

Set a compatible terminal type for the current session,
resolving the issue.

```bash
nano /etc/locale.gen
```

Reopened the locale configuration and removed the leading `#`
from `en_US.UTF-8 UTF-8`. Saved with Ctrl+O, Enter, and
exited with Ctrl+X.

```bash
locale-gen
```

Generated the US English UTF-8 locale. The output confirmed
`en_US.UTF-8... done` and `Generation complete`.

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Set the system's default language to US English with UTF-8 encoding.

```bash
echo "KEYMAP=us" > /etc/vconsole.conf
```

Set the console keyboard layout to US.

## 15. Set hostname and enable services

```bash
echo "arch-dmytro" > /etc/hostname
```

Set the VM's hostname to `arch-dmytro`.

```bash
systemctl enable NetworkManager
```

Enabled NetworkManager to start automatically at boot.

```bash
systemctl enable systemd-timesyncd
```

Enabled automatic time synchronization at boot.

## 16. Set the root password

```bash
passwd
```

Set and confirmed a password for the root administrator account.
The terminal hid the password while typing and confirmed
`password updated successfully`.

## 17. Create user accounts

```bash
useradd -m -G wheel -s /bin/bash dmytro
```

Created my account, `dmytro`. The `-m` option created a home
directory, `-G wheel` added the account to the wheel group,
and `-s /bin/bash` selected Bash as the initial shell.

```bash
passwd dmytro
```

Set and confirmed my account password.

```bash
useradd -m -G wheel -s /bin/bash codi
```

Created the required `codi` account with a home directory,
wheel group membership, and Bash as the initial shell.

```bash
passwd codi
```

Set and confirmed codi's password. Both password commands
reported `password updated successfully`.

The wheel group will receive sudo permissions in the next step.
An alternative shell will be installed and configured later.

## 18. Configure sudo permissions

```bash
EDITOR=nano visudo
```

Uncommented the following line to allow wheel group members
to use sudo with their own password:

```text
%wheel ALL=(ALL:ALL) ALL
```

```bash
visudo -c
```

Validated the configuration. The result was `/etc/sudoers: parsed OK`.

```bash
sudo -l -U dmytro
sudo -l -U codi
```

Confirmed both accounts could run all commands through sudo.

## 19. Install Zsh and SSH

```bash
pacman -S zsh openssh
```

Attempted to install Zsh as an alternative to Bash and OpenSSH
for remote access. Accepted the signing-key import, but package
verification failed with an "unknown trust" error. Answered `y`
to delete the affected cached packages. No packages were installed.

```bash
pacman-key --populate archlinuxarm
```

Attempted to load the ARM signing keys, but the required
`archlinuxarm.gpg` file was missing.

```bash
pacman -S archlinuxarm-keyring
```

Attempted to install the missing keyring package. This also
failed with unknown trust, so I accepted deletion of the cached
package and used the live installer's keyring files instead.

```bash
exit
```

Left the chroot and returned to the live environment inside the VM.

```bash
ls -l /usr/share/pacman/keyrings/
```

Confirmed the live environment contained `archlinuxarm.gpg`,
`archlinuxarm-trusted`, and `archlinuxarm-revoked`.

```bash
cp /usr/share/pacman/keyrings/archlinuxarm* /mnt/usr/share/pacman/keyrings/
```

Copied those ARM keyring files into the installed system.

```bash
arch-chroot /mnt
```

Re-entered the installed system.

```bash
pacman-key --populate archlinuxarm
```

Successfully imported the ARM keys, locally signed one trusted
key, and updated the trust database.

```bash
clear
```

Attempted to clear the screen, but the new chroot session again
reported that `kmscon` was an unknown terminal type.

```bash
pacman -S archlinuxarm-keyring zsh openssh
```

Retried installation. Signature verification passed, but pacman
reported file conflicts with the three manually copied keyring files.

```bash
pacman -S --overwrite 'usr/share/pacman/keyrings/archlinuxarm*' archlinuxarm-keyring zsh openssh
```

Allowed the keyring package to replace only the copied ARM
keyring files so pacman could manage them. Successfully installed
the ARM keyring, Zsh, OpenSSH, and the libedit dependency.

```bash
export TERM=xterm-256color
```

Restored a compatible terminal type for the current chroot session.

The installation completed successfully with package signature
checking enabled.

## 20. Configure Zsh and enable SSH

```bash
chsh -s /usr/bin/zsh dmytro
```

Changed my account's default login shell to Zsh.

```bash
chsh -s /usr/bin/zsh codi
```

Changed codi's default login shell to Zsh.

```bash
systemctl enable sshd
```

Enabled the SSH server to start automatically at boot.

```bash
getent passwd dmytro codi
```

Verified both accounts used `/usr/bin/zsh` as their login shell.

```bash
systemctl is-enabled sshd
```

Confirmed SSH was enabled. Its running status will be checked
after booting the installed system.

## 21. Configure Zsh colors and aliases

Opened the Zsh configuration file for the personal account:

```bash
nano /home/dmytro/.zshrc
```

Added command completion, a colored prompt, and useful aliases:

```zsh
# Enable command completion.
autoload -Uz compinit
compinit

# Color the username, hostname, and current directory.
PROMPT='%F{green}%n@%m%f %F{blue}%~%f %# '

# Useful shortcuts and colored output.
alias ls='ls --color=auto'
alias ll='ls -lah'
alias grep='grep --color=auto'
alias ..='cd ..'
alias update='sudo pacman -Syu'
```

The prompt displays the username and hostname in green and the
working directory in blue. The aliases provide a detailed file
listing, colored output, a shortcut to the parent directory,
and a system update command.

Copied the configuration to the codi account and assigned
ownership of each file:

```bash
cp /home/dmytro/.zshrc /home/codi/.zshrc
chown dmytro:dmytro /home/dmytro/.zshrc
chown codi:codi /home/codi/.zshrc
```

Switched to the personal account to test the configuration:

```bash
su - dmytro
ll
```

The prompt initially displayed incorrectly. Inspected its setting:

```bash
grep '^PROMPT=' ~/.zshrc
```

Corrected the PROMPT setting in the configuration file so that
the entire setting appeared on one line:

```bash
nano ~/.zshrc
```

Reloaded the configuration:

```bash
source ~/.zshrc
```

Confirmed that the username and hostname appeared in green,
the working directory appeared in blue, and ll produced a
detailed directory listing.

Returned to root, copied the corrected configuration to codi,
and restored its ownership:

```bash
exit
cp /home/dmytro/.zshrc /home/codi/.zshrc
chown codi:codi /home/codi/.zshrc
```

Tested the second account:

```bash
su - codi
ll
```

Confirmed that codi also had the colored prompt and a working
ll alias. Switching users displayed a ttyname warning, but the
shell opened and the configuration worked.

Returned to the root shell inside the installed system:

```bash
exit
```

## 22. Install and configure the UEFI bootloader

Checked the installed kernel files and available bootloader tools:

```bash
ls -lh /boot
command -v bootctl grub-install
```

Confirmed that `/boot/Image` and `/boot/initramfs-linux.img`
existed and that bootctl was available.

Installed systemd-boot on the EFI System Partition:

```bash
bootctl --esp-path=/boot install
```

The command copied the ARM64 bootloader to
`/boot/EFI/systemd/systemd-bootaa64.efi` and installed the
fallback loader at `/boot/EFI/BOOT/BOOTAA64.EFI`.
EFI variable modifications were skipped inside the chroot.

The installation warned that `/boot` and its random seed were
accessible to other users. Edited the filesystem configuration:

```bash
nano /etc/fstab
```

On the `/boot` entry, changed `fmask=0022,dmask=0022` to
`fmask=0077,dmask=0077` to restrict access to root.

Attempted to apply the permissions by remounting:

```bash
mount -o remount,fmask=0077,dmask=0077 /boot
findmnt -no OPTIONS /boot
```

The check still showed the old permissions. Unmounted and
mounted the partition again with the required options:

```bash
cd /
umount /boot
mount -o fmask=0077,dmask=0077 /dev/vda1 /boot
```

An initial typo, `unmount`, returned "command not found";
the correct command was `umount`.

Verified the active options and saved configuration:

```bash
findmnt -no OPTIONS /boot
cat /etc/fstab
```

Confirmed that both used `fmask=0077,dmask=0077`.

Opened the bootloader configuration:

```bash
nano /boot/loader/loader.conf
```

Replaced its contents with:

```text
default arch.conf
timeout 5
editor no
```

This selects the Arch entry by default, displays the boot menu
for five seconds, and disables editing kernel options from
the boot menu.

Created the Arch Linux ARM boot entry:

```bash
nano /boot/loader/entries/arch.conf
```

Added:

```text
title Arch Linux ARM
linux /Image
initrd /initramfs-linux.img
options root=UUID=3074f1ac-f1a2-4a64-b913-083d1726dd0b rw
```

The entry identifies the kernel, initial RAM filesystem, and
root filesystem. The root filesystem is identified by UUID.

Checked the boot entry:

```bash
bootctl --esp-path=/boot list
```

Confirmed that `arch.conf` was recognized as the default entry
with the correct kernel, initramfs, and root UUID. Booting from
the virtual disk will be tested after the remaining setup.

## 23. Create and enable a swap file

Created a 2 GiB swap file:

```bash
dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress
```

Restricted access to root:

```bash
chmod 600 /swapfile
```

Formatted the file as swap and activated it:

```bash
mkswap /swapfile
swapon /swapfile
```

Added an entry to `/etc/fstab` so swap activates automatically
at boot:

```bash
echo '/swapfile none swap defaults 0 0' >> /etc/fstab
```

Verified the active swap and saved configuration:

```bash
swapon --show
tail -n 3 /etc/fstab
```

Confirmed that `/swapfile` was active with a size of 2 GiB and
0 bytes used. The correct swap entry was present in `/etc/fstab`.

## 24. Install the desktop and enable graphical login

Checked the available desktop packages:

```bash
pacman -Sg lxqt
pacman -Si xorg-server sddm breeze-icons
```

Installed LXQt, the Xorg display server, the SDDM login manager,
and Breeze icons:

```bash
pacman -S lxqt xorg-server sddm breeze-icons
```

Selected all 25 LXQt group packages, including Openbox, and
selected noto-fonts as the font provider. Installation completed
successfully. Service reloads were skipped because installation
was running inside the chroot.

Enabled SDDM and configured the system to boot graphically:

```bash
systemctl enable sddm
systemctl set-default graphical.target
```

Verified the configuration and available desktop sessions:

```bash
systemctl is-enabled sddm
systemctl get-default
ls /usr/share/xsessions/
```

The results showed `enabled`, `graphical.target`, and the session
files `lxqt.desktop` and `openbox.desktop`. Graphical login will
be tested after booting from the virtual disk.

## 25. First boot and system verification

Shut down the Archboot environment, cleared the installation ISO
from UTMs CD/DVD drive, and started the VM from its virtual disk.

Selected Arch Linux ARM in the systemd-boot menu. The SDDM login
screen appeared with both user accounts. Selected LXQt Desktop
and successfully logged in as dmytro.

Opened QTerminal and checked the account, shell, and sudo access:

```bash
whoami
echo $SHELL
sudo whoami
```

The results were `dmytro`, `/usr/bin/zsh`, and `root`.
The configured prompt colors also displayed correctly.

The `hostname` command was unavailable, so checked the hostname
using hostnamectl:

```bash
hostnamectl --static
```

Confirmed that the hostname was `arch-dmytro`.

Checked internet connectivity and running services:

```bash
ping -c 3 archlinux.org
systemctl is-active NetworkManager sshd
```

All three ping packets returned with 0% packet loss.
NetworkManager and sshd both reported `active`.

Verified that swap activated after boot:

```bash
swapon --show
```

Confirmed that `/swapfile` was active with a size of 2 GiB.

## 26. Test SSH access for both users

Found the VMs IPv4 address from QTerminal:

```bash
ip -br -4 addr
```

The network interface `enp0s1` was UP with address
`192.168.64.3/24`.

Opened Terminal on the Mac and connected to the VM:

```bash
ssh dmytro@192.168.64.3
```

Accepted the first-connection host key prompt and authenticated
with the dmytro account password.

Verified the remote account and sudo access:

```bash
whoami
sudo whoami
```

The commands returned `dmytro` and `root`.

Disconnected and connected as codi:

```bash
exit
ssh codi@192.168.64.3
```

Authenticated with codis password and tested the configuration:

```bash
whoami
echo $SHELL
sudo whoami
ll
```

Confirmed that codi could log in over SSH, used `/usr/bin/zsh`,
and could run commands through sudo. The colored prompt and
ll alias also worked.
