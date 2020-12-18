1. Download the CRUX ISO image (crux-3.6.iso).
> Donwload link: http://ftp.morpheus.net/pub/linux/crux/

2. Create USB bootable (dd command).
```bash
dd if=crux-3.6.iso of=/dev/sdX # USB path.
```
3. Boot from USB.
4. Create GPT partition.
```bash
parted /dev/nvme0n1

(parted)
mklabel gpt
unit mib

mkpart efi 1 129
mkpart boot 129 385
mkpart lvm 385 -1

quit
```
5. Create PV/VG/LV.
```bash
pvcreate /dev/nvme0n1p3

vgcreate vgzero /dev/nvme0n1p3

lvcreate -L 32GiB -n swap vgzero
lvcreate -L 32GiB -n root vgzero
lvcreate -L 64GiB -n home vgzero
```
6. Create Filesystems.
```bash
mkfs.vfat /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
mkfs.btrfs /dev/mapper/vgzero-root
mkfs.btrfs /dev/mapper/vgzero-home
mkswap /dev/mapper/vgzero-swap
swapon /dev/mapper/vgzero-swap
```
7. Mount /mnt
```bash
mount /dev/mapper/vgzero-root /mnt

mkdir /mnt/{boot,home}
mount /dev/mapper/vgzero-home /mnt/home
mount /dev/nvme0n1p2 /mnt/boot

mkdir /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi
```
> Reference: http://www.nkly.me/blog/2019/install-gentoo/

8. Type **setup** to start the package installation script.
> Note: Make sure to select the **grub2-efi** packages from the **opt** collection during the package selection phase.

9. Type **setup-chroot** for enter to a new system, or follow these steps.
```bash
mount --bind /dev /mnt/dev
mount --bind /tmp /mnt/tmp
mount --bind /run /mnt/run
mount -t proc proc /mnt/proc
mount -t sysfs none /mnt/sys
mount -t devpts -o noexec,nosuid,gid=tty,mode=0620 devpts /mnt/dev/pts
mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars

chroot /mnt /bin/bash
```
10. Config system.
```bash
# set root password
passwd

# fstab
vim /etc/fstab
#------------------------
/dev/mapper/vgzero-root    /            btrfs    noatime,nodiratime    0    1
/dev/mapper/vgzero-swap    none         swap     noatime,nodiratime    0    0
/dev/mapper/vgzero-home    /home        btrfs    noatime,nodiratime    0    2
/dev/nvme0n1p2             /boot        ext4     defaults    0    1
/dev/nvme0n1p1             /boot/efi    vfat     defaults    0    2
devpts    /dev/pts    devpts    noexec,nosuid,gid=tty,mode=0620    0    0
shm       /dev/shm    tmpfs     defaults    0    0
#------------------------

# config rc.conf
vim /etc/rc.conf
#------------------------
FONT=default
KEYMAP=us
TIMEZONE=Asia/Bangkok
HOSTNAME=mouseegg
SYSLOG=sysklogd
SERVICES=(lo net crond sshd)
#------------------------

# config locales
localedef -i en_US -f UTF-8 en_US.UTF-8

# enable network interface
ifconfig --up enp6s0
```
> Reference: https://crux.nu/Main/Handbook3-6#ntoc11

11. Config and build kernel.
```bash
cd /usr/src/linux-5.4.80
make menuconfig
#------------------------
# CONFIG_EFI_PARTITION=y
-*- Enable the block layer  --->
   Partition Types  --->
      [*]   EFI GUID Partition support
#------------------------
# CONFIG_EFIVAR_FS=y
File systems  --->
   Pseudo filesystems  --->
      <*> EFI Variable filesystem
#------------------------
#CONFIG_EFI=y
Processor type and features  --->
   [*] EFI runtime service support
#------------------------
# CONFIG_FB_EFI=y
Device Drivers  --->
   Graphics support  --->
      Frame buffer Devices  --->
         {*} Support for frame buffer devices --->
            [*] EFI-based Framebuffer Support
      Console display driver support  --->
         <*> Framebuffer Console support
         [*]   Map the console to the primary display device
#------------------------
make all
make modules_install
cp arch/x86/boot/bzImage /boot/vmlinuz-5.4.80
cp System.map /boot
```
> Reference: https://crux.nu/Wiki/UEFI

12. Enable 'contrib' collection and update 'ports'
```bash
# enable for ports
cd /etc/ports
mv contrib.rsync.inactive contrib.rsync

# uncomment the line prtdir /usr/ports/contrib in file /etc/prt-get.conf
vim /etc/prt-get.conf
#------------------------
# the following line enables the user maintained contrib collection
prtdir /usr/ports/contrib
#------------------------

# update ports
ports -u
```
> Reference: https://crux.nu/Main/Handbook3-6#ntoc46

13. Install **dracut** and create  new **initramfs**
```bash
# install dracut
prt-get install dracut

# create new initramfs
dracut --force /boot/initramfs-$(uname -r).img
```
> Reference: https://www.admin-magazine.com/Archive/2020/55/Rebuilding-the-Linux-ramdisk

14. Install GRUB.
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=curx
grub-mkconfig -o /boot/grub/grub.cfg
```
> Reference: https://wiki.archlinux.org/index.php/GRUB#UEFI_systems
15. Reboot

#### Situation commands
1. change LV Status from 'NOT available' to 'available'
```bash
lvchange -ay /dev/vgzero/root
```
> reference: https://www.linuxquestions.org/questions/linux-newbie-8/lvm-mount-via-usb-4175530567/