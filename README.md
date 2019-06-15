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
