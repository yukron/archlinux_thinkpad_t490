# Arch on Thinkpad T490 (BTRFS + encryption + dual-boot)

**Note:** full commands list is located in the end of document.

Was tested on:
- Thinkpad T490 20N2000KRT (Core i7 8565U, 16GB RAM, 512GB SSD, Intel UHD 620).
- Thinkpad L390
- Thinkpad T480s

## Prerequisites
1. USB Drive (8GB+)
2. You know what you are doing and have some skills

This guide assumes you got yourself a brand new laptop with Windows 10 installed and want to dual-boot it with Arch Linux.

### 1. Shrink NTFS partition
Laptop comes with NVMe SSD varying from 256GB to 1TB, I have decided to limit Windows 10 to ~100GB and allocate other ~400GB to Arch Linux:

1. Login to Windows 10
2. Launch Disk management tool:

    Start -> type in `diskmgmt.msc`
3. Select largest partition and Shrink volume via right-click menu to around 80-100GB, which is more than enough to handle everyday office tasks (to be sincere, I only use Windows to update Lenovo firmwares/BIOS).

You may notice that the last partition is occupied by WIN_DRV volume, which I am going to move before the freed space in the next step. That is not required but I prefer to group Windows/Linux partitions.

### 2. Move WIN_DRV partition (Optional)
Get SystemRescueCD (http://www.system-rescue-cd.org/Download/), and write it onto you USB drive with Rufus (https://rufus.ie/).

1. Reboot laptop and use F12 to load boot menu
2. Boot from your USB drive with SystemRescueCD
3. When booted, execute `startx`
4. Find `Gparted` graphical utility in applications and launch it
5. Move WIN_DRV partition right after Windows 10 partition (and before free space).
6. Check everything, then hit `Apply`.

Now to the Arch installation.

### 3. Arch installation
Get Arch ISO (https://www.archlinux.org/download/), and write it on your USB drive with Rufus (https://rufus.ie/).

#### Boot USB
1. Reboot laptop and use F12 to load boot menu
2. Boot from your USB drive with Arch Linux

We are going to install everything from remote shell so you can copy-paste (with caution) all listed commands. Easiest way to have `bash` on you Windows 10 is to install Git for Windows: https://git-scm.com/download/win.

#### Configure remote access
On Arch now, start `ssh` daemon, set root password and setup networking:
```
passw
systemctl start sshd
wifi-menu
```

After Wi-Fi setup check you IP address via `ip a` command:
```
$ ip a
4: wlp0s20f3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.67/24 brd 192.168.1.255 scope global noprefixroute wlp0s20f3
       valid_lft forever preferred_lft forever
    inet6 xxxx::xxxx:xxxx:xxxx:xxxx/64 scope link
       valid_lft forever preferred_lft forever
```

#### Partition NVMe drive
Target partitions setup will look like this:
```
/dev/nvme0n1p1 (260M)     EFI
/dev/nvme0n1p2 (16M)      Microsoft reserved
/dev/nvme0n1p3 (100.6G)   Microsoft basic data (system)
/dev/nvme0n1p4 (1000M)    Lenovo recovery

/dev/nvme0n1p5 (16.5G)    Linux swap
/dev/nvme0n1p6 (385G)     Linux
```
Use `cfdisk /dev/nvme0n1` or `parted /dev/nvme0n1` to add linux partitions.

##### Use UUIDs
Note: though using old-school device names (e.g. /dev/nvme0n1) could look OK for most users, we are going to use UUIDs.
It can be useful in case you are installing Arch on external USB drive (or, like in my situation, if your Winows 10 decided it's time to upgrade from 1809 to 1903 and add new partition, which shifts device partitions numbers).

Get UUIDs:
`blkid`
```
/dev/nvme0n1: PTUUID="48f81834-1786-45cb-81c9-723157985d2b" PTTYPE="gpt"
/dev/nvme0n1p1: LABEL="SYSTEM" UUID="5052-19C4" TYPE="vfat" PARTLABEL="EFI system partition" PARTUUID="e8559af3-ed5e-4f88-ac3a-be34cafff1d9"
/dev/nvme0n1p2: PARTLABEL="Microsoft reserved partition" PARTUUID="37c6b363-727f-4d57-8e73-25eac91b393d"
/dev/nvme0n1p3: TYPE="BitLocker" PARTLABEL="Basic data partition" PARTUUID="ef301617-c94c-4a55-8577-97e43b72eb58"
/dev/nvme0n1p4: LABEL="WinRE_DRV" UUID="DA9C54AA9C5482C5" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="ac8be247-ef74-430e-9c02-af9e9ed2f7e0"
/dev/nvme0n1p5: PARTLABEL="nvme_swap" PARTUUID="ad3f7e54-4bf9-4432-be2c-d941d069637b"
/dev/nvme0n1p6: UUID="a252825e-6321-424c-b2bb-04375dff8327" TYPE="crypto_LUKS" PARTLABEL="nvme_system" PARTUUID="7ebc59d4-2081-4d63-ae69-0619233e6ef3"
```

Encrypt & open root partition:   
```
cryptsetup luksFormat --align-payload=8192 -s 256 -c aes-xts-plain64 /dev/nvme0n1p6
cryptsetup open /dev/nvme0n1p6 cryptsystem
```

Encrypt and use swap partition:
```
cryptsetup open --type plain --key-file /dev/urandom /dev/nvme0n1p5 swap
mkswap -L swap /dev/mapper/swap
swapon -L swap
```

Create & mount btrfs:
```
mkfs.btrfs --force --label btrfs_system /dev/mapper/cryptsystem
o=defaults,x-mount.mkdir
o_btrfs=$o,compress=zstd,noatime,space_cache=v2
mount -t btrfs LABEL=btrfs_system /mnt
```

Create btrfs subvolumes:
```
btrfs subvolume create /mnt/root
btrfs subvolume create /mnt/home
btrfs subvolume create /mnt/snapshots
umount -R /mnt
mount -t btrfs -o subvol=root,$o_btrfs LABEL=btrfs_system /mnt
mount -t btrfs -o subvol=home,$o_btrfs LABEL=btrfs_system /mnt/home
mount -t btrfs -o subvol=snapshots,$o_btrfs LABEL=btrfs_system /mnt/.snapshots
```

Mount EFI partition:
```
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

#### Prepare base system
Edit `/etc/pacman.d/mirrorlist` and make sure the nearest mirror is on top of the list.

Install basic packages to root partition, fix and add proper swap partition to `/etc/crypttab` and boot to the base system via systemd:
```
pacstrap /mnt base vim
genfstab -L -p /mnt >> /mnt/etc/fstab
sed -i "s+LABEL=swap+/dev/mapper/swap+" /mnt/etc/fstab
echo "swap /dev/nvme0n1p5 /dev/urandom swap,cipher=aes-cbc-essiv:sha256,size=256" >> /mnt/etc/crypttab
cp /etc/pacman.d/mirrorlist /mnt/etc/pacman.d/mirrorlist
systemd-nspawn -bD /mnt
```

#### Configure system
Setup locale:
```
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
localectl set-locale LANG=en_US.UTF-8
timedatectl set-ntp 1
timedatectl set-timezone Europe/Minsk
pacman -S terminus-font
```

Edit `/etc/vconsole.conf`:

```
LOCALE="en_US.UTF-8"
HARDWARECLOCK="UTC"
TIMEZONE="Europe/Minsk"
KEYMAP="ru"
CONSOLEFONT="cyr-sun16"
CONSOLEMAP=""
USECOLOR="yes"
```

Set hostname:
```
hostnamectl set-hostname YOU_HOSTNAME
```

#### Setup bootloader
Install required packages:
```
pacman -Syu base-devel btrfs-progs intel-ucode
```

Edit `/etc/mkinitcpio.conf` and make sure this listed in MODULES:
```
HOOKS=(base udev autodetect modconf block keyboard keymap encrypt filesystems btrfs)
```

Then run mkinitcpio, set root password and exit systemd:
```
mkinitcpio -p linux
passwd
poweroff
```

Add boot record to EFI. `cryptdevice=UUID=*` is /dev/nvme0n1p6 UUID, which can be seen by running `blkid` command.
```
efibootmgr -v
efibootmgr -d /dev/nvme0n1 -p 1 -c -L "Arch Linux" -l /vmlinuz-linux -u "cryptdevice=UUID=your_uuid_goes_here:cryptsystem root=/dev/mapper/cryptsystem rw rootflags=subvol=root initrd=/intel-ucode.img initrd=/initramfs-linux.img"
reboot
```

System is ready to boot. Login with root and password set before, then you can start using (sort of, it's still has almost no packages installed). Proceed next to install basic software, KDE Plasma and other additional applications.

#### Install packages
Connect to wired or wireless network:
```
wifi-menu
dhcpcd
```

Uncomment Color and multilib repos in `/etc/pacman.conf`:
```
Color
...
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Configure `visudo`, uncommenting line `%wheel ALL=(ALL) NOPASSWD: ALL`. With it you won't need to enter your password each time you run commands with `sudo`:
```
visudo
```

Create user, add to sudoers and set password.
```
useradd -m YOURNAME
usermod -aG wheel YOURNAME
passwd YOURNAME
```

Install openssh ans start it:
```
pacman -Syyu
pacman -S openssh
systemctl enable sshd && systemctl start sshd
```

You can now connect to your Thinkpad via `ssh` and proceed:
```
# in case you don't have it already, generate ssh keypair:
# ssh-keygen
ssh-copy-id YOURNAME@THINKPAD_ADDRESS
# enter user password, now you can connect with your key
ssh YOURNAME@THINKPAD_ADDRESS
```

Install various useful utils:
```
sudo pacman -S python python pip mc zip p7zip unrar htop lsof strace git git-lfs mercurial
```

Get `yay` - AUR/pacman helper written in Go:
```
mkdir git && cd git
git clone https://aur.archlinux.org/yay.git
cd yay && makepkg -si
yay -Syyu
```
