## Installation ##
This setup will achieve a Gentoo Machine compiled with PGO, LTO and Graphite. It uses UEFI, OpenRC, Grub and a manually configurated Kernel with initrams provided by Dracut.
Also, it is recommended to use a LiveGUI to ease things up.
The steps are:
1. Preparing Disks
2. Installing Stage3
3. Chrooting & Preparing Defaults (locale, profile, timezone)
4. Updating Gentoo ebuild Repository
5. Sorting dev-lang/perl
6. Compiling Dependencies
7. Adding Gentoo Repositories and switching sync to git
8. Adding a default user and passwords
9. Creating mount locations for other disks
10. Editing fstab to include /boot, /, swap, other disks and enable Portage TMPDIR on tmpfs
11. Updating GCC to enable PGO, LTO and Graphite
12. Emerging ltoize to help with LTO packages
13. Rebuilding the world
14. Emerging the Kernel
15. Building the Kernel and initramfs
16. Emerging and configuring GRUB
17. Rebooting the system
18. Emerging Xorg and a Window Manager

In the repo there are files which help with USE flags to be able to build Steam and Wine without having to recompile dependencies to other abis, this way most of the deps required for them are already built with multilib abis.

### 1. Preparing Disks ###
The partition scheme is as following: 
<table>
    <tr>
        <td>Partition</td>
        <td>Filesystem</td>
        <td>Size</td>
        <td>Description</td>
    </tr>
    <tr>
      <td>/dev/nvme0n1p1</td>
      <td>fat</td>
      <td>2MiB</td>
      <td>bios boot</td>
    </tr>
      <tr>
      <td>/dev/nvme0n1p2</td>
      <td>fat32</td>
      <td>800MiB</td>
      <td>boot partition</td>
    </tr>
      <tr>
      <td>/dev/nvme0n1p3</td>
      <td>swap</td>
      <td>~8GB</td>
      <td>swap</td>
    </tr>
      <tr>
      <td>/dev/nvme0n1p4</td>
      <td>ext4</td>
      <td>rest of the disk</td>
      <td>root partition</td>
    </tr>
</table>

<code>livecd ~# parted /dev/nvme0n1</code>

<pre>GNU Parted 3.2
Using /dev/nvme0n1
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpt

(parted) mkpart primary fat32 1MiB 3MiB

(parted) mkpart primary fat32 3MiB 803MiB

(parted) mkpart primary 803MiB 9G

(parted) mkpart primary 9G -1

(parted) name 1 grub                                                      
(parted) name 2 boot                                                      
(parted) name 3 swap
(parted) name 4 gentoo

(parted) set 1 bios_grub on                                               
(parted) set 2 boot on                                                    
(parted) set 3 swap on

(parted) print                                                            
Model: Sabrent (nvme)
Disk /dev/nvme0n1: 512GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system     Name    Flags
 1      1049kB  3146kB  2097kB  fat32           grub    bios_grub
 2      3146kB  842MB   839MB   fat32           boot    boot, esp
 3      842MB   9000MB  8158MB                  swap    swap
 4      9000MB  512GB   503GB                   gentoo

(parted) quit
</pre>

### Creating Filesystems ###
<pre>livecd ~# mkfs.vfat -n "BIOS" /dev/nvme0n1p1 
mkfs.fat 4.1 (2017-01-24)

livecd ~# mkfs.vfat -F32 -n "BOOT" /dev/nvme0n1p2
mkfs.fat 4.1 (2017-01-24)

livecd ~# mkswap -L "swap" /dev/nvme0n1p3

livecd ~# mkfs.ext4 -L "root" /dev/nvme0n1p4</pre>

### Mounting Filesystems ###

OBS: If you are not using Gentoo Livecd, do:

<code>livecd ~# mkdir /mnt/gentoo</code>

Before proceeding.

<pre>livecd ~# mount -v -t ext4 /dev/nvme0n1p4 /mnt/gentoo
livecd ~# swapon -v /dev/nvme0n1p3</pre>

### 2. Installing Stage 3 ###
Choose a autobuild from: http://distfiles.gentoo.org/releases/amd64/autobuilds/, then:

<pre>
livecd ~# cd /mnt/gentoo/
livecd /mnt/gentoo # wget -c http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-openrc/stage3-amd64-openrc-20220904T170535Z.tar.xz
livecd /mnt/gentoo # tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
livecd /mnt/gentoo # rm -v -f stage3-amd64-* 
</pre>

### Configuring make.conf and package.use ###
This is where I will put mine make.conf and also my default package.use settings. It is pretty important to the following values to fit your Machine:
1. CPU_FLAGS_X86
2. MAKE_OPTS
3. USE
4. L10N
5. LINGUAS
6. VIDEO_CARDS

<code>livecd /mnt/gentoo # vi /mnt/gentoo/etc/portage/make.conf</code>
<pre>
#     .vir.                                d$b
#  .d$$$$$$b.    .cd$$b.     .d$$b.   d$$$$$$$$$$$b  .d$$b.      .d$$b.
#  $$$$( )$$$b d$$$()$$$.   d$$$$$$$b Q$$$$$$$P$$$P.$$$$$$$b.  .$$$$$$$b.
#  Q$$$$$$$$$$B$$$$$$$$P"  d$$$PQ$$$$b.   $$$$.   .$$$P' `$$$ .$$$P' `$$$
#    "$$$$$$$P Q$$$$$$$b  d$$$P   Q$$$$b  $$$$b   $$$$b..d$$$ $$$$b..d$$$
#   d$$$$$$P"   "$$$$$$$$ Q$$$     Q$$$$  $$$$$   `Q$$$$$$$P  `Q$$$$$$$P
#  $$$$$$$P       `"""""   ""        ""   Q$$$P     "Q$$$P"     "Q$$$P"
#  `Q$$P"                                  """


#LTO 
#NTHREADS="auto"
#source make.conf.lto

# Compiler Jobs
MAKEOPTS="-j24"

# Compiler Flags
#COMMON_FLAGS="-march=native ${CFLAGS} -pipe"
COMMON_FLAGS="-march=native -pipe -O2"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 sse4a ssse3"

# Features and Defaults
EMERGE_DEFAULT_OPTS="--verbose --quiet-build --keep-going --jobs=24 --load-average=24 --with-bdeps y --complete-graph y"
FEATURES="split-elog parallel-install parallel-fetch"

# Keywords and Licenses
ACCEPT_KEYWORDS="~amd64"
ACCEPT_LICENSE="*"

# USE FLAGS
USE="-systemd -bindist -kde -plasma -wayland -gnome pulseaudio elogind dbus colord git hddtemp v4l vaapi zsh-completion qt pgo lto graphite fontconfig truetype udisks icu lm-sensors networkmanager bluetooth wifi unicode opengl vulkan X gtk nvenc"

# Directories
PORTDIR="/var/db/repos/gentoo"
DISTDIR="/var/cache/distfiles"
PKGDIR="/var/cache/binpkgs"

# Languages
LC_MESSAGES=C
L10N="en en-US pt-BR"
LINGUAS="en en_US pt_BR"

#Logging
PORTAGE_ELOG_CLASSES="info warn error log qa"
PORTAGE_ELOG_SYSTEM="echo save"

# Other
GRUB_PLATFORM="efi-64"
VIDEO_CARDS="nvidia"
CHOST="x86_64-pc-linux-gnu"
INPUT_DEVICES="libinput"
</pre>

For the package.use config, do:

<code>livecd /mnt/gentoo # cp -r /path/to/package.use/ /mnt/gentoo/etc/portage/</code>

Remember to check every file inside the package.use to verify it will fit your Machine.

### Building default Gentoo repo ###

<pre>
livecd /mnt/gentoo # mkdir --parents /mnt/gentoo/etc/portage/repos.conf
livecd /mnt/gentoo # cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
</pre>

### 3. Chrooting & Preparing Defaults ###

<pre>
livecd /mnt/gentoo # cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
livecd /mnt/gentoo # mount -t proc /proc /mnt/gentoo/proc
livecd /mnt/gentoo # mount --rbind /sys /mnt/gentoo/sys
livecd /mnt/gentoo # mount --make-rslave /mnt/gentoo/sys
livecd /mnt/gentoo # mount --rbind /dev /mnt/gentoo/dev
livecd /mnt/gentoo # mount --make-rslave /mnt/gentoo/dev
livecd /mnt/gentoo # mount --bind /run /mnt/gentoo/run
livecd /mnt/gentoo # mount --make-slave /mnt/gentoo/run 
livecd /mnt/gentoo # test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
livecd /mnt/gentoo # mount -t tmpfs -o nosuid,nodev,noexec shm /dev/shm
livecd /mnt/gentoo # chmod 1777 /dev/shm
</pre>

One liner: 

<pre>
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/ && mount -t proc /proc /mnt/gentoo/proc && mount --rbind /sys /mnt/gentoo/sys && mount --make-rslave /mnt/gentoo/sys && mount --rbind /dev /mnt/gentoo/dev && mount --make-rslave /mnt/gentoo/dev && mount --bind /run /mnt/gentoo/run && mount --make-slave /mnt/gentoo/run && test -L /dev/shm && rm /dev/shm && mkdir /dev/shm && mount -t tmpfs -o nosuid,nodev,noexec shm /dev/shm && chmod 1777 /dev/shm
</pre> 


<pre>
livecd /mnt/gentoo # chroot /mnt/gentoo /bin/bash 
livecd / # source /etc/profile
livecd / # export PS1="(chroot) $PS1"
(chroot) livecd / #
</pre>

<code>(chroot) livecd / # mount /dev/nvme0n1p2 /boot</code>

Do a quick emerge update to sort your profile. It will be updated again with git later.

<code>(chroot) livecd / # emerge-webrsync</code>

### Sorting the defaults ###

Firstly, we choose a profile.
<pre>
(chroot) livecd ~# eselect profile list
...
(chroot) livecd ~# eselect profile set 1
(chroot) livecd ~# eselect profile list
Available profile symlink targets:
[1]   default/linux/amd64/17.1 (stable) *
...
</pre>

OBS: I like using the non-desktop profile because it is really minimal. Any additional packages that are required will be built on a @world emerge anyways.

Timezone:
<pre>
(chroot) livecd ~# echo Brazil/East > /etc/timezone
(chroot) livecd ~# emerge --config sys-libs/timezone-data
</pre>

Locales:

Generate them with:
<pre>
(chroot) livecd ~# nano -w /etc/locale.gen 
...
C.UTF-8 UTF-8
en_US.UTF-8 UTF-8
en_US ISO-8859-1
pt_BR.UTF-8 UTF-8
pt_BR ISO-8859-1

(chroot) livecd ~# locale-gen
</pre>

Then choose it to C.

<pre>
(chroot) livecd # eselect locale list
(chroot) livecd # eselect locale set 1
(chroot) livecd # eselect locale list
...
[1]  C.*
</pre>

Now reload the environment with:

<code>(chroot) livecd /etc # env-update && source /etc/profile && export PS1="(chroot) $PS1"</code>

### 4. Compiling Dependencies && Updating Gentoo Repositories ###

This is where things start getting wacky. I like to compile vim, htop, dev-vcs/git, eselect-repository and ntfs3g right of the gate to sort things out. Vim is because I'm too used to it, htop is useful for checking how the system is going, git to sync portage (and other repo's), eselect-repository to enable them and ntfs because I have disks that use them.
Before doing this compiles, I have to either mask perl or update it because git complains about it being out-of-date. Here I will mask it, I will update it later after GCC has LTO.

Masking perl:

<code>(chroot) livecd # echo ">dev-lang/perl-5.34" > /etc/portage/package.mask/perl"</code>

Emerging the packages:

<code>(chroot) livecd # emerge vim htop dev-vcs/git eselect-repository ntfs3g </code>

Now with git installed we can switch Portage to git. Edit the following file to match it:

<pre>(chroot) livecd # vim /etc/portage/repos.conf/gentoo.conf
...

[DEFAULT]
main-repo = gentoo

[gentoo]
location = /var/db/repos/gentoo
sync-type = git
sync-uri = https://github.com/gentoo-mirror/gentoo
auto-sync = yes
sync-git-verify-commit-signature = yes
sync-openpgp-key-path = /usr/share/openpgp-keys/gentoo-release.asc
</pre>

Save and quit it. Now I will enable some repositories with eselect-repository, if you just want LTO overlay, just add mv and lto-overlay. However, here I also enable guru and the thegreatmcpain since I use a lot of their ebuilds.

<code>(chroot) livecd # eselect repository enable mv lto-overlay guru thegreatmcpain</code>

Before we update Gentoo Ebuild Repo, we need to clean /var/db/repos/gentoo. You can do it with:

<code>(chroot) livecd # rm -rf /var/db/repos/gentoo</code>

Now we sync it with:

<code>(chroot) livecd # emerge --sync</code>

### 5.Creating mount locations for other disks & adding a user ###
Creating a user now? Are you absolutely insane? Well, there is a method for my madness. I will edit the fstab to be able to use RAM as a portage TMPDIR, so while I'm at it why not also include my other disks that I want on it? And for doing that, I need to create the mount points on the /media/user/ folder. So getting started:

<code>(chroot) livecd # useradd -m -G audio,cdrom,floppy,games,portage,usb,video,wheel -s /bin/bash gui</code>

<code>(chroot) livecd # passwd gui </code>

While at it, also define root password.

<code>(chroot) livecd # passwd </code>

Now creating the mountpoints for my NTFS disks and a ext4 SSD:
<pre>
(chroot) livecd # mkdir /media/gui/
(chroot) livecd # mkdir /media/gui/{Games,Backups,HD,SSD}
</pre>

### 6. Configuring fstab for Portage TMPDIR on RAM & mountpoints for other disks ###

So here I will setup the fstab. For Portage TMPDIR size, check the default Gentoo wiki page, but my recommendation is to not do it unless you have a fair amount of RAM to do it. Around 16gb is more than enough than to do it globally, however, if you have around 8gb to do it, you will prob want to deactivate it for packages like Firefox and Libreoffice.
A tip, run blkid on another terminal to make it easier to setup fstab.
<pre>
(chroot) livecd # blkid
/dev/sdb2: UUID="67744fc1-7051-4070-81e2-03111fbbc13f" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="a41e0dee-a5ce-4e8b-a5b4-dcc83bd9ab19"
/dev/nvme0n1p1: SEC_TYPE="msdos" LABEL_FATBOOT="GRUB" LABEL="GRUB" UUID="704E-3C4F" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="grub" PARTUUID="a0842a34-4ffe-443e-bcae-3fad255ab72c"
/dev/nvme0n1p2: LABEL_FATBOOT="BOOT" LABEL="BOOT" UUID="6FA2-081B" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="boot" PARTUUID="cabf1221-ea10-4125-8798-c5da8052e3f9"
/dev/sdd1: LABEL="HD" BLOCK_SIZE="512" UUID="AE9C1DE89C1DAC39" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="7b72cd99-a884-4684-b2ce-d9333057f8ea"
/dev/sdb1: UUID="241E-C051" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="14fd1d3f-3bbe-4f77-93a7-d843920d86f4"
/dev/sdc1: LABEL="Jogos" BLOCK_SIZE="512" UUID="746A319B6A315AD6" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="ee7d1813-fc77-4d7d-b584-3f9eabbdcef0"
/dev/sda3: LABEL="Backups" BLOCK_SIZE="512" UUID="1626DA6F26DA4EFD" TYPE="ntfs" PARTLABEL="LDM data partition" PARTUUID="489c7e0f-1ce5-11ed-b365-c8b29bdf1e0b"
/dev/nvme0n1p3: LABEL="swap" UUID="b972d9c5-a585-405f-95ff-f35ee2a99d62" TYPE="swap" PARTLABEL="swap" PARTUUID="3c382b50-38af-4caa-9432-cc80821b1313"
/dev/nvme0n1p4: LABEL="gentoo" UUID="0dd9af81-1547-4a4a-8ad3-848f8d3ee630" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="gentoo" PARTUUID="6bfe92e6-def6-4893-aa1c-f3f2afef72e8"
/dev/sda2: PARTLABEL="Microsoft reserved partition" PARTUUID="489c7e0b-1ce5-11ed-b365-c8b29bdf1e0b"
/dev/sda1: PARTLABEL="LDM metadata partition" PARTUUID="489c7e0a-1ce5-11ed-b365-c8b29bdf1e0b"
</pre>

<pre>
(chroot) livecd # vim /etc/fstab
...

UUID=6FA2-081B                                  /boot                   vfat    noatime                                                         0 1
UUID=0dd9af81-1547-4a4a-8ad3-848f8d3ee630       /                       ext4    defaults,noatime,errors=remount-ro,discard                      0 1
UUID=67744fc1-7051-4070-81e2-03111fbbc13f       /media/gui/SSD          ext4    defaults,noatime,discard                                        0 1
UUID=AE9C1DE89C1DAC39                           /media/gui/HD           ntfs-3g defaults                                                        0 0
UUID=746A319B6A315AD6                           /media/gui/Games        ntfs-3g defaults                                                        0 0
UUID=1626DA6F26DA4EFD                           /media/gui/Backups      ntfs-3g defaults                                                        0 0
UUID=b972d9c5-a585-405f-95ff-f35ee2a99d62       none                    swap    sw,defaults,noatime,discard                                     0 0
tmpfs                                           /var/tmp/portage        tmpfs   size=16G,uid=portage,gid=portage,mode=775,nosuid,noatime,nodev  0 0
</pre>

Now we mount everything using:

<code> (chroot) livecd # mount -a </code>

Check it with:

<code> (chroot) livecd # mount </code>

### 7. Re-compiling GCC to enable builds with PGO, LTO and Graphite ###

Remember to include PGO, LTO, Graphite in your make.conf USE flags. Then:

<code> (chroot) livecd # emerge --ask gcc </code>

After it is finished, if the GCC version was upgraded, do:

<pre>
(chroot) livecd # gcc-config -l
 [1] x86_64-pc-linux-gnu-11.3.0 *
 [2] x86_64-pc-linux-gnu-12.2.0 
 (chroot) livecd # gcc-config 2
  * Switching native-compiler to x86_64-pc-linux-gnu-10.1.0 ...
>>> Regenerating /etc/ld.so.cache...                                                                                                                                                                                                                                     [ ok ]

 * If you intend to use the gcc from the new profile in an already
 * running shell, please remember to do:

 *   . /etc/profile
</pre>

Now, regardless of GCC version upgraded or not, do:

<pre>
(chroot) livecd ~# source /etc/profile
livecd ~# export PS1="(chroot) $PS1"
(chroot) livecd ~# emerge --ask --oneshot --usepkg=n sys-devel/libtool
</pre>

This enables the re-compiled gcc flags and also re-compiles the libtool.

### 7. Emerging ltoize to help with LTO packages ###

Ltoize, from https://github.com/InBetweenNames/gentooLTO, is an overlay that helps performing optimizations and also sets useful flags for packages. I highly recommend reading it to check how it works, explaining it here is beyond the scope of these Instructions.

<code> (chroot) livecd # emerge ltoize </code>

Now we will enable it on our make.conf to make use of it.

<pre>
(chroot) livecd # vim /etc/portage/make.conf
...
#     .vir.                                d$b
#  .d$$$$$$b.    .cd$$b.     .d$$b.   d$$$$$$$$$$$b  .d$$b.      .d$$b.
#  $$$$( )$$$b d$$$()$$$.   d$$$$$$$b Q$$$$$$$P$$$P.$$$$$$$b.  .$$$$$$$b.
#  Q$$$$$$$$$$B$$$$$$$$P"  d$$$PQ$$$$b.   $$$$.   .$$$P' `$$$ .$$$P' `$$$
#    "$$$$$$$P Q$$$$$$$b  d$$$P   Q$$$$b  $$$$b   $$$$b..d$$$ $$$$b..d$$$
#   d$$$$$$P"   "$$$$$$$$ Q$$$     Q$$$$  $$$$$   `Q$$$$$$$P  `Q$$$$$$$P
#  $$$$$$$P       `"""""   ""        ""   Q$$$P     "Q$$$P"     "Q$$$P"
#  `Q$$P"                                  """


#LTO 
NTHREADS="auto"
source /etc/portage/make.conf.lto

# Compiler Jobs
MAKEOPTS="-j24"

# Compiler Flags
COMMON_FLAGS="-march=native ${CFLAGS} -pipe"
#COMMON_FLAGS="-march=native -pipe -O2"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 sse4a ssse3"

# Features and Defaults
EMERGE_DEFAULT_OPTS="--verbose --quiet-build --keep-going --jobs=24 --load-average=24 --with-bdeps y --complete-graph y"
FEATURES="split-elog parallel-install parallel-fetch"

# Keywords and Licenses
ACCEPT_KEYWORDS="~amd64"
ACCEPT_LICENSE="*"

# USE FLAGS
USE="-systemd -bindist -kde -plasma -wayland -gnome pulseaudio elogind dbus colord git hddtemp v4l vaapi zsh-completion qt pgo lto graphite fontconfig truetype udisks icu lm-sensors networkmanager bluetooth wifi unicode opengl vulkan X gtk nvenc"

# Directories
PORTDIR="/var/db/repos/gentoo"
DISTDIR="/var/cache/distfiles"
PKGDIR="/var/cache/binpkgs"

# Languages
LC_MESSAGES=C
L10N="en en-US pt-BR"
LINGUAS="en en_US pt_BR"

#Logging
PORTAGE_ELOG_CLASSES="info warn error log qa"
PORTAGE_ELOG_SYSTEM="echo save"

# Other
GRUB_PLATFORM="efi-64"
VIDEO_CARDS="nvidia"
CHOST="x86_64-pc-linux-gnu"
INPUT_DEVICES="libinput"
</pre>

Add these to your make.conf and edit COMMON_FLAGS line with:

<pre>
NTHREADS="auto"

source /etc/portage/make.conf.lto

COMMON_FLAGS="-march=native ${CFLAGS} -pipe"
</pre>

### 9. Rebuilding the World ###

Finally we will rebuild @world. However, before doing it, we need to unmask dev-lang/perl and update it.

<code> (chroot) livecd # rm /etc/portage/package.mask/perl </code>

To update:

<code> (chroot) livecd # perl-cleaner --reallyall </code>

After that is finished, rebuild world with:

<code> (chroot) livecd # emerge --ask --verbose --update --deep --with-bdeps=y --newuse  --keep-going --backtrack=30 --emptytree --exclude 'gcc' @world</code>

Note that gcc is excluded, since we already compiled it.

After it is finished, lets clean some things up with:

<code> (chroot) livecd # emerge --depclean </code>

### 10. Emerging the Kernel ###

Personally, I like building the Kernel with the experimental use flag. It provides some new features that come in handy with my CPU, however, as everything else in Linux, it is up to you what to choose.

<pre>
(chroot) livecd ~# emerge --ask sys-kernel/gentoo-sources
(chroot) livecd ~# emerge --ask sys-kernel/linux-firmware
</pre>

The linux-firmware is to provide blobs for compatibility with my hardware.

### 11. Building the Kernel and initramfs ###

On this guide, I will be building the kernel manually and using dracut for the initramfs. However, if you want a more automated process, try using genkernel (https://wiki.gentoo.org/wiki/Genkernel).

Either way, you will want to make a symlink. Do it with:

<pre>
(chroot) livecd / # eselect kernel list 
Available kernel symlink targets:
[1]   linux-5.19.6-gentoo
(chroot) livecd / # eselect kernel set
</pre>

Now, I already have a kernel that is pretty minimal and only has the modules that my Machine requires. However, doing it requires a very honest amount of time and I'm not really sure if the results make it worth. You will be reduzing the kernel size on the disk, maybe reduzing your boot time but it won't be that much considering the hardware that is available nowadays.
But, it can also be a nice learning process on how your system works and how it communicates with the kernel. And not to mention it will reduce your kernel compile time by a fair amount. To make the process easier, I recommend using sys-apps/pciutils and using lspci to get some information. Also, sys-kernel/modprobed-db (available on guru repo) will provide you with a list of all the modules that your machine require. You can read more about it on https://wiki.archlinux.org/title/Modprobed-db.

Lets start building the kernel:

<pre>
(chroot) livecd # cd /usr/src/linux
(chroot) livecd /usr/srlinux/ # make menuconfig
</pre>

To actually compile it, where jX is the amount of jobs:

<pre>
(chroot) livecd # make -jX && make -jX modules_install
(chroot) livecd # make install
</pre>

With that out of the way, lets build dracut and create our initramfs.

<pre>
(chroot) livecd # emerge --ask sys-kernel/dracut
(chroot) livecd # dracut --hostonly --kver 5.19.6-gentoo
</pre>

### 12. Emerging and configuring GRUB ###

Things are almost done! We just need a bootloader and sorting a few services to start at boot.

OBS: if you want GRUB to run os-prober and find other OS in your machine, build it with mount useflag and also emerge os-prober.

To compile GRUB:

<code> (chroot) livecd # emerge --ask --verbose sys-boot/grub </code>

If you didn't specify to use UEFI for GRUB in your make.conf, do it with:

<code>(chroot) livecd # echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf </code>

To install GRUB: 

<code>(chroot) livecd # grub-install --target=x86_64-efi --efi-directory=/boot </code>

Make any configurations on your GRUB with:

<code>(chroot) livecd # vim /etc/default/grub </code>

Some noteworthy configs:

<pre>
GRUB_GFXMODE=1920x1080
GRUB_DISABLE_OS_PROBER=false
GRUB_GFXPAYLOAD_LINUX=keep
</pre>


Now make the GRUB config with:

<code>(chroot) livecd # grub-mkconfig -o /boot/grub/grub.cfg </code>

### 13. Installing tools ###

If everything went fine, your system should be able to boot into Gentoo right now. However, it's easier to install and configure tools such as Networking and System Logging before going into TTY.

Let's start by setting hostname:

<pre>(chroot) livecd # vim /etc/conf.d/hostname
hostname="funsociety"</pre>

Chaging the locale:

<pre>
(chroot) livecd # eselect locale list 
Available targets for the LANG variable:
  [1]   C *
  [2]   C.utf8
  [3]   POSIX
  [4]   en_US
  [5]   en_US.iso88591
  [6]   en_US.utf8
  [7]   pt_BR
  [8]   pt_BR.iso88591
  [9]   pt_BR.utf8
  [ ]   (free form)
(chroot) livecd # eselect locale set 6
(chroot) livecd # source /etc/profile 
</pre>

Changing /etc/hosts to use our hostname:

<pre>
(chroot) livecd # vim /etc/hosts
...
# IPv4 and IPv6 localhost aliases
127.0.0.1       funsociety localhost
::1             funsociety localhost
</pre>

Now for networking, first find your device name:
<pre>
(chroot) livecd # ifconfig -a
enp5s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.15.30  netmask 255.255.255.0  broadcast 192.168.15.255
        inet6 fe80::bf80:71e3:7fa2:b1d4  prefixlen 64  scopeid 0x20<link>
        ether 24:4b:fe:e1:a4:df  txqueuelen 1000  (Ethernet)
        RX packets 9514641  bytes 14162342545 (13.1 GiB)
        RX errors 984302  dropped 314  overruns 0  frame 714631
        TX packets 1869076  bytes 195960485 (186.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device memory 0xfc600000-fc61ffff  

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 5563  bytes 697189 (680.8 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5563  bytes 697189 (680.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlp4s0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether c8:b2:9b:df:1e:07  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
</pre>

enp5s0 is the name of my ethernet connection. I will be using NetworkManager on this system, built with the dhcpcd USE flag, since it works nicely with my WM.

Building networkmanager:

<code> (chroot) livecd # emerge --ask net-misk/networkmanager</code>

Enabling enp5s0 to use DHCP:

<pre>
(chroot) livecd # vim /etc/conf.d/net
...
config_enp5s0="dhcp"
</pre>

Enabling it at boot:

<code>rc-update add NetworkManager default</code>

Now let's start configuring the rc.conf to enable parellel service start and boot and also logging.

<pre>
(chroot) livecd # vim /etc/rc.conf
...
rc_parallel="YES"
rc_logger="YES"
</pre>

Emerging the logger:

<code> (chroot) livecd # emerge --ask --verbose app-admin/syslog-ng </code>

Starting it and enabling it at boot:

<pre> 
(chroot) livecd # rc-update add syslog-ng default 
(chroot) livecd # rc --service syslog-ng start 
</pre>

Installing a cron daemon for scheduled command:

<code> (chroot) livecd # emerge --ask --verbose sys-process/cronie </code>

Starting it and enabling it at boot:

<pre>
(chroot) livecd # rc-update add cronie default
(chroot) livecd # rc --service cronie start 
</pre>

Emerging mlocate to allow file indexing:

<code> (chroot) livecd # emerge --ask --verbose sys-apps/mlocate  </code>

(no service needed)

Emerging log rotation so that they don't use all our disk space:

<code> (chroot) livecd # emerge --ask --verbose app-admin/logrotate </code>

Emerging Portage tools:

<code> (chroot) livecd # emerge --ask --verbose app-portage/{mirrorselect,eix,gentoolkit,euses} </code>

Emerging chrony to synchronize our clock:

<code> (chroot) livecd # emerge --ask net-misc/chrony </code>

Enable it at startup:

<code> (chroot) livecd # rc-update add chronyd default </code>

### 13. Emerging Xorg and a Window Manager ###

So, if everything went smoothly, our system should be already good enough for boot. However, it will only prove a TTY. Let's change that by installing Xorg and a Window Manager.

Emerging Xorg:

<code> (chroot) livecd # emerge --ask x11-base/xorg-server </code>

Also, I want to use xrandr to setup my screens, which is not included on xorg-server:

<code> (chroot) livecd # emerge --ask x11-apps/xrandr </code>

Before building awesome, I want to make sure I use the most up-to-date version of it. To do so, let's add a rule for it on the package.accept_keywords:

<code> (chroot) livecd # echo "x11-wm/awesome **" > /etc/portage/package.accept_keywords/awesome </code>

Now to build AwesomeWM:

<code> (chroot) livecd # emerge --ask x11-wm/awesome </code>

The Terminal Emulator I mostly use is kitty, so I'm going to build it now:

<code> (chroot) livecd # emerge --ask x11-terms/kitty </code>

Finally, I'm going to emerge a few other utilities for this Machine, including Pulseaudio, Thunar and Firefox. This could all be done inside the Gentoo, but since I can setup portage to emerge all of this and come back when it's ready, I'd rather already do it.

So for this, let's do:

<code> (chroot) livecd # emerge --ask pulseaudio thunar firefox </code>

And this should be it! Hope this page was useful for you, and there is more to come. The next steps I will add will be creating the ~/.xinitrc and also the ~/.xprofile.
