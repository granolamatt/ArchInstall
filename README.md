# ArchInstall
Arch Linux install procedure for lvm install
with help from https://gilyes.com/Installing-Arch-Linux-with-LVM/ and https://wiki.archlinux.org/index.php/User:Soloturn/Quick_Installation_guide_UEFI

## Update the system clock
```
timedatectl set-ntp true
```

## Set up LVM
To view details about the available storage devices:
```
lsblk
```
## Prepare the disk
Using gparted:
- Create a GPT Partition
- Create a FAT32 EFI partition 1 GB label EFI
- Linux LVM partition spanning the rest of the disk label vg1

### Create an LVM physical volume on the new partition:

```
pvcreate /dev/sdX2
```

### Create a volume group:
```
vgcreate vg1 /dev/sdX2
```

### Create logical volumes for boot, root, swap and home:

```
lvcreate -L 600G -n root vg1
lvcreate -L 4G -n swap vg1
```

### Format the logical volumes:

```
mkfs.ext4 /dev/vg1/root
mkswap /dev/vg1/swap
swapon /dev/vg1/swap
```

### Then finally mount them under the live system’s /mnt:

```
mount /dev/vg1/root /mnt
mkdir /mnt/esp
mount /dev/sdX1 /mnt/esp
mkdir -p /mnt/esp/arch
mkdir -p /mnt/esp/EFI/Boot
mkdir /mnt/boot
mount --bind /mnt/esp/arch /mnt/boot
```

## Install base packages
```
pacstrap /mnt base base-devel
```
## Generate fstab
```
genfstab -U /mnt >> /mnt/etc/fstab
```
## Change root to new system
```
arch-chroot /mnt /bin/bash
```
## Set up locale
### In /etc/locale.gen uncomment en_US.UTF-8 UTF-8 or whatever else locale you need then generate the new locale(s):

```
LANG=C perl -i -pe 's/#(en_US.UTF)/$1/' /etc/locale.gen
locale-gen
localectl set-locale LANG="en_US.UTF-8"
```
## Change hostname
```
echo myhostname > /etc/hostname
```
## Add multilib, in /etc/pacman.conf uncomment:
```
multilib]
Include = /etc/pacman.d/mirrorlist
```
### Update pacman
```
pacman -Syu
```

## Add dialog and refind-efi
```
pacman -S dialog wpa_supplicant refind-efi
```

## Setup refind for uefi booting
```
cp /usr/share/refind/refind_x64.efi /esp/EFI/Boot/bootx64.efi
cp -r /usr/share/refind/drivers_x64/ /esp/EFI/Boot/
```
## Edit /esp/EFI/Boot/refind.conf
```
menuentry "Arch Linux" {
        icon     EFI/refind/icons/os_arch.png
        volume   "esp"
        loader   /arch/vmlinuz-linux
        initrd   /arch/initramfs-linux.img
        options  "dolvm root=/dev/mapper/vg1-root ro"
}
```

## Change root password
```
passwd
```

## Add a user account
```
useradd -m -G wheel -s /bin/bash archie
passwd archie
```

## Update initramfs
### Add LVM2 hook to /etc/mkinitcpio.conf:

#### /etc/mkinitcpio.conf
```
HOOKS="...lvm2 filesystems..."
```
### Re-generate the initramfs image:
```
mkinitcpio -p linux
```

## To Install packages from another system
### On other syste:
```
pacman -Qqe | grep -vx "$(pacman -Qqm)" > Packages
```

### On new system in chroot
```
xargs -a Packages pacman -S --noconfirm --needed
```

