# Arch Manual Installation

This is a minimal Arch installation for my own setup.

This is for keeping track of the steps I have taken and it is not a replacement for the guide on the ArchWiki ([here](https://wiki.archlinux.org/title/Installation_guide)).

## Get the ISO

1. Install [Ventoy](https://www.ventoy.net/en/doc_start.html) on a USB drive.
2. Get latest [official Arch ISO](https://archlinux.org/download/).
3. Boot with your Ventoy USB device.

## Pre-installation

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

### Create partitions

First, you can view current system partitions using:

```bash
lsblk
# OR
fdisk -l
```

Use the fdisk utility to create any required partition (here assuming you are using the disk **sda**):

```bash
fdisk /dev/sda
# Then follow instructions, do not write out using "w" until you are absolutely sure.
```

NB:

- EFI partition size recommended between 512MB to 1GB.
  - If installing alongside another OS (e.g. Windows), you can just use the existing EFI partition.
- Swap partition is optional, size recommended between $\sqrt{\text{RAM Size}}$ to $2 \times \text{RAM Size}$.
- Root partition should take up rest of the disk (however much free space you want or have left).

### Format partitions

```bash
# Assuming you made a new partition sda1 for EFI boot, format it.
# Otherwise, skip this step if using an existing EFI partition (e.g nvme0n1p1).
mkfs.fat -F32 /dev/sda1

# Assuming you made a new partition sda2 for swap, format it.
# Otherwise, skip this step if not using a swap (or have an existing swap partition).
mkswap /dev/sda2

# Assuming your root partition is sda3, format as BTRFS
mkfs.btrfs /dev/sda3
```

### Mount partitions

```bash
# Mount the swap
swapon /dev/sda2

# Mount the root partition to /mnt
mount /dev/sda3 /mnt

# Mount the boot partition to /mnt/boot/efi
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
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

### Install base packages (pacstrap)

Install some essential packages using pacstrap:

- Base and base development packages
- Linux kernel and header files
- Linux firmware
- Intel and AMD CPU microcode
- Basic text editors (`vi` and `nano`)
- Networking (using `NetworkManager`)
- Git
- Userspace utilities for file systems (`btrfs-progs`, `exfatprogs`, `e2fsprogs`, `ntfs-3g`)

```bash
pacstrap /mnt base base-devel linux linux-headers linux-firmware intel-ucode amd-ucode vi nano networkmanager git btrfs-progs exfatprogs e2fsprogs ntfs-3g
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
blkid | grep 'UUID='

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

> You should now be root user in the new system.

### Update user account settings

Update the password for root user.

```bash
passwd
```

Create a new user account (called user) to wheel group, then update the password.

```bash
useradd -m -G wheel -s /bin/bash user
passwd user
```

Edit `/etc/sudoers` to allow wheel group to execute any command, by uncommenting this line:

```bash
# %wheel ALL=(ALL:ALL) ALL
```

(Optional) Edit `/etc/sudoers` to allow non-sudo use of shutdown, reboot, mount, umount, by updating this line:

```bash
# %wheel ALL=(ALL:ALL) NOPASSWD: ALL
```

to

```bash
%wheel ALL=(ALL:ALL) NOPASSWD: /usr/bin/shutdown,/usr/bin/reboot,/sbin/mount,/sbin/umount
```

### Set clock and locale

```bash
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```

Uncomment the correct line in `/etc/locale.gen`:

```bash
#en_GB.UTF-8 UTF-8
```

then generate the locales.

```bash
locale-gen
```

### Set language

```bash
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
```

### Set keyboard mapping

```bash
echo "KEYMAP=uk" > /etc/vconsole.conf
```

### Set the hostname

```bash
echo "myhostname" > /etc/hostname
```

## Install AUR helper (yay)

> Switch to the new sudo user from here onwards.

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

## Install bootloader (rEFInd with Secure Boot using shim)

Install required packages, install rEFInd, and sign the kernel with the local key that was created. ([Reference](https://wiki.archlinux.org/title/REFInd#Using_Machine_Owner_Key))

(Make sure you are still in `user`, not `root`)

```bash
yay -S --needed efibootmgr refind shim-signed sbsigntools

sudo refind-install --shim /usr/share/shim-signed/shimx64.efi --localkeys

sudo sbsign --key /etc/refind.d/keys/refind_local.key --cert /etc/refind.d/keys/refind_local.crt --output /boot/vmlinuz-linux /boot/vmlinuz-linux
```

Automate the updating of rEFInd files in the EFI system partition (ESP) by creating a pacman hook: ([Reference](https://wiki.archlinux.org/title/REFInd#Upgrading))

`/etc/pacman.d/hooks/refind.hook`
```
[Trigger]
Operation=Upgrade
Type=Package
Target=refind

[Action]
Description = Updating rEFInd on ESP
When=PostTransaction
Exec=/usr/bin/refind-install
```

## Exit chroot and reboot

Now you should have a minimal working system, exit out of the system, unmount the partitions and then reboot.

```bash
exit
umount -R /mnt
reboot
```

During boot, certify rEFInd boot in the MOK manager utility using the certificate at `/EFI/refind/keys/refind_local.cer`.

## Setup Desktop Environment

### Connect to internet on new system

Assuming you have successfully booted into new system, start up NetworkManager and connect to the internet.

```bash
sudo systemctl enable NetworkManager.service
sudo systemctl start NetworkManager.service

# Now configure the networking
nmtui
```

## Install display manager (Ly)

```bash
yay -S ly
systemctl enable ly.service
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
yay -S --needed nvidia-dkms nvidia-settings
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
