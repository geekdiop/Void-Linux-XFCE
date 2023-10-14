Void_Linux_Full_Disk_Encryption_XFCE
Void-Linux-UEFI-LUKS-EXT4

##### Void Linux UEFI install on NVME with AMD graphic 2022-09
##### This guide will refer to the OS disk as /dev/nvme0n1 and Uses ext4 as filesystem
##### I encourage you to peruse the Void Linux Handbook at [Void](https://docs.voidlinux.org/about/index.html) and read the [original](https://gitlab.com/BreakPointSSC/void-linux-uefi-luks-btrfs/-/blob/main/Void_Linux_Full_Disk_Encryption_KDE.txt) guide.  [This](https://unixsheikh.com/tutorials/real-full-disk-encryption-using-grub-on-void-linux-for-bios.html) and [this](https://gist.github.com/Dko1905/dbb88d092aa973a8ba244eb42c5dd6a6) also are great.

-----------------------------------------------

**INITIAL SETUP:**
- Download the ISO file from (https://voidlinux.org/download/).
    - Burn it to a USB/CD ... && boot it.  
    - Login as `root` with password `voidlinux`.
    - configure your wireless connection manually using wpa_supplicant and dhcpcd:
        ```
            cp /etc/wpa_supplicant/wpa_supplicant.conf /etc/wpa_supplicant/wpa_supplicant-<wlan-interface>.conf
            wpa_passphrase <SSID> <passphrase> >> /etc/wpa_supplicant/wpa_supplicant-<wlan-interface>.conf
            sv restart dhcpcd
            ip link set up <interface>
        ```
    -   Setup your keyboard:
            > loadkeys KEYMAP  (e.g, loadkeys us or loadkeys fr).

#### PARTITION THE DISK: ####

   -  I have a 512G ssd, and I'm going to create 3 partitions:
      -  1) EFI System Partition : 200M - 1GB. It's your choise.
      -  2) Root partition, including home directory: 216G.
      -  3) "Personal files" partition: 260G.
        
    To list your block devices, write this command lsblk on your terminal; (my device/harddrive is:  /dev/nvme0n1): 
   > **lsblk**      #You have located your harddrive.
       
    Then use `cfdisk` or `fdisk` command to create your partitions: I've used (cfdisk) to create my partitions and this is the result: 
   > **cfdisk**

```
    Device                             Start                End           Sectors           Size Type
>>  /dev/nvme0n1p1                      2048             553247            551200         269.1M EFI System             
    /dev/nvme0n1p2                    555008          453539839         452984832           216G Linux filesystem
    /dev/nvme0n1p3                 453539840         1000215182         546675343         260.7G Linux filesystem
```

#### SET UP ENCRYPTED ROOT VOLUME: #### 
- Encrypt root partition with LUKS1 to avoid any problems: 
    ``` 
   1-  cryptsetup luksFormat --type luks1 -y /dev/nvme0n1p2 
   2-  cryptsetup  --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --use-random luksFormat /dev/nvme0n1p3
    ```
     
- Now, Let's open the LUKS partition. And it will be mounted under /dev/mapper/cryptroot  #you can change this "cryptroot" into another name
   
   ```
   cryptsetup open /dev/nvme0n1p2 cryptroot
   ```

#### LVM: ####  
- I'll create a 16G Swap and the rest for root files. But first let's create a volume group and then create logical volumes in our volume group: 
    ```
    vgcreate myvoid /dev/mapper/cryptroot
    ```
  - rename myvoid into another name you like.
    
- Create a swap partition: 
    ```
    lvcreate --name swap -L 20G myvoid
    ```
- And the rest of the space is root: 
  ```
    lvcreate --name root -l 100%FREE myvoid
  ```

#### FORMAT THE PARTITIONS: #### 
```
    mkfs.vfat -F32 -n EFI /dev/nvme0n1p1        #Format the EFI System Partition
    mkfs.ext4 /dev/mapper/myvoid-root           #Format the root Partition
    mkswap /dev/mapper/myvoid-swap              #create our swap
    lsblk -f                                    #Verify file systems
```

### Mount & Chroot: ###
```
    mount /dev/mapper/myvoid-root /mnt           # Mount root
    for dir in dev proc sys run; do mkdir -p /mnt/$dir ; mount --rbind /$dir /mnt/$dir ; mount --make-rslave /mnt/$dir ; done
    mkdir /mnt/efi                              #Create mount point for the ESP
    mount -o rw,noatime /dev/nvme0n1p1 /mnt/efi
```
- Let's install the base system + some packages: 
   - For glibc (I installed this one):
       ```
       xbps-install -Sy -R https://alpha.de.repo.voidlinux.org/current -r /mnt base-system linux-mainline cryptsetup grub-x86_64-efi nano bash-completion NetworkManager iwd vim wget curl gcc
       ```
       - Copy the DNS config into the new root so that XBPS can still download new packages inside of it:
          > cp /etc/resolv.conf /mnt/etc/
    
      - Chroot into the new installation: 
          > chroot /mnt /bin/bash

### INSTALLATION CONFIGURATION: ###
- Set the time:
      ``` ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime ``` 
- Uncomment desired locales by:
      ``` nano /etc/default/libc-locales ``` 
    - for me:
      ``` echo 'en_US.UTF-8 UTF-8' >> /etc/default/libc-locales ``` 
    - Set a hostname: 
      ``` echo myvoid > /etc/hostname ``` 
    - For glibc builds, generate locale files:
      ``` xbps-reconfigure -f glibc-locales ``` 
>  
    > Edit this file /etc/rc.conf, if you like, to suit your needs.
> 
    - Change root's default shell to bash:
      ``` chsh -s /bin/bash root ``` 
    - Create a new user:
    >
    > useradd --create-home --groups wheel,users,audio,video,storage,cdrom,input --shell /bin/bash myUsername_here

- Change the root password and assign a password to your new user:
    > password root
    
    > password myUsername_here

- Enable users in the wheel group to use sudo.
```
  EDITOR=nano visudo
  Uncomment %wheel ALL=(ALL) ALL
```

### CONFIGURE SUBREPOS AND MIRRORS: ###
```
    Sync XBPS with the remote repository:
    xbps-install -S
    Install the nonfree subrepo to enable installing proprietary firmware: xbps-install void-repo-nonfree && xbps-install -S
    Synchronize XBPS with the new mirros:   xbps-install -S
    Verify the new mirrors: xbps-query -L
```

### CONFIGURING FSTAB: ###
    Edit the (/etc/fstab) file and add some lines -- becareful /with/ this part:
    > vim /etc/fstab   ---> this is my file:
   
 ```
#
# See fstab(5).
#
# <file system> <dir>   <type>  <options>               <dump>  <pass>

UUID=XXXX-XXXX                               /efi            vfat    defaults,noatime        0       2

UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX	   /               ext4    defaults,noatime        0       0 

UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX	   swap            swap    defaults                0       0

tmpfs                                        /tmp 		       tmpfs   defaults,nosuid,nodev 	 0	      0

```
***use (blkid) command to see your UUIDs and set them manually***

- Or, you can use this method to redirect the output to the fstab file: 
    > blkid -o value -s UUID  /dev/nvme0n1p1 >> /etc/fstab
    > 
    > blkid -o value -s UUID /dev/mapper/myvoid-root >> /etc/fstab 
    > 
    > blkid -o value -s UUID  /dev/mapper/myvoid-swap >> /etc/fstab

### INSTALLING GRUB: ###
We've already installed this package ("xbps-install grub-x86_64-efi")
- Enable encryption support: 
    >
    > echo GRUB_ENABLE_CRYPTODISK=y >> /etc/default/grub
    >
- Add needed UUIDs to /etc/default/grub:  
```
GRUB_CMDLINE_LINUX_DEFAULT="rd.lvm.vg=myvoid rd.luks.uuid=UUID_here"
```
  **NB** : _UUID_here is the uuid of /dev/nvme0n1p2 not the uuid of /dev/mapper/root)_
  
- Add also this line:
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=4 page_poison=1 rd.auto=1 rd.luks.allow-discards"
```
  1) "page_poison=1" to mitigate use-after-free vulnerabilities.
  2) "rd.auto=1" to autodetect LUKS.
  3) "rd.luks.allow-discards" to allow SSD TRIM support for LUKS volumes.

- Install grub as the bootloader: 
```
   grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id="Void Linux"
```
- And then generate grub config:
```
grub-mkconfig -o /boot/grub/grub.cfg
```
### [CREATE KEYFILE TO AVOID HAVING TO ENTER THE PASSPHRES TWICE ON BOOTUP](https://gitlab.com/BreakPointSSC/void-linux-uefi-luks-btrfs/-/blob/main/Void_Linux_Full_Disk_Encryption_KDE.txt): ### 
- Create the keyfile out of random data: 
```
    dd bs=515 count=4 if=/dev/urandom of=/boot/keyfile.bin
```
- Add a 2nd key slot to the LUKS encrypted volume with keyfile.bin as the key:
```
  cryptsetup -v luksAddKey /dev/nvme0n1p2 /boot/keyfile.bin
```
- Secure the keyfile by setting appropriate permissions
```
  chmod 000 /boot/keyfile.bin
```
- Allow only root to access /boot:
```
  chmod -R g-rwx,o-rwx /boot
```
- Set up crypttab:
```
  echo 'cryptroot UUID=<UUID OF /dev/nvme0n1p2> /boot/keyfile.bin luks' > /etc/crypttab
    > the result is: cryptroot UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX /boot/keyfile.bin luks
```
- Cofigure dracut to include the keyfile and crypttab in the initial RAM disk:
```
  echo 'install_items+=" /boot/keyfile.bin /etc/crypttab "' > /etc/dracut.conf.d/10-crypt.conf
```

### D-BUS AND ELOGIND: ###

- Install elogind and version of dbus with support for elogind features:

```
  xbps-install dbus-elogind dbus-elogind-libs dbus-elogind-x11 elogind

```

- To make NetworkManager use iwd as the Wi-Fi backend instead of wpa_supplicant: 
  
  ```
  mkdir -p /etc/NetworkManager/conf.d/
  vim /etc/NetworkManager/conf.d/wifi_backend.conf 
  ```
  - and add these lines: 
  ```
  [device]
  wifi.backend=iwd
  wifi.iwd.autoconnect=yes 

  ```
- (Optional) If target PC has an AMD CPU, install AMD CPU microcode and AMD GPU firmware:
```
      xbps-install linux-firmware-amd
```

- Install the OpenGL driver / the Khronos Vulkan Loader / Xorg and Wayland:
```
      xbps-install mesa-dri vulkan-loader xorg xorg-server-xwayland
``` 

- AMD RADEON:
    - Install the mesa vulkan driver  /   the xorg driver /   the video acceleration driver  and the OpenCL compute driver:
      ```
              xbps-install mesa-vulkan-radeon xf86-video-amdgpu   mesa-vaapi mesa-vdpau   mesa-opencl clinfo
      ```

- Install PipeWire:
```
    xbps-install rtkit pipewire alsa-pipewire libjack-pipewire
```

  - Set up pipewire autostart:
  ```
        cat <<EOF >> /etc/xdg/autostart/pipewire.desktop
        [Desktop Entry]
        Type=Application
        Name=Pipewire
        Exec=pipewire
        EOF
  ```
  - Set up pipewire-pulse autostart:
  ```
      cat <<EOF >> /etc/xdg/autostart/pipewire-pulse.desktop
      [Desktop Entry]
      Type=Application
      Name=Pipewire-Pulse
      Exec=pipewire-pulse
      EOF
  ```
  
- INSTALL PACKAGES FOR SYSTEM LOGGING AND NTP:
```
  xbps-install chrony socklog-void
  usermod -a -G socklog username #Add user to the socklog group

```

### - ENABLE SERVICES ###
  - Enable socklog, a syslog implementation from the author of runit.
   ```
    ln -s /etc/sv/socklog-unix /var/service/
    ln -s /etc/sv/nanoklogd /var/service/
   ```
  
  - Enable D-Bus which is required by NetworkManager, KDE, GNOME, and Bluetooth
    ```
      ln -s /etc/sv/dbus /var/service/
    ```
  - Enable the iNet Wireless Daemon for Wi-Fi support
    ```
        ln -s /etc/sv/iwd /var/service/
    ```
  - Enable the NetworkManager service to get networking started.
 
    ```
    ln -s /etc/sv/NetworkManager /var/service/
    ```
  - Enable chrony for Network Time Protocol support
   ```
    ln -s /etc/sv/chronyd /var/service/
   ```
  - Set up PipeWire ALSA

   ```
    mkdir -p /etc/alsa/conf.d
   ```
   ```
    ln -s /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
   ```
   ```
    ln -s /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
   ```  
  - Enable Bluetooth
   ```
    ln -s /etc/sv/bluetoothd /var/service/
   ```
 
### USER APPLICATIONS (Optional): ###
- From [HERE](https://gitlab.com/BreakPointSSC/void-linux-uefi-luks-btrfs/-/blob/main/Void_Linux_Full_Disk_Encryption_KDE.txt)

### DESKTOP ENVIRONMENT: XFCE && Lightdm: ###  (enable it after, delte above)
```
  sudo xbps-install  xfce4
```
```
  sudo xbps-install lightdm lightdm-gtk-greeter
```
  - Enable lightdm// Display Manager
   ```
    ln -s /etc/sv/lightdm/ /var/service/
   ```
-   then:
    -   Try rebooting if things are running slow after the first login.
    -   If the default directories such as Documents, Downloads, and Pictures are missing, run:
         ```
             $ xdg-user-dirs-update
         ```

###FINALIZATION: ###
- ensure all installed packages are configured properly:
```
  xbps-reconfigure -fa

```
- Exit the chroot and reboot: 
```
  exit
```
```
  umount -R /mnt
```
 ```
  reboot
```
