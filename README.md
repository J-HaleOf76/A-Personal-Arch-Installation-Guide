# A PERSONAL ARCH INSTALLATION GUIDE
## I always forgot on how to configure some things in arch so here it is - a guide.
## This guide is focused for systemd-boot and UEFI setups.
Arch changes a lot of things in packaging like removing a lot of packages in base group. So, I will mostly forgot those yet again.

**I know that there is an Arch Wiki and etc etc. THIS IS A PERSONAL GUIDE so please don't bash me**  

I'm using GPT and systemd-boot in my installation.  

# Set the keyboard layout
The default console keymap is US. Available layouts can be listed with:
`# ls /usr/share/kbd/keymaps/**/*.map.gz`

To modify the layout, append a corresponding file name to loadkeys, omitting path and file extension. For example, to set a German keyboard layout:  
`# loadkeys us`


# Update the system clock
Use timedatectl to ensure the system clock is accurate:  
`# timedatectl set-ntp true`



# Disk Partitioning
+ **Verify the boot mode**
If UEFI mode is enabled on an UEFI motherboard, Archiso will boot Arch Linux accordingly via systemd-boot. To verify this, list the efivars directory:  
`# ls /sys/firmware/efi/efivars`

+ **Check the drives**
The most common main drive is **sda** *I think*  
`# lsblk`

+ **Let’s clean up our main drive to create new partitions for our installation**  
`# gdisk /dev/sda`  
  - Press *x* to enter **expert mode**. Then press *z* to **zap** our drive. Just hit *y* when prompted about wiping out GPT and blanking out MBR.  
  
Remember *x* button is to enter export mode and *z* is to zap or wipe-out the drive. **Think carefully before pressing Y you can't undo this!**

+ **Now let’s create our partitions**  
`# cgdisk`
  - Enter the device filename of your main drive, e.g. /dev/sda. Just press enter when warned about damaged GPT.

Now we should be presented with our main drive showing the partition number, partition size, partition type, and partition name. If you see list of partitions, delete all those first.

+ **Let’s create our boot partition**  

    + Hit New from the options at the bottom.
    + Just hit enter to select the default option for the first sector.
    + Now the partion size - Arch wiki recommends 200-300 MB for the boot + size. Let’s make it 500MiB or 1GB in case we need to add more OS to our machine. I’m gonna assign mine with 1024MiB. Hit enter.
    + Set GUID to **EF00**. Hit enter.
    + Set name to boot. Hit enter.
    + Now you should see the new partition in the partitions list with a partition type of EFI System and a partition name of boot. You will also notice there is 1007KB above the created partition. That is the MBR. Don’t worry about that and just leave it there.

+ **Create the swap partition**

    + Hit New again from the options at the bottom of partition list.
    + Just hit enter to select the default option for the first sector.
    + For the swap partition size, it is advisable to have 1.5 times the size of your RAM. I have 8GB of RAM so I’m gonna put 12GiB for the partition size. Hit enter.
    + Set GUID to **8200**. Hit enter.
    + Set name to swap. Hit enter.

+ **Create the root partition**

    + Hit New again.
    + Hit enter to select the default option for the first sector.
    + Hit enter again to input your root size.
    + Also hit enter for the GUID to select default.
    + Then set name of the partition to root.


+ **Create the home partition**

    + Hit New again.
    + Hit enter to select the default option for the first sector.
    + Hit enter again to use the remainder of the disk.
    + Also hit enter for the GUID to select default.
    + Then set name of the partition to home.

Lastly, hit *Write* at the bottom of the patitions list to write the changes to the disk. Type *yes* to confirm the write command. Now we are done partitioning the disk. Hit *Quit* to exit cgdisk.


# FORMATTING PARTITIONS

+ **Let’s see the new partition list**
`# lsblk`

**You should see something LIKE:**

| NAME | MAJ:MIN | RM | SIZE | RO | TYPE | MOUNTPOINT |
| --- | --- | --- | --- | --- | --- | --- |
| sda | 8:0 | 0 | 477G | 0 | disk |   |
| sda1 | 8:1 | 0 | 1 | 0 | part | /boot |
| sda2 | 8:2 | 0 | 1 | 0 | part | [SWAP] |
| sda3 | 8:3 | 0 | 175G | 0 | part | / |
| sda4 | 8:4 | 0 | 300G | 0 | part | /home  |


  +  sda1 is the boot partition
  +  sda2 is the swap partition
  +  sda3 is the home partition
  +  sda4 is the root partition

+ **Format the boot partition as FAT32**  
`# mkfs.fat -F32 /dev/sda1`  

+ **Enable the swap partition**  
`# mkswap /dev/sda2`  
`# swapon /dev/sda2`  

+ **Format home and root partition as ext4**  
`# mkfs.ext4 /dev/sda3`  
`# mkfs.ext4 /dev/sda4`  


# Connect to internet  
We need to make sure we are connected to the internet to be able to install Arch Linux base packages. Let’s see the names of our interfaces.  
`# ip link`

If you are on a wired connection, you can enable your wired interface by systemctl start dhcpcd@<interface>.  
`# systemctl start dhcpcd@enp3s0`

If you are on a laptop, you can connect to a wireless access point using wifi-menu -o <wireless_interface>.  
`# systemctl enable netctl`  
`# wifi-menu wlp7s0`  

Ping google to make sure we are online**  
`# ping -c 3 google.com`  

If you receive Unknown host or Destination host unreachable response, means you are not online yet. Review your network configuration and redo the steps above.

After setting up the drive and connecting to the internet, we are now ready to install Arch Linux to our system.

First we need to mount the partitions that we created earlier to their appropriate mount points.

+ Mount the root partition:  
`# mount /dev/sda3 /mnt`  

*If /mnt doesn't exists create it*  
`mkdir /mnt`  

+ Mount the boot partition  
`# mkdir /mnt/boot`  
`# mount /dev/sda1 /mnt/boot`  

+ Mount the home partion  
`# mkdir /mnt/home`  
`#  mount /dev/sda4 /mnt/home`  

We don’t need to mount **swap** since it is already enabled.  

Now let’s go ahead and install base, linux and base-devel packages into our system. The kernel is now not included in the base group including other application like editors. Even the lovely *nano* doesn't survived the snap.  

So let's also install the packages detached from the base group.  
`# pacstrap /mnt base linux linux-firmware base-devel less logrotate man-db man-pages which dhcpcd inetutils jfsutils mdadm perl reiserfsprogs sysfsutils systemd-sysvcompat texinfo usbutils xfsprogs s-nail netctl nano iputils`  

It's your decision if you want to install the packages.  

Just hit enter/yes for all the prompts that come up. Wait for Arch to finish installing the base, linux and base-devel and other packages.  


+ **Now let’s generate our fstab file**  
`# genfstab -U /mnt >> /mnt/etc/fstab`  


# Configuration

Now, chroot to the newly installed system  
`# arch-chroot /mnt /bin/bash`

The Locale defines which language the system uses, and other regional considerations such as currency denomination, numerology, and character sets. Possible values are listed in */etc/locale.gen*. Uncomment *en_US.UTF-8*, as well as other needed localisations.

Save the file, and generate the new locales  

`# locale-gen`  
`# echo 'LANG=en_US.UTF-8' > /etc/locale.conf`  


# Configure time:

A selection of timezones can be found under */usr/share/zoneinfo/*. Since I am in the Philippines, I will be using */usr/share/zoneinfo/Asia/Manila*. Select the appropriate timezone for your country:  

`# ln -s /usr/share/zoneinfo/Asia/Manila /etc/localtime`


Run hwclock to generate /etc/adjtime:  
`# hwclock --systohc`

# Localization

*Uncomment en_US.UTF-8 UTF-8* and other needed locales in `/etc/locale.gen`, and generate them with:  
`# locale-gen`

Create the locale.conf file, and set the LANG variable accordingly:  
`echo 'LANG=en_US.UTF-8' > /etc/locale.conf`

If you set the keyboard layout, make the changes persistent in vconsole.conf:  
`echo 'KEYMAP=us' > /etc/vconsole.conf`


# Network configuration

Create the hostname file:  
`echo '*myhostname*' > /etc/hostname`

Add matching entries to hosts:  
`nano /etc/hosts`  

Add this:  

127.0.0.1   localhost  
::1   localhost  
127.0.1.1   *hostname*.localdomain	*myhostname*  

If the system has a permanent IP address, it should be used instead of 127.0.1.1.

# Initramfs  
Creating a new initramfs is usually not required, because mkinitcpio was run on installation of the kernel package with pacstrap.
For LVM, system encryption or RAID, modify mkinitcpio.conf and recreate the initramfs image:  
`# mkinitcpio -P`


# Root password
Set the root password:  
`# passwd`

# Wireless Connections
For wireless connections, install iw, wpa_supplicant, and (for wifi-menu) dialog:  
`# pacman -S iw wpa_supplicant dialog`

# Adding Repositories
Enable multilib and AUR repositories in /etc/pacman.conf:  
`# nano /etc/pacman.conf`

Uncomment multilib (remove # from the beginning of the lines):  
[multilib]
Include = /etc/pacman.d/mirrorlist

Add the following lines at the end of /etc/pacman.conf to enable AUR repo:  
`[archlinuxfr]
SigLevel = Never
Server = http://repo.archlinux.fr/$arch`

# Password and Users
Set password for root:
`# passwd`

Add a new user with username *YOURUSERNAME*:  
`# useradd -m -g users -G wheel,storage,power,video,audio,rfkill -s /bin/bash *YOURUSERNAME*`  

Set password for *YOURUSERNAME*:  
`# passwd YOURUSERNAME`


# Add the new user to sudoers:
`EDITOR=nano visudo`

Uncomment *# %wheel ALL=(ALL) ALL*


# Install the boot loader:  
`# bootctl install`

Create a boot entry:  
`# nano /boot/loader/entries/arch.conf`

`title Arch Linux`  
`linux /vmlinuz-linux`  
`initrd  /initramfs-linux.img`  
`options root=/dev/sda3 rw`

# Exit chroot and reboot:  
`# exit`
`# reboot`


# POST INSTALLATION
# Pacman  
**This is for my setup so don't follow it if you don't want to.**
Basic Xorg apps  
`sudo pacman -S xorg-server xorg-xrdb xorg-xinit xorg-xkill xclip xorg-xbacklight xorg-xbacklight xorg-xrandr xorg-xdpyinfo xorg-xev`

Graphics  
`sudo pacman -S xf86-video-intel gstreamer-vaapi`

Audio  
`sudo pacman -S alsa-utils pulseaudio-alsa pulseaudio-bluetooth pulseaudio`

For filesystem  
`sudo pacman -S unrar unzip p7zip gvfs-mtp libmtp android-udev nemo nemo-fileroller nemo-preview ffmpeg ffmpegthumbnailer ntfs-3g`

Browser  
`sudo pacman -S firefox`

Terminal Emulators  
`sudo pacman -S kitty xterm`

Network  
`sudo pacman -S networkmanager dhclient modemmanager usb_modeswitch network-manager-applet macchanger mobile-broadband-provider-info aircrack-ng`

Security  
`sudo pacman -S tlp ufw`

Bluetooth  
`sudo pacman -S bluez bluez-utils blueman`

Misc and tools  
`sudo pacman -S git neovim atom neofetch pacman-contrib imagemagick maim mupdf acpi redshift arandr lightdm-webkit2-greeter dconf-editor acpid acpi_call`

Settings  
`sudo pacman -S lxappearance`

Virualbox  
`sudo pacman -S virtualbox virtualbox-host-modules-arch linux-headers virtualbox-guest-iso`

Media  
`sudo pacman -S vlc mpc mpd ncmpcpp  perl-image-exiftool feh pulseeffects flowblade simplescreenrecorder`

Media editors  
`sudo pcman -S gimp inkscape`

# Yay
`git clone https://aur.archlinux.org/yay.git`

AUR apps  
`yay -S plymouth-git compton-tryone-git cava-git lightdm-webkit-theme-aether awesome-git vicious-git mugshot flat-remix flat-remix-gtk pamac-aur gtk3-nocsd-git`

`yay -S virtualbox-ext-oracle`

`yay -S gtk3-mushrooms`

