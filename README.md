# Arch Linux Project

## Obtaining and Booting .iso
- Obtained iso using the [download page](https://archlinux.org/download/)
    - I opted to download from [the university of Arizona mirror](https://www.arizona.edu)
- The SHA256 sum was checked by using PowerShell's `Get-FileHash` comand
- Using VirtualBox, a VM was created linked to the .iso
    - The VM was given:
        - 5GiB of RAM
        - 25GB Disk space
        - 8 Processors
        - 128MB VRAM
    - EFI was enabled
- The VM was then booted to the live image

## Initial Booting Steps

#### Check UEFI boot mode
First, the boot mode was checked using `cat /sys/firmware/efi/fw_platform_size`. This command resulted in `64` meaning it properly booted into 64-bit UEFI.

#### Connect to Internet

First, the network interface was tested using `ip link` which correctly showed the `enp0s3` interface from the VM. A quick test using `ping 1.1.1.1` indicated that the internet was connected and funtional within the live environment.

> [!WARNING]
> Internet functionality while live booting **does not** mean that internet will work when booted into the system. The live boot has additional tools for internet connectivity that must be installed manually. The first time I booted, I didn't have internet to install anything which was a headache. Make sure to follow the network functionality section.

#### Disk Paritioning

The virtual hard disk provided by the hypervisor must be partitioned into different sections. For my case, I partitioned the disk into two parts: one for the EFI/bootloader and one for the arch filesystem. 

The steps for this are:  
1. Use the `fdisk -l` command to see current partitions/disks
    - the /dev/sda disk was the proper disk for my system to make the partitions
2. Use the `fdisk /dev/sda` command to partition the disk
3. Create a new partition with `n`
4. Select primary partition
5. Select size
    - I opted for 500MB for the EFI partition and the remainder of the disk for the root partition
6. Set the partition type with `t`
    - The EFI partition will be EFI type (FAT32)
    - The root partition will be linux (EXT4)
7. Repeat steps 3-6 for both partitions

The next step is to format the new partitions so they align with the file system types assigned to them. Since the EFI partition is /dev/sda1, the command `mkfs.fat -F 32 /dev/sda1` formatted it properly to use FAT32. The root partition was formatted with `mkfs.ext4 /dev/sda2` to properly format it to use EXT4. 

## Mounting and Interacting with the New System

#### Mounting the file systems

The disks that were formatted must be mounted to the live image in order to make them bootable. To do this, the command used was `mount /dev/sda2 /mnt` which placed the new root system partition into the /mnt directory. Further, the EFI partition was mounted as well using `mount --mkdir /dev/sda1 /mnt/boot`. This command made a new directory (/boot) in the root filesystem to hold the boot partition. 

#### Install essential packages to mounted system

The next step is to install packages onto the mounted disk. To simplify, I used the basic installation with the command `pacstrap -K /mnt base linux linux-firmware`. This command installed to /mnt the base, linux, and linux-firmware packages. 

> [!WARNING]
> Make sure not to use pacman here; that installs packages to the live image versus the new system. Pacstrap works for the base packages and pacman will work once chroot-ed in.

#### Generate fstab file

The command `genfstab -U /mnt >> /mnt/etc/fstab` is used to generate an fstab file to allow proper mounting of file systems on startup. 

#### Interacting with the full system using `chroot`

Now, a basic filesystem is in place on our mounted root partition, so it is possible to use the `chroot` command to interact directly with the new system. this is done using the `arch-chroot /mnt` command.

#### Locale settings

The following commands were run to set locale and improve quality-of-life on the machine:

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=us" > /etc/vconsole.conf
echo "arch-project" > /etc/hostname
```

#### Package updates

The packages on the system were updated using: `pacman -Syu`

#### Network Function

To ensure network function when booted into the system, the following command is run: `pacman -S networkmanager netctl`.

#### Enable login

A root password is made using the `passwd` command such that I can login to the system after reboot.

#### Intalling the Bootloader

The following steps were taken to install the GRUB bootloader:

1. Install packages using `pacman -S grub efibootmgr`
2. Mount the EFI partition using `mount /dev/sda1 /boot`
3. Install grub using `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
4. Use the command `grub-mkconfig -o /boot/grub/grub.cfg` to add configuration files allowing grub to see the root os on startup

> [!WARNING]
> My first attempt, I didn't do the `grub-mkconfig` command, and it led the bootloader to not see the arch install. This was a real headache to figure out because the initilization commands had to come manually from the GRUB CLI. **Make sure to add configuration files to ensure Arch shows up in the bootloader!!**

#### Rebooting the system

The `reboot` command is used to reboot the system. This allows a direct boot through grub into the newly installed system.

> [!TIP]
> If anything happens wrong on the reboot, it's not too hard to recover! On my first reboot, there was no network manager, and GRUB didn't see the Arch install. If anything strange happens, and the system does not boot, use the live image again. Reboot into the image and remount the root partition (`mount /dev/sda2 /mnt`). You can modify the system files directly from the live image without needing to worry about booting into it. Once it's fixed, you can remove the live image. 

## Additional Requirements

### Add additional users

To add the users calvin and codi, the `useradd` command is used

```bash
useradd -m -g wheel calvin # -m tag means to add a home directory
useradd -m -g wheel codi # -g wheel means to add the user to the wheel group (for sudo permissions)
```

Then the wheel group must be given sudo permisions. To do this the `visudo` command is used. I uncommented the line reading:

```
%wheel ALL=(ALL:ALL) ALL
```

> [!NOTE] 
> For my system, vi had to be installed. I didn't bother trying to change the default text editor for visudo.

This allowed all users in the wheel group to be able to use sudo commands.

### Desktop Environment

I opted to install the LXQT desktop environment with the following steps:

1. Install xorg with `pacman -S xorg`
2. Install all packages in lxqt group with `pacman -S lxqt`
3. Install icons with `pacman -S breeze-icons`
4. Install display manager with `pacman -S sddm`
5. Enable sddm with `systemctl enable sddm`
    - This lets the system boot directly into sddm on startup
6. Reboot

> [!WARNING]
> sddm **must** be enabled with systemctl before running the sddm command. Otherwise, it will only display the login screen, and login attempts do not work. I'm not sure why this is, but this happened to me in my install.  

### Alternative Shell

I opted to switch to the zsh shell using the following steps:

1. Install zsh with `pacman -S zsh`
2. Use the command `chsh -s /bin/zsh` to set the curren user's default shell to zsh

### Install SSH

1. Install openssh using `pacman -S openssh`
2. Enable and start the service with:
```bash
systemctl enable sshd
systemctl start sshd
```

### Color Coding in Terminal

To add colors in the terminal, individual flags must be set for each command and a custom prefix an be set. To add the command flags, alias can be used. The prefix can be made a variable. These commands can be added to the .bashrc file to be run on login.

.bashrc:
```bash
alias ls='ls --color=auto'
alias grep='grep --color=auto'

PS1='\[\e[38;5;140;1m\]\w\[\e[0m\]: \[\e[38;5;85m\]\u\[\e[0m\]\$' # custom prefix with color
```

> [!TIP] [Bash Prompt Generator](https://bash-prompt-generator.org/) is a pretty cool tool for generating fun prompts.

### Add Custom Aliases

I added the following aliases the shell configuration file:

```bash
alias update-packages='pacman -Syu'
alias c='clear'
alias ll='ls -lah --color=auto'
```