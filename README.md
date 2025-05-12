# Archinstall Guide

## Verify ISO Signature
```bash
gpg --keyserver-options auto-key-retrieve --verify archlinux-2025.05.01-x86_64.iso.sig
```

## Time Setup
```bash
timedatectl set-timezone Europe/Berlin
timedatectl set-ntp true
```

## 1. Create the BIOS boot partition (1 MiB, ef02 ) 
```bash
lsblk
gdisk /dev/sdX
# Command (？ for help): n
# Partition number (1-128, default 1): 1 
# First sector (default):[Press Enter]
# Last sector: +1M
# Hex code or GUID (default 8300): ef02
# This is required for GRUB to boot from BIOS when using a
```
## 2. Create the root partition (the rest of the disk, 8300
```bash
# Command (? for help): n
# Partition number (2-128, default 2): 2 
# First sector (default): [Press Enter]
# Last sector (default): [Press Enter]
# Hex code or GUID (default 8300):[Press Enter]

# write with w
```

## format partition
```
mkfs.btrfs /dev/sdX2
```

## Mount Filesystems (Btrfs Example)
```bash
mount /dev/sdX2 /mnt
cd /mnt
btrfs subvolume create @
btrfs subvolume create @home
cd -
umount /mnt
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/sda2 /mnt
mkdir -p /mnt/home
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@home /dev/sda2 /mnt/home
```

## Base Installation
```bash
pacstrap -K /mnt base
genfstab -U -p /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

## Locale & Timezone
```bash
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
pacman -Syu neovim
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "KEYMAP=us" >> /etc/vconsole.conf
echo "samsung" >> /etc/hostname
```

## User Setup
```bash
passwd
useradd -m -g users -G wheel test
passwd test
mkdir /etc/sudoers.d
chmod 755 /etc/sudoers.d
echo "test ALL=(ALL) ALL" >> /etc/sudoers.d/test
chmod 0440 /etc/sudoers.d/test
```

## Networking & Tools
```bash
pacman -Syu sudo reflector rsync
sudo reflector -c Germany -a 12 --sort rate --save /etc/pacman.d/mirrorlist
```

## Base System Packages
```bash
pacman -Syu base-devel linux linux-headers linux-firmware btrfs-progs grub \
mtools networkmanager network-manager-applet openssh sudo neovim git iptables \
ipset ufw reflector acpid grub-btrfs
```

## Documentation & UI
```bash
pacman -S man-db man-pages texinfo bluez bluez-utils pipewire alsa-utils \
pipewire-pulse pipewire-jack sof-firmware ttf-firacode-nerd kitty firefox xdg-user-dirs
```

## Microcode
```bash
pacman -S intel-ucode
```

## Initramfs
```bash
nvim /etc/mkinitcpio.conf
# Add: MODULES=(btrfs atkbd)
mkinitcpio -p linux
```

## GRUB Bootloader
```bash
grub-install --target=i386-pc --boot-directory=/boot /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```

## Enable Services
```bash
systemctl enable NetworkManager
systemctl enable sshd
systemctl enable ufw
systemctl enable reflector.timer
systemctl enable acpid
systemctl enable fstrim.timer
```

## Exit Chroot and Poweroff
```bash
exit
poweroff
```

## AUR Helper (Paru)
```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

## Timeshift Snapshots
```bash
paru -S timeshift timeshift-autosnap
sudo timeshift --list-devices
sudo timeshift --create --comments "[2025-05-10] Initial Snapshot" --tags D
```

## User Setup
```bash
su
cd ~
echo "export EDITOR=nvim" > .bashrc
source .bashrc
systemctl edit --full grub-btrfsd
# change this: ExecStart=/usr/bin/grub-btrfsd --syslog /.snapshots
# to: ExecStart=/usr/bin/grub-btrfsd --syslog -t
grub-mkconfig -o /boot/grub/grub.cfg
```

## ZRAM Swap
```bash
pacman -S zram-generator
mkdir -p /etc/systemd/zram-generator.conf.d/
nvim /etc/systemd/zram-generator.conf.d/zram.conf
```

Contents of `zram.conf`:
```
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
```

```bash
systemctl daemon-reexec
systemctl start /dev/zram0
reboot
```

## Login Manager & Desktop
```bash
sudo pacman -S ly
systemctl enable ly
systemctl start ly
sudo pacman -S cosmic
```

## Firewall for KDE Connect
[Allow KDE Connect Through Firewall](https://www.incredigeek.com/home/allow-kde-connect-through-firewall/)
```
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw limit 22/tcp
sudo ufw allow 1714:1764/udp
sudo ufw allow 1714:1764/tcp
sudo ufw enable
sudo ufw status numbered
```
