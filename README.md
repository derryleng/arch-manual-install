# Arch Manual Installation

> See my alternative scripted installation [here](https://github.com/derryleng/arch-scripted-install).

This is a minimal Arch installation for my own setup. I wrote this to keep track of the steps I have taken and it is definitely not a replacement for the far more detailed and comprehensive guide on the ArchWiki (which can be found [here](https://wiki.archlinux.org/title/Installation_guide)).

## Table of Contents

- [Get the ISO](#get-the-iso)
- [Install Arch](#install-arch)
  - [Configure keyboard layout](#configure-keyboard-layout)
  - [Connect to the internet (Wi-Fi)](#connect-to-the-internet-wi-fi)
  - [Update system clock](#update-system-clock)
  - [Create partitions](#create-partitions)
  - [Format partitions](#format-partitions)
  - [Mount partitions](#mount-partitions)
  - [Update mirrors and package lists](#update-mirrors-and-package-lists)
  - [Install base packages](#install-base-packages)
  - [Generate fstab file](#generate-fstab-file)
  - [Change root into new system](#change-root-into-new-system)
- [Finish Setting Up](#finish-setting-up)
  - [Update user account settings](#update-user-account-settings)
  - [Adjust clock, locale, language, keyboard layout](#adjust-clock-locale-language-keyboard-layout)
  - [Set the hostname](#set-the-hostname)
  - [Install bootloader (grub)](#install-bootloader-grub)
  - [Exit chroot and reboot](#exit-chroot-and-reboot)
  - [Connect to internet on new system](#connect-to-internet-on-new-system)
- [Install AUR helper (yay)](#install-aur-helper-yay)
- [Install Display manager](#install-display-manager)
- [Install Fonts](#install-fonts)
- [Install Sound (pipewire)](#install-sound-pipewire)
- [Install Graphics drivers (nvidia)](#install-graphics-drivers-nvidia)
- [Install WM](#install-wm)
  - [awesome](#awesome)
  - [bspwm](#bspwm)
- [Install Themes](#install-themes)
- [Install More Useful Stuff](#install-more-useful-stuff)
- [Clone Dotfiles](#clone-dotfiles)

## Get the ISO

1. Download latest [official Arch ISO](https://archlinux.org/download/).
2. Use [Ventoy](https://www.ventoy.net/en/doc_start.html).
3. Boot with your Ventoy USB device.

## Install Arch

### Configure keyboard layout

```bash
loadkeys uk
```

### Connect to the internet (Wi-Fi)

```bash
iwctl
```

Then in the iwctl prompt:

```bash
station wlan0 scan
station wlan0 get-networks
station wlan0 connect ...YOUR_SSID...
```

### Update system clock

```bash
timedatectl set-timezone "Europe/London"
timedatectl set-ntp true
```

### Create partitions

First, you can view current system partitions using:

```bash
lsblk
# OR
fdisk -l
```

Use the fdisk utility to create any required partition (here assuming you are using the disk **nvme1n1**):

```bash
fdisk /dev/nvme1n1
# Then follow instructions, do not write out using "w" until you are absolutely sure.
```

NB:

- EFI partition size recommended between 512MB to 1GB.
  - If installing alongside another OS (e.g. Windows), you can just use the existing EFI partition.
- Swap partition is optional, size recommended between $\sqrt{\text{RAM Size}}$ to $2 \times \text{RAM Size}$.
- Root partition should take up rest of the disk (however much free space you want or have left).

### Format partitions

```bash
# Assuming you made a new partition nvme1n1p1 for EFI boot, format it.
# Otherwise, skip this step if using an existing EFI partition (e.g nvme0n1p1).
mkfs.fat -F32 /dev/nvme1n1p1

# Assuming you made a new partition nvme1n1p2 for swap, format it.
# Otherwise, skip this step if not using a swap (or have an existing swap partition).
mkswap /dev/nvme1n1p2

# Assuming your root partition is nvme1n1p3, format it.
# Commonly used filesystem is Ext4:
mkfs.ext4 /dev/nvme1n1p3
# Or you can use BTRFS:
mkfs.btrfs /dev/nvme1n1p3
```

### Mount partitions

```bash
# Mount the swap
swapon /dev/nvme1n1p2

# Mount the root partition to /mnt
mount /dev/nvme1n1p3 /mnt

# Mount the boot partition to /mnt/boot/efi
mkdir -p /mnt/boot/efi
mount /dev/nvme1n1p1 /mnt/boot/efi
```

### Update mirrors and package lists

Use reflector to update the fastest mirrors for pacman.

```bash
reflector --country GB --protocol https --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
```

Make sure package lists are up to date.

```bash
pacman -Sy
```

### Install base packages

Install some essential packages using pacstrap:

- Base and base development packages
- Linux kernel and header files
- Linux firmware
- Intel microcode (for Intel CPU)
- Basic text editors (vi and nano)
- Networking (using NetworkManager)
- Git

```bash
pacstrap /mnt base base-devel linux linux-headers linux-firmware intel-ucode vi nano networkmanager git
```

### Generate fstab file

```bash
genfstab -U /mnt >> /mnt/etc/fstab

# Have a look and check for errors.
nano /mnt/etc/fstab
```

If you want to mount some windows disks on startup:

```bash
# Find the UUID of the partitions you need to mount
blkid | grep ' UUID='

# Make some mount directories
mkdir -p /mnt/windows/c
mkdir -p /mnt/windows/d
# etc...

# Edit fstab
nano /etc/fstab
# Add new lines e.g.
#
# UUID=OUR_C_DRIVE_UUID /mnt/windows/c ntfs-3g defaults,nls=utf8,umask=000,dmask=027,fmask=137,uid=1000,gid=1000,windows_names 0 0
#
# and etc for the other drives.
```

### Change root into new system

```bash
arch-chroot /mnt
```

**You should now be root user in the new system.**

## Finish Setting Up

### Update user account settings

Update the password for root user.

```bash
passwd
```

Create a new user account and then add them to sudoers.

```bash
useradd -m -G wheel -s /bin/bash derry
passwd derry
sed -i 's/# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/' /etc/sudoers
```

(Optional) Edit /etc/sudoers to allow non-sudo use of shutdown, reboot, mount, umount.

```bash
sed -i 's+# %wheel ALL=(ALL:ALL) NOPASSWD: ALL+%wheel ALL=(ALL:ALL) NOPASSWD: /usr/bin/shutdown,/usr/bin/reboot,/sbin/mount,/sbin/umount+' /etc/sudoers
```

### Adjust clock, locale, language, keyboard layout

```bash
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
hwclock --systohc
```

Uncomment the correct line in /etc/locale.gen and generate the locales.

```bash
sed -i 's/#en_GB.UTF-8 UTF-8/en_GB.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
```

Set language and keyboard mapping.

```bash
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
echo "KEYMAP=uk" > /etc/vconsole.conf
```

Install and enable ntp.

```bash
pacman -S ntp
systemctl enable ntpd.service
```

### Set the hostname

```bash
echo "myhostname" > /etc/hostname
```

### Install bootloader (grub)

Install the bootloader, here we use Grub for EFI boot.

You can also use os-prober to detect other bootloaders - skip the last two steps if you don't need this.

```bash
pacman -S --needed grub os-prober efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=archlinux
grub-mkconfig -o /boot/grub/grub.cfg

# Uncomment os-prober in grub configs
sed -i 's/#GRUB_DISABLE_OS_PROBER=false/GRUB_DISABLE_OS_PROBER=false/' /etc/default/grub

# Create the update-grub command
touch /usr/sbin/update-grub
echo '#!/bin/sh
set -e
exec grub-mkconfig -o /boot/grub/grub.cfg "$@"' > /usr/sbin/update-grub

# Then set file ownership to make it useable:
chown root:root /usr/sbin/update-grub
chmod 755 /usr/sbin/update-grub

# Update grub configs
update-grub
```

### Exit chroot and reboot

Now you should have a minimal working system, exit out of the system, unmount the partitions and then reboot.

```bash
exit
umount -R /mnt
reboot
```

### Connect to internet on new system

Assuming you have successfully booted into new system, start up NetworkManager and connect to the internet.

```bash
sudo systemctl enable NetworkManager.service
sudo systemctl start NetworkManager.service

# Now configure the networking
nmtui
```

## Install AUR helper (yay)

```bash
mkdir repos
cd repos
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
yay -Y --gendb
yay -Syu --devel
yay -Y --devel --save
```

See more details here: https://github.com/Jguer/yay

## Install Display manager

```bash
yay -S sddm qt5-quickcontrols2
sudo systemctl enable sddm.service
```

Apply a nice theme

```bash
# Clone and build theme
git clone https://github.com/GistOfSpirit/TerminalStyleLogin
bash TerminalStyleLogin/scripts/build.sh

# Make some edits
sed -i 's/fontSize=[0-9]\+/fontSize=18/' TerminalStyleLogin/theme.conf
sed -i 's/^\(.*{proxy.hostName}.*\)/\/\* \1 \*\//' TerminalStyleLogin/Main.qml

# Copy contents of build folder to /usr/share/sddm/themes/TerminalStyleLogin
mkdir -p /usr/share/sddm/themes/TerminalStyleLogin
cp -r TerminalStyleLogin/build/* /usr/share/sddm/themes/TerminalStyleLogin

# Set sddm theme
touch /etc/sddm.conf
echo "[Theme]
Current=TerminalStyleLogin" > /etc/sddm.conf
```

## Install Fonts

```bash
yay -S adobe-source-code-pro-fonts adobe-source-han-sans-cn-fonts \
adobe-source-han-sans-jp-fonts adobe-source-han-sans-kr-fonts \
ttf-jetbrains-mono-nerd noto-fonts noto-fonts-emoji ttf-font-awesome \
ttf-opensans ttf-dejavu ttf-liberation cantarell-fonts
```

## Install Sound (pipewire)

```bash
yay -S pipewire pipewire-alsa pipewire-jack pipewire-pulse \
gst-plugin-firmware libpulse wireplumber pavucontrol
```

## Install Graphics drivers (nvidia)

```bash
yay -S --needed nvidia nvidia-settings
```

## Install WM

```bash
yay -S --needed xorg-server xbindkeys xclip xdo xorg-xbacklight xorg-xdpyinfo  \
xorg-xinit xorg-xinput xorg-xkill xorg-xrandr xorg-xsetroot archlinux-xdg-menu \
gvfs blueman volumeicon network-manager-applet ibus-pinyin \
picom thunar alacritty rofi dunst
```

### awesome

```bash
yay -S --needed awesome-git
```

### bspwm

```bash
yay -S --needed bspwm sxhkd polybar
```

## Install Themes

```bash
yay -S bibata-cursor-theme-bin arc-gtk-theme-git \
papirus-icon-theme-git python-qdarkstyle qt5ct lxappearance
```

## Install More Useful Stuff

```bash
yay -S --needed man-db tldr baobab bash-completion discord firewalld firefox \
fish fzf gnome-disk-utility gnome-font-viewer gnome-keyring htop neovim \
polkit reflector tlp vlc xdg-user-dirs visual-studio-code-bin octopi spotify

sudo systemctl enable tlp
sudo systemctl enable bluetooth.service
sudo systemctl enable firewalld
```

## Clone Dotfiles

```bash
git clone --bare https://github.com/derryleng/dotfiles.git /home/derry/.dotfiles

# Set dotfiles alias if not done already
# alias dotfiles='git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'

dotfiles config --local status.showUntrackedFiles no
dotfiles checkout
```
