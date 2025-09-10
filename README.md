# Arch Manual Installation

This is a minimal Arch installation for my own setup.

This is for keeping track of the steps I have taken and it is not a replacement for the guide on the ArchWiki ([here](https://wiki.archlinux.org/title/Installation_guide)).

## Get the ISO

1. Install [Ventoy](https://www.ventoy.net/en/doc_start.html) on a USB drive.
2. Get latest [official Arch ISO](https://archlinux.org/download/).
3. Boot with your Ventoy USB device.

## Live ISO environment steps

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
```

### Change root into new system

```bash
arch-chroot /mnt
```

## After arch-chroot into new system

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

### Set locale

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

### Install bootloader (grub)

Install the bootloader, here we use Grub for EFI boot.

```bash
pacman -S --needed grub os-prober efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=archlinux
grub-mkconfig -o /boot/grub/grub.cfg
```

Enable OS Prober by uncommenting the line in `/etc/default/grub`:

```bash
#GRUB_DISABLE_OS_PROBER=false/GRUB_DISABLE_OS_PROBER=false
```

then run the `grub-mkconfig` line again to save changes.

## Exit arch-chroot and reboot into new system

Now you should have a minimal working system, exit out of the system, unmount the partitions and then reboot.

```bash
exit
umount -R /mnt
reboot
```

## Post-install setup

### Connect to internet

Assuming you have successfully booted into new system, start up NetworkManager and connect to the internet.

```bash
sudo systemctl enable NetworkManager.service
sudo systemctl start NetworkManager.service

# Now configure the networking
nmtui
```

### Install AUR helper (yay)

```bash
mkdir ~/repos
cd ~/repos
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
yay -Y --gendb
yay -Syu --devel
yay -Y --devel --save
```

### Install display manager (ly)

```bash
yay -S ly
sudo systemctl enable ly.service
```

### Install fonts

- ttf-jetbrains-mono-nerd
- noto-fonts
- noto-fonts-cjk
- noto-fonts-emoji
- noto-fonts-extra
- adobe-source-code-pro-fonts
- adobe-source-sans-fonts
- adobe-source-serif-fonts
- adobe-source-han-sans-otc-fonts
- adobe-source-han-serif-otc-fonts
- ttf-hanazono
- ttf-liberation
- ttf-dejavu
- ttf-font-awesome

### Install sound (pipewire)

- pipewire
- pipewire-alsa
- pipewire-jack
- pipewire-pulse
- gst-plugin-firmware
- libpulse
- wireplumber

### Install graphics (nvidia)

- nvidia-dkms

Enable `modeset` by adding this line to `/etc/modprobe.d/nvidia.conf`:

```
options nvidia_drm modeset=1
```

Add the following to `MODULES` array in `/etc/mkinitcpio.conf`:

```
MODULES=(... nvidia nvidia_modeset nvidia_uvm nvidia_drm ...)
```

Rebuild the initramfs with `sudo mkinitcpio -P`.

### Install window manager (hyprland)

- hyprland
- uwsm
- alacritty
- wofi

### Clone dotfiles

Then reboot!

### Install software

- xdg-user-dirs
- flatpak

Install the following as flatpaks:

- librewolf
- flatseal






### Install More Useful Stuff

```bash
yay -S --needed man-db tldr baobab bash-completion discord firewalld firefox \
fish fzf gnome-disk-utility gnome-font-viewer gnome-keyring htop neovim \
polkit reflector tlp vlc xdg-user-dirs visual-studio-code-bin octopi spotify

sudo systemctl enable tlp
sudo systemctl enable bluetooth.service
sudo systemctl enable firewalld
```

### Clone Dotfiles

```bash
git clone --bare https://github.com/derryleng/dotfiles.git /home/derry/.dotfiles

# Set dotfiles alias if not done already
# alias dotfiles='git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'

dotfiles config --local status.showUntrackedFiles no
dotfiles checkout
```
