# Installing Gentoo Linux

This tutorial provides instructions for installing Gentoo Linux. The instructions use the simplest installation paths with a minimal set of tools to speed up initial builds. Complete information about all possible installation options and installation modifications can be found in [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation) on the official site.

## Preparation

First, you need to [download Minimal Installation CD](https://www.gentoo.org/downloads/) install and boot from it.

### Preparing to Work via SSH Server

1. For convenience, you need to raise the SSH service `/etc/init.d/sshd restart` and change the password for the root user `passwd`.
2. Next, find out the IP address, machine `ip a` and connect to it via SSH `ssh roo@ip`.
3. Check the network `ping -c3 ya.ru`

---

#### Disc Preparation
Let's see what disks we have `lsblk`
```
NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0   7:0    0 382.4M  1 loop /mnt/livecd
sda     8:0    0    25G  0 disk
sr0    11:0    1   433M  0 rom  /mnt/cdrom
```
- Prepare disk `cfdisk /dev/sda`.
- Create a 1 GB boot sector and partition the rest of the space.
- Run `sync`.
- Check that all went well `lsblk`.

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 382.4M  1 loop /mnt/livecd
sda      8:0    0    25G  0 disk
├─sda1   8:1    0     1G  0 part
├─sda2   8:2    0     4G  0 part
└─sda3   8:3    0    20G  0 part
sr0     11:0    1   433M  0 rom  /mnt/cdrom
```

- Format the `mkfs.ext2 /dev/sda1` boot sector
- ormat the main disk partition `mkfs.ext2 /dev/sda3`.
- Initialize the swap: `mkswap /dev/sda2` and turn it on `swapon /dev/sda2`.
- Mount the root partition `mount /dev/sda3 /mnt/gentoo`.
- Download the stage3 archive: 

```
wget https://mirror.yandex.ru/gentoo-distfiles/releases/amd64/autobuilds/current-stage3-amd64/stage3-amd64-20210630T214504Z.tar.xz -P /mnt/gentoo
```
9. Unpack the image `tar xpf stage3-amd64-20210630T214504Z.tar.xz -C .`

## Base Configuration

Let's do the initial configuration and set the compilation parameters: `nano -w /mnt/gentoo/etc/portage/make.conf`

The final configuration file looks like this: 
```config
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
COMMON_FLAGS="-O2 -pipe -mtune=native"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
MAKEOPTS="-j2"
ACCEPT_LICENSE="*"
USE="-systemd static-libs -perl udev bash-completion"

# NOTE: This stage was built with the bindist Use flag enabled
PORTDIR="/var/db/repos/gentoo"
DISTDIR="/var/cache/distfiles"
PKGDIR="/var/cache/binpkgs"

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=ru_RU.UTF8

```

#### Setting up mirrors (optional)

Choose the nearest mirror so that the loading speed will be faster: `mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf`

Configure the repository by creating a directory `mkdir --parents /mnt/gentoo/etc/portage/repos.conf` and copy there the Gentoo configuration file `cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf`

The file `/mnt/gentoo/etc/portage/repos.conf/gentoo.conf` should be something like this:

```config
[DEFAULT]
main-repo = gentoo

[gentoo]
location = /var/db/repos/gentoo
sync-type = rsync
sync-uri = rsync://rsync.gentoo.org/gentoo-portage
auto-sync = yes
sync-rsync-verify-jobs = 1
sync-rsync-verify-metamanifest = yes
sync-rsync-verify-max-age = 24
sync-openpgp-key-path = /usr/share/openpgp-keys/gentoo-release.asc
sync-openpgp-keyserver = hkps://keys.gentoo.org
sync-openpgp-key-refresh-retry-count = 40
sync-openpgp-key-refresh-retry-overall-timeout = 1200
sync-openpgp-key-refresh-retry-delay-exp-base = 2
sync-openpgp-key-refresh-retry-delay-max = 60
sync-openpgp-key-refresh-retry-delay-mult = 4
sync-webrsync-verify-signature = yes
```

#### Setting up DNS

Copy the DNS information: `cp --dereference /etc/resolv.conf /mnt/gentoo/etc/`

Mount the necessary file systems: 

```bash
mount --types proc /proc /mnt/gentoo/proc \
&& mount --rbind /sys /mnt/gentoo/sys \
&& mount --rbind /dev /mnt/gentoo/dev

```

#### Isolated Evironment

It is now necessary to switch to the isolated environment:
```bash
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

- Mount the boot partition: `mount /dev/sda1 /boot`.
- Set up Portage: `emerge-webrsync`
- Gentoo news is updated more often than the handbook. To set up the system correctly and not to miss anything, it is recommended to read the news list: `eselect news list` and `eselect news read`
- If required, a profile must be selected. To see a suitable profile use `eselect profile list` and if necessary change it with the command `eselect profile set XX`, where `XX` is the number of the profile in the list.

- You need to update @world so that the base part of the system changes. `emerge --ask --verbose --update --deep --newuse @world`

### Recommended Configurations

The most important configuration is ***USE***. In this variable it is possible to specify keywords that affect the parameters of the assembly. Values with the flag "-" will be excluded from the build. A list of available flags can be seen with `emerge --info | grep ^USE`.


## Kernel Build
Let's build the kernel. Since this tutorial is speed oriented, we will use the autotuning method with genkernel.

- Add the kernel code: `emerge --ask sys-kernel/gentoo-sources`.
- Then install the package `emerge --ask sys-kernel/genkernel`.
- Make sure that the image is in the right directory: `ls -l /usr/src/linux
- In the directory `/usr/src/` rename the file this way: `mv linux{-5.10.61-gentoo,} .`


Then, edit `nano -w /etc/fstab` where the boot line should contain the necessary device. At the same time we specify the other devices.

```text
/dev/sda1   /boot        ext2    defaults,noatime     0 2
/dev/sda2   none         swap    sw                   0 0
/dev/sda3   /            ext4    noatime              0 1
  
/dev/cdrom  /mnt/cdrom   auto    noauto,user          0 0
```

Then run `genkernel all`.

#### Hostname Setup
Set hostname `nano -w /etc/conf.d/hostname`

```text
hostname="tux"
```

#### Network Setup
Set up the network with the command `emerge --ask --noreplace net-misc/netifrc` and add an automatic start of the network connection when the system starts up

```bash
cd /etc/init.d
ln -s net.lo net.eth0
rc-update add net.eth0 default
```

Change the hosts file `nano -w /etc/hosts`
```text
# These are the mandatory settings for the current system
127.0.0.1 tux.homenetwork tux localhost
```

Set the password for the root user `passwd`.

Make the necessary settings for OpenRC: 
```bash
nano -w /etc/rc.conf
nano -w /etc/conf.d/keymaps
nano -w /etc/conf.d/hwclock
```


### Configuring the System Log

To configure the system log, you must install the package and add it to the default startup level.

```bash
emerge --ask app-admin/sysklogd
rc-update add sysklogd default
```

### Configuring SSHD

Add sshd: `rc-update add sshd default` and uncomment the "serial console" section: `nano -w /etc/inittab`

```text
# SERIAL CONSOLES
s0:12345:respawn:/sbin/agetty 9600 ttyS0 vt100
s1:12345:respawn:/sbin/agetty 9600 ttyS1 vt100
```

## Installing the GRUB2 Boot Loader

- Add a boot loader `emerge --ask --verbose sys-boot/grub:2`
- Install it on the disk: `grub-install /dev/sda`
- Must generate a configuration file `grub-mkconfig -o /boot/grub/grub.cfg`


## Finish Installation

Exit the isolated environment and unmount all mounted partitions. 

```
exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot
```

## Additional Actions

#### Adding a non-root User

Log in as root user

```
useradd -m -G users,wheel,audio -s /bin/bash larry
passwd larry
```

#### Deleting Used Files

After the installation there is a system archive left in the system, let's remove it: `rm /stage3-*.tar.*`



