# arch-linux-install
This file includes a quick guide to installing arch linux

1. [Install with encrypted home](#install-with-encrypted-home)


## Install with encrypted home

1. Download the image and make a bootable flash drive by any method (rufus is preferable). <br>
2. Boot into bios, disable secure boot, switch system boot to usb, save and reboot. Then start installing arch linux. <br>
3. If you have a cable connection then skip this step, if you have a wifi connection then you need to use the `iwctl` tool. <br>
Enter the `iwctl` command then look up the station name via `iwctl device list`. <br>
Then type `iwctl station <station_name> get-networks` this will show the available networks. Next enter `station <station_name> connect <network_name>` then enter the network password. Next, exit iwctl by typing `exit`.  <br>
You can check if the connection was successful by `ping google.com` or any other address. <br>

4. Then using the command `cfdisk /dev/..diskname..` (e.g. /dev/sda, /dev/nvme0n1, etc.). Let's assume the name of our disk is `/dev/sda`. <br>
You need to create partitions for your future system. I usually create 1G for boot partition, 8G for swap partition or more, the rest I distribute between root and home partitions. <br>
Once created, the structure is as follows: <br>
`/dev/sda1`, `/dev/sda2`, `/dev/sda3`, `/dev/sda4`. <br>
You can check it by typing the `lsblk` command. <br>

5. Next we need to encrypt the home partition. Let's say it is `/dev/sda4`. <br>
Enter the command `cryptsetup luksFormat /dev/sda4` <br>
After that we need to enter YES and enter the desired password for decryption. <br>

6. Next we need to open the encrypted partition by typing `cryptsetup open /dev/sda4 <name>` (here you can specify the desired name for the name of the decrypted partition, let's say it will be `crypthome`). <br>
Then enter the password and we will have our decrypted partition on the path `/dev/mapper/crypthome`. <br>

7. Next we need to set the file system for our partitions and also enable swap if there is one. <br>
Enter `mkfs.fat -F32 /dev/sda1` (boot partition) <br>
`mkfs.ext4 /dev/sda3` (root partition) <br>
`mkfs.ext4 /dev/mapper/crypthome` (set file system for decrypted home) <br>
`mkswap /dev/sda2` (install swap) <br>
`swapon /dev/sda2` (activate swap) <br>

8. The next step is to mount the partitions. <br>
`mount /dev/sda3 /mnt` (root partition) <br>
`mkdir /dev/boot` (directory for boot loader) <br>
`mkdir /dev/home` (directory for home) <br>
`mount /dev/sda1 /mnt/boot` (boot partition) <br>
`mount /dev/mapper/crypthome /mnt/home` (decrypted home partition) <br>

9. Next, install the system using pacstrap <br>
`pacstrap -i /mnt linux linux-firmware base base-devel nano sudo networkmanager grub lvm2 cryptsetup efibootmgr ` <br>

10. Generate fstab file <br>
`genfstab -U /mnt > /mnt/etc/fstab` <br>

11. Next, you need to enter the system using arch-chroot <br>
`arch-chroot /mnt` <br>

12. Next, set the locale, time, and other system variables <br>
`nano /etc/locale.gen` - select system language <br>
`locale-gen` - generate languages <br>
`echo “LANG=en_US.UTF-8” > /etc/locale.conf` - set environment variable <br>
`ln -sf /usr/share/zoneinfo/Europe/Kyiv /etc/localtime` - set time zone. Here you should specify your own time zone. <br>
`hwclock --systohc` - set the time <br>
`echo archlinux > /etc/hostname` - set hostname <br>
`nano /etc/hosts` - specify host mapping: <br>
127.0.0.1 localhost <br>
::1       localhost <br>
127.0.1.1 archlinux.localdomain archlinux <br>

13. Next, create a new user and assign roles and password to it <br>
`useradd -m -G wheel,storage,power,audio,video -s /bin/bash username` <br>
`passwd` - password for root user <br>
`passwd username` - password for the created user <br>

14. Next, we edit this file: <br>
`nano /etc/sudoers` <br>
Uncomment this line `%wheel ALL=(ALL) ALL` <br>

15. Next, in the crypttab file you need to specify the mapping of the encrypted and decrypted partition <br>
Enter the command: <br>
`blkid -o valud -s UUID /dev/sda4 >> /etc/crypttab` (sda is the encrypted home partition). This will move the UUID of the partition in the crypttab file to the end of the file. <br>
Next we need to edit this file. <br>
`nano /etc/crypttab` <br>
There should be the following structure <br>
crypthome UUID=<uuid_from_blkid_command> none luks <br>

16. Next, edit the file: <br>
`nano /etc/mkinitcpio.conf` <br>
In the HOOKS section, write the following: <br>
HOOKS=(... block encrypt lvm2 ...) <br>
Execute the command `mkinitcpio -P` <br>

17. Installing grub: <br>
`grub-install --efi-directory=/boot --bootloader-id=GRUB /dev/sda` In `bootloader-id` you can specify any name you want. Where `/dev/sda` you need to specify a disk, not a partition, but a disk. <br>
Enter the command: `grub-mkconfig -o /boot/grub/grub.cfg` <br>

18. Enabling networkmanager: <br>
`systemctl enable NetworkManager` <br>

19. Exit, unmount and reboot <br>
`exit` <br>
`umount -R /mnt` <br>
`reboot` <br>

20. Exit to bios and switch the boot to the new system <br>

21. Enter crypthome password, user login and password <br>

22. Connect to wifi again if necessary <br>
`nmcli device wifi connect <network_name> password <password>` <br>

23. Change the `getty@.service` file so that when you enter the password for crypt home, the user logs in at once <br>
`sudo nano /lib/systemd/system/getty\@.service` <br>
In the `ExecStart` line, remove `-o '-p -- \\u'` and add `-a username` instead <br>

24. Install sddm and a graphical environment such as kde plasma <br>
`sudo pacman -S sddm plasma-desktop konsole kscreen` <br>

25. Running sddm <br>
`sudo systemctl enable --now sddm` <br>

26. Enter the user password. Congratulations, you're in. <br>
