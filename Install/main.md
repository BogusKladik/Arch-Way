# Install

First, let's check the Internet connection, I have a wired connection, so I didn't need to do any additional actions to check the connection

```bash
ping 8.8.8.8
```

You can also check the time synchronization status, but if there is an Internet connection, they should be set automatically

```bash
timedatectl status
```

Next, let's check the boot mode, if there is something here, then it is UEFI. I installed the system in UEFI boot mode

```bash
ls /sys/firmware/efi/efivars
```

Now let's make the disk layout. I had a free space on the disk in advance on which I would install the system. To do this, we use the `cfdisk` utility on the disk in which we want to install Arch Linux. In order to find out which disks are connected to the system, you need to register `lsblk` in the console. In the utility, I made the markup so that 512MB is an EFI partition (I specified the type of EFI partition) and the rest of the space under Linux filesystem
P.S. 512MB is enough for 2 cores of the system, but there is not enough for 2 cores of the system with the nvidia module enabled in mkinitcpio

```bash
cfdisk /dev/sda
```

Then we format the sections. Then we format the sections. I have these sections 6 and 7 respectively

```bash
mkfs.vfat -F32 /dev/sda6
mkfs.btrfs -L "Arch-PC" /dev/sda7
```

Next, we will mount the sections and subvolumes

```bash
mount /dev/sda7 /mnt
cd /mnt
btrfs subvolume create ./@
btrfs subvolume create ./@home
btrfs subvolume create ./@log
btrfs subvolume create ./@pkg
btrfs subvolume create ./@.snapshots
cd ..
umount -R /mnt

mount -o rw,relatime,compress=zstd:2,discard=async,space_cache=v2,subvol=@ /dev/sda7 /mnt

mkdir /mnt/boot
mkdir /mnt/home
mkdir /mnt/var
mkdir /mnt/var/log
mkdir /mnt/var/cache
mkdir /mnt/var/cache/pacman
mkdir /mnt/var/cache/pacman/pkg
mkdir /mnt/.snapshots

mount -o rw,relatime,compress=zstd:2,discard=async,space_cache=v2,subvol=@home /dev/sda7 /mnt/home
mount -o rw,relatime,compress=zstd:2,discard=async,space_cache=v2,subvol=@log /dev/sda7 /mnt/var/log
mount -o rw,relatime,compress=zstd:2,discard=async,space_cache=v2,subvol=@pkg /dev/sda7 /mnt/var/cache/pacman/pkg
mount -o rw,relatime,compress=zstd:2,discard=async,space_cache=v2,subvol=@.snapshots /dev/sda7 /mnt/.snapshots
mount /dev/sda6 /mnt/boot
```

Then we install the initial packages. If you don't have intel then install the appropriate microcode

```bash
pacstrap -K /mnt base base-devel linux linux-zen linux-firmware btrfs-progs dosfstools linux-headers linux-zen-headers intel-ucode iucode-tool archlinux-keyring nano
```

Generate the fstab file and go to the chroot

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

Then we will choose our local time and synchronize it

```bash
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
hwclock --systohc
```

Next, we will change some files

nano /etc/locale.gen

```bash
en_US.UTF-8 UTF-8
ru_RU.UTF-8 UTF-8
```

Then we generate locales

```bash
locale-gen
```

nano /etc/pacman.conf(Here we have to uncomment some properties and add one (ILoveCandy))

```bash
Color
ILoveCandy
VerbosePkgLists
ParallelDownloads = 5

[multilib]
Include = /etc/pacman.d/mirrorlist
```

nano /etc/locale.conf

```bash
LANG=ru_RU.UTF-8
```

nano /etc/vconsole.conf

```bash
KEYMAP=ru
FONT=cyr-sun16
```

nano /etc/hostname

```bash
Arch-PC
```

nano /etc/hosts

```bash
127.0.0.1 localhost
::1 localhost
127.0.1.1 Arch-PC.localdomain Arch-PC
```

Optional: We can update mirrors for faster loading

```bash
reflector --sort rate -l 10 --save /etc/pacman.d/mirrorlist
```

Next, we update databases and packages, as well as install recommended packages for desktop. I have a video card from Nvidia, if you have another one, then install the appropriate driver

```bash
pacman -Syu
pacman -S zram-generator
pacman -S grub efibootmgr grub-btrfs os-prober
pacman -S networkmanager
pacman -S network-manager-applet
pacman -S pipewire pipewire-alsa pipewire-jack pipewire-pulse gst-plugin-pipewire libpulse wireplumber
pacman -S vim openssh htop wget smartmontools xdg-utils
pacman -S dkms
pacman -S xorg-server xorg-xinit nvidia-dkms
pacman -S plasma-meta konsole kwrite dolphin ark sddm sddm-kcm kmix packagekit-qt5 lib32-nvidia-utils plasma-wayland-session egl-wayland
```

Setting the password for root

```bash
passwd
```

Create a user and give him a password

```bash
useradd -mG wheel -s /bin/bash bogus_kladik
passwd bogus_kladik
```

nano /etc/sudoers(Uncomment this line)

```bash
%wheel ALL=(ALL:ALL) ALL
```

Let's launch services for the Internet and for the boot screen

```bash
systemctl enable NetworkManager.service
systemctl enable sddm
```

nano /etc/default/grub(Uncomment this line if we have 2 systems, before that we installed `os-prober`, if you do not have 2 systems, then either do not install this package, or delete it)

```bash
GRUB_DISABLE_OS_PROBER=false
```

Let's create kernel images and install `grub` and create configs

```bash
mkinitcpio -P
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

Exit the chroot, unmount the partitions and restart the system, not forgetting to disconnect the flash drive

```bash
exit
cd ..
umount -R /mnt
reboot
```
