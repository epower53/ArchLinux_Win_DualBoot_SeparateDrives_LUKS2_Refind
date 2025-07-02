# Dual boot Arch Linux and Windows 10/11 with secure boot, rEFInd boot manager, full-disk LUKS2 encryption with auto-TPM2 unlocking, and BTRFS
This is my take on how to dual boot Arch and Windows on separate physical disks with rEFInd, full LUKS2 disk encryption with TPM2 automated unlocking, BTRFS, and secure boot, all using a single EFI partition shared between both operating systems. None of the how-to's / writeups / tutorials I've found gave me exactly what I want, so I took bits I liked from each and came up with my own. These instructions assume two SATA drives (sda and sdb) - adjust as needed for your system (e.g. nvme0n1, etc).

# Install Windows First
1. Create a custom-sized 2GB EFI partition before installing Windows. There are a couple of options for this:

   a) Using the Windows install media, press `<Shift>-<F10>` to open a command prompt when you arrive at the partition creation/selection screen, and enter the following:

       diskpart
         list disk
         select disk <X>                  #replace <X> with the disk you want to use for Windows
         clean                            #equivalent to sgdisk -Z
         convert gpt
         create partition efi size=2048   #size is in MB
         format fs=fat32 quick
         exit
       exit
     Click the "refresh" icon to make sure the GUI shows the newly created partition. Select the remaining empty space on the disk with the EFI partition and click `<Install>`. Windows will use the EFI partition you created rather than make its own.

   b) Use your Arch live ISO thumbdrive to create the partition. Boot from the thumbdrive, and then:

       sgdisk -Z /dev/sda                 #zap the disk so we can start fresh
       gdisk /dev/sda
         o                                #create a new GPT
         n
         <Enter>                          #choose the default partition number
         <Enter>                          #choose the default start sector
         +2G                              #make the size 2GB
         ef00                             #partition type for EFI
         c                                #let's add a helpful name to the partition
         1
         Shared EFI                       #Call it anything you like
         w                                #write the changes
         y                                #confirm
       mkfs.fat -F 32 /dev/sda1
       shutdown -r now
     During the reboot, make sure to switch to the Windows install media, and install Windows to the empty space on the same disk with the new EFI partition. Windows will use the EFI partition you created rather than make its own.
   
2. Complete all your favorite Windows post-installation tasks (drivers, updates, etc).
3. Make sure to disable hibernate and fast boot features.
4. (Optional) I like to run <a href="https://github.com/ChrisTitusTech/winutil">Chris Titus WinUtil</a> (it's a Powershell script) to remove telemetry and other unnecessary/invasive bloat. If you use Chris' tool, make sure to enable the UTC clock option to prevent Windows and Arch from fighting over the system clock.
5. If you do not run Chris Titus WinUtil, make sure to set the Windows system clock to UTC.
6. (Optional) enable Bitlocker. If you choose to do this, make sure you save a copy of the recovery key somewhere other than the OS you're configuring. There's a reasonable chance that the TPM slots on your mainboard could get borked somewhere along the line, and you'll need this key to get back into Windows if that happens. I just print the recovery key and stick it in my filing cabinet.

# Get online, create the Linux partition, encrypt it, and create the filsystem
7. Reboot and enter the UEFI BIOS.
8. Disable secure boot and fast boot options, if you haven't already done so.
9. Boot into your live Arch USB.
10. If using wired networking, odds are you already have Internet access. Verify with `ip addr`. If your ethernet interface didn't receive an IP address from your DHCP server, you'll need to debug this on your own.
11. If using a wireless adapter you'll need to join a network:

        iwctl
          device list                           #get the name of your wireless device. Let's assume it's wlan0
          station wlan0 scan                    #scan for wifi networks
          station wlan0 get-networks            #look over the results of the scan to find your network
          station wlan0 connect <Network Name>  #enter the network password at the prompt and all should be good
          <Ctl>-D                               #exit the interactive iwd session
12. Verify device labeling... Windows should be on `/dev/sda`, with `/dev/sdb` empty and waiting. `fdisk -l` should show:
   - `/dev/sda`
     - `/dev/sda1                                  #2GB EFI partition`
     - `/dev/sda2                                  #16MB Windows MSR (why?)`
     - `/dev/sda3                                  #NTFS main Windows partition`
     - `/dev/sda4                                  #Windows recovery partition (ha!)`
   - `/dev/sdb`                      
13. Create one giant partition for the encrypted Linux volume:

        gdisk /dev/sdb
          o                                      #GPT creation
          n
          <Enter>                                #default partition number is fine
          <Enter>                                #default start sector is fine
          <Enter>                                #default end sector is the biggest we can make this partition
          8309                                   #Linux LUKS partition type
          c                                      #Let's give it a name
          1
          ArchLUKS                               #Pick a name that makes sense to you
          w                                      #write changes
          y                                      #confirm
14. Check that `/dev/sdb1` was created successfully

        fdisk -l
15. Encrypt the partition. You can dig through the docs on LUKS and/or read from the many available tutorials and you'll see people using all sorts of options for this step - the defaults are actually pretty sane choices for everyday use, so I don't bother with much. I like to specify luks2 (even though it's a default) just to be sure, and when unlocking the partition I like to enable TRIM (--allow-discards) and set it up to always remember that choice (--persistent).

        cryptsetup luksFormat --type=luks2 /dev/sdb1
        YES
        <Choose your own passphrase>                                  #choose something that's long and difficult, but still possible to type - you'll need to type it a couple of more times
        cryptsetup open --allow-discards --persistent /dev/sdb1 luks  #use your passphrase to unlock the encrypted partition and give it an easy name (e.g. luks)
16. Create a BTRFS filesystem, mount it, and create subvolumes

        mkfs.btrfs -L ArchLinux /dev/mapper/luks                      #the name ArchLinux can be anything - it never gets referenced again, so choose freely
        mount /dev/mapper/luks /mnt                                   #this mount is temporary so we can create subvolumes
        btrfs subvolume create /mnt/@                                 #root goes here
        btrfs subvolume create /mnt/@home                             #home goes here... any additional subvolumes are optional and you can customize as you like. Your planned BTRFS feature use should guide you.
        btrfs subvolume create /mnt/@log
        btrfs subvolume create /mnt/@cache
        btrfs subvolume create /mnt/@snap
        btrfs subvolume create /mnt/@swap
17. Remount the root subvolume to `/mnt`

        umount /mnt
        export sv="rw,noatime,compress=zstd,discard=async,space_cache=v2"
        mount -o $sv,subvol=@ /dev/mapper/luks /mnt
18. Create mountpoint directories

        mkdir -p /mnt/{home,var/log,var/cache,.snapshots,swap,btrfs}
19. Mount the subvolumes and BTRFS root volume

        mount -o $sv,subvol=@home /dev/mapper/luks /mnt/home
        mount -o $sv,subvol=@log /dev/mapper/luks /mnt/var/log
        mount -o $sv,subvol=@cache /dev/mapper/luks /mnt/var/cache
        mount -o $sv,subvol=@snap /dev/mapper/luks /mnt/.snapshots
        mount -o $sv,subvol=@swap /dev/mapper/luks /mnt/swap
        mount -o $sv,subvolid=5 /dev/mapper/luks /mnt/btrfs
20. Create an efi folder and mount the shared EFI partition

        mkdir /mnt/efi                                                #We don't need to create /boot - it will be created automatically and will sit in the @ subvolume. Only the EFI partition needs to be unencrypted
        mount /dev/sda1 /mnt/efi

# Initial installation - just the basics to prep the new filesystem
21. Configure the mirrorlist and update to get the best list of servers in your neighborhood. Replace the country code "US" with the code for whatever country you're in.

        cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak              #make a backup in case we need to revert
        reflector -c "US" -f 12 -l 10 -n 12 --save /etc/pacman.d/mirrorlist   #pacstrap should now use the most appropriate download servers
        vim /etc/pacman.conf                                                  #use nano if you don't like vi keybindings
      If you plan on playing some older games or have other need of 32-bit libraries, uncomment the following two lines:
      
            [multilib]
            Include=/etc/pacman.d/mirrorlist
      Uncomment the following lines
  
            UseSyslog
            Color
            VerbosePkgLists
      Add an extra line for a special treat
  
            ILoveCandy
      Comment out the line
  
            #NoProgressBar
      (Optional) Set `ParallelDownloads = 10` if you have a fast Internet connection.
  
      If using vi or vim, `<Esc>` to make sure you're not in edit mode, then `:wq` to write changes and quit.
  
      If using nano, `<Ctl>-s` to save and `<Ctl>-x` to quit.
22. Use `pacstrap` to populate essential packages into `/mnt`. If you have an Intel CPU, replace `amd-ucode` with `intel-ucode`. If you know you'll never use `vim`, replace with a text editor of your choice (e.g. `nano`). If you don't have a wireless adapter you can omit `iwd`.

        pacstrap -K /mnt base base-devel linux linux-headers linux-firmware sudo vim amd-ucode btrfs-progs networkmanager sbctl cryptsetup bash-completion iwd git refind efibootmgr
23. Copy the modified `pacman.conf` into the new filesystem

        cp -f /etc/pacman.conf /mnt/etc/
24. Generate fstab and add permissions options to the efi partition

        genfstab -U /mnt >> /mnt/etc/fstab
        vim /mnt/etc/fstab                                      #use your favorite editor
    Add the following options the end of the options for the line that mounts the efi partition:

        fmask=0137,dmask=0027
    Save and quit the editor.
# Initial installation (cont'd): working inside the new installation
Replace country-specific codes, languages, and locales with whatever is appropriate for you. 

25. Configure the new system as root
      
        arch-chroot /mnt
26. Set a root password

        passwd
27. Set the time zone and clock

        ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
        timedatectl set-ntp true
        hwclock --systohc --utc
28. Pick a host name

        echo "archwinPC" > /etc/hostname                      #whatever name you like
29. Set name resolution basics in the hosts file

        vim /etc/hosts                                        #or nano, or whatever
    Make the file look like this, and replace `archwinPC` with whatever name you chose for your host

            127.0.0.1            localhost
            ::1                  localhost
            127.0.1.1            archwinpc.localdomain    archwinpc
30. Set your locale

        export locale="en_US.UTF-8"                          #change this to whatever your regional locale should be
        echo "LANG=${locale}" > /etc/locale.conf
        vim /etc/locale.gen                                  #or nano, etc
    Scroll until you find the locale that matches your needs and uncomment the line. In my case I uncommented `en_US.UTF-8`. Save and quit the editor.

        locale-gen
        echo "KEYMAP=us" > /etc/vconsole.conf                #replace with whatever keymap is best for you
31. Create a user and give sudo access

        useradd -m -G wheel -s /bin/bash gilligan            #whatever username you like. You can also add to other groups here if you like
        passwd gilligan                                      #choose a password for your user
        EDITOR=vim visudo                                    #I guess this would work with nano, but I've never tried
    Scroll down until you find the line `# %wheel ALL=(ALL) ALL` and uncomment it and remove the leading whitespace. Save and quit the editor.
32. Get a list of TPM2 device(s) on the system and make a note of the driver(s) used. In my case the driver I need is `tpm_crb`.

        systemd-cryptenroll --tpm2-device=list
33. Edit mkinitcpio.conf to make sure the unified kernel images we'll be generating are aware of TPM, LUKS, and BTRFS

        vim /etc/mkinitcpio.conf
    Edit the `HOOKS` line to read `HOOKS=(base systemd microcode autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)`
    
    Add your TPM driver to `MODULES`. E.g. in my case it reads `MODULES(tpm_crb)`
    
    Add `btrfs` to `BINARIES` so that it reads `BINARIES(btrfs)`
34. Edit the kernel command line so it's aware of the encrypted filesystem layout and is able to mount /. To do this we need the `UUID` of the base partition `/dev/sdb1`. Rather than mess with copy/paste, it's easier to just dump the UUID into a file and then add the rest of the command string around it.

        blkid -s UUID -o value /dev/sdb1 > /etc/kernel/cmdline                  #this file doesn't exist yet, so this is ok
        vim /etc/kernel/cmdline                                                 #are you using vim, yet?
    Add the following text around the `<uuid>` that's already in the file (replace `amd-ucode.img` with `intel-ucode.img` if appropriate):

          rd.luks.name=<uuid>=root rd.luks.options=tpm2-device=auto root=/dev/mapper/root rw rootflags=subvol=@ initrd=amd-ucode.img initrd=initramfs-%v.img add_efi_memmap
    Save and quit the editor.
35. Configure the .preset. I've left the fallback option in place in case I decide to add an lts kernel to my installation. You can keep or omit it as you choose.

          vim /etc/mkinitcpio.d/linux.preset
    Make the file look like this:

              ALL_config="/etc/mkinitcpio.conf"
              ALL_kver="/boot/vmlinuz-linux"

              PRESETS=('default' 'fallback')

              #default_config="/etc/mkinitcpio.conf"
              #default_image="/boot/initramfs-linux.img"
              default_uki="/efi/EFI/Linux/arch-linux.efi"
              default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

              #fallback_config="/etc/mkinitcpio.conf"
              #fallback_image="/boot/initramfs-linux-fallback.img"
              fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
              fallback_options="-S autodetect"
    Save and quit the editor.
36. Create the `Linux` folder in the EFI partition

        mkdir -p /efi/EFI/Linux
37. Make unified kernel images for the 1st time

        mkinitcpio -P
38. Everything is now in place in the EFI partition. Install refind and it should self-configure so that Windows and Arch show up in the boot menu.

        refind-install
39. (Optional) Configure make to leave a few horses in the stable so we can still interact with the system when it's busy putting things together for us

        vim /etc/makepkg.conf
    Scroll down to `MAKEFLAGS` and make the line look like

            MAKEFLAGS="-j$(nproc --ignore=2)"                  #total number of threads - 2 so we always have a couple available for other tasks
    Save and quit the editor.
40. Install a UKI pacman hook to trigger a rebuild after a microcode update. As usual, replace `amd-ucode` with `intel-ucode` if appropriate for your system.

        mkdir -p /etc/pacman.d/hooks/
        vim /etc/pacman.d/hooks/ucode.hook
    Make it look like this:
    
            [Trigger]
            Operation=Install
            Operation=Upgrade
            Operation=Remove
            Type=Package
            Target=amd-ucode
            Target=linux
            
            [Action]
            Description=Update AMD Microcode module in initcpio
            Depends=mkinitcpio
            When=PostTransaction
            NeedsTargets
            Exec=/bin/sh -c '! grep -qx linux && /usr/bin/mkinitcpio -P'
    Note that this is for an edge case and will only run mkinitcpio to regenerate UKIs for a ucode update in the case that ucode was updated but `linux` was not. When `linux` updates a mkinitcpio is triggered elsewhere, which will also cover a ucode update (if there was one).

41. Exit arch-chroot so we can run a couple of more config tasks before rebooting.

        exit
42. Enable some services we'll need, and mask one that's obsolete

        systemctl --root /mnt enable systemd-resolved systemd-timesyncd NetworkManager
        systemctl --root /mnt mask systemd-networkd
43. Reboot to finish installation and make sure to stop by your UEFI BIOS on the way - if the `--firmware-setup` option below doesn't kick you into the UEFI setup for your mainboard, you'll have to do it the old fashioned way by mashing `<F2>` or `<Del>` or whatever your system requires during POST.

        sync
        systemctl reboot --firmware-setup
# Configure secure boot
44. Place the system into secure boot setup mode. Instructions for how to do this differ by mainboard manufacturer - check your owner's manual or search online to find instructions that work for your system.
45. Boot the system. The refind installer replaced bootx64.efi, and since there's only one EFI partition on the system it _should_ just boot right into the refind menu. Select Arch (regular, not the fallback), and continue to load the OS.
- Note that you'll need to type in the LUKS password for your encrypted partition in order to proceed.
46. Log in with the user you created (in my case, gilligan)
47. Verify that the system is in secure boot setup mode.

        sbctl status
  The above command should produce the following output:
  
  ![sbsetup](https://github.com/user-attachments/assets/2b57b1cf-d4ff-48bc-a017-e382e3761dc0)

48. Create and enroll secure boot keys. Make sure to use the `-m` option to enroll the Microsoft keys alonside the new Arch secure boot keys. For a single-OS system this isn't critical, but for the dual-boot setup here it's mandatory.

        sudo sbctl create-keys
        sudo sbctl enroll-keys -m
        sudo sbctl verify
    The last command shold provide a list of items that need to be signed.
49. Sign the bootloader and both primary and fallback (if you have one) Arch .efi files, along with the refind boot manager and its associated BTRFS driver

        sudo sbctl sign -s /efi/EFI/boot/bootx64.efi
        sudo sbctl sign -s /efi/EFI/Linux/arch-linux.efi
        sudo sbctl sign -s /efi/EFI/Linux/arch-linux-fallback.efi
        sudo sbctl sign -s /efi/EFI/refind/refind_x64.efi
        sudo sbctl sign -s /efi/EFI/refind/drivers_x64/btrfs_x64.efi
    The `-s` option is really handy, since sbctl will file away the info on the signed file in a database it maintains. When you regenerate UKIs it should automatically re-sign the new files, so you'll never have to repeat this step.
50. Re-install the linux kernel to test the auto-signing feature. You should see a message that the newly-created .efi file(s) were signed after being created.

        sudo pacman -S linux
51. Reboot and check that the UEFI BIOS is now in standard secure boot mode (not setup mode).

        shutdown -r now
# Enable auto-TPM2 decryption of the LUKS volume so you won't need to type in the password every time the system boots
52. Reboot into Arch and log in as your user.
53. Create a recovery key so your LUKS partition can be unlocked in the event of an emergency (e.g. auto-TPM unlocking gets broken and you've lost/forgotten the passphrase). Save this directly to a file and transfer to a USB drive for storage on another machine, printing and archiving, etc. Don't keep it on the encrypted volume... it won't do much good if it's encrypted and you need it to decrypt.

        sudo systemd-cryptenroll /dev/sdb1 --recovery-key > archwinPC-btrfs-recovery.txt
54. Create and bind a LUKS key to the TPM2 on your system. I choose to bind to PCRs 0, 2, 4, and 7, although most online how-tos only use 0 and 7. If the system measurements during boot change, the values in these PCRs will be different and the TPM will refuse to release the unlock key. What to do if this happens is covered in step 55.

        sudo systemd-cryptenroll /dev/sdb1 --tpm2-device=auto --tpm2-pcrs=0,2,4,7
    - PCR 0 measures the system firmware
    - PCR 2 measures the extended firmware (e.g. OPROMs)
    - PCR 4 measures the boot manager code
    - PCR 7 measures the secure boot state.
    
    A full description of the PCRs can be <a href="https://wiki.archlinux.org/title/Trusted_Platform_Module">found on the Arch wiki here</a>.
55. (Recovery) If any of the boot-time measurements made by the TPM system aren't correct, auto-unlocking the LUKS volume won't take place. A common scenario is Windows doing something in the EFI partition that causes a red flag for the TPM measurements. This happened to me once during testing of this write-up. If this happens:

      a) boot into Arch and use the password you created to unlock the LUKS partition
    
      b) run the following command to wipe the old keys from the TPM PCRs and re-enroll (must be done all at once in a single command)

        sudo systemd-cryptenroll /dev/sdb1 --wipe-slot=tpm2 --tpm2-device=auto --tpm2-pcrs=0,2,4,7

# Final configuration - add a swapfile
This could be done earlier, but whatever... for swapfile size it's really up to you. I like to have the hibernation option available to me, so my swapfile size is chosen to be larger than my total system memory. I've seen differnt rules of thumb for this, 1.5X, 2X, (1+sqrt(2))X, etc. I chose to keep it simple with my test setup and used 1.5X... for 16GB RAM this results in a 24GB swap file. If you don't want to use hibernation, go smaller if you like. Change the size below to suit your needs.

56. Create a swapfile, enable it, and make sure fstab knows about it

        sudo btrfs filesystem mkswapfile --size 24g --uuid clear /swap/swapfile
        sudo swapon /swap/swapfile
        sudo vim /etc/fstab                                                       #or nano...      
     Add the line:
            
            /swap/swapfile    none  swap  defaults    0 0
     Save and quit the editor.
  
57. We need to know how to resume from hibernation... for this an offset into the root volume is needed. Grab the required offset by executing:

        sudo btrfs inspect-internal map-swapfile -r /swap/swapfile
58. Now edit the kernel command line with this offset:

        sudo vim /etc/kernel/cmdline
    Append the following at the end of the line, using the `<offset>` reported in step 57:

          resume=/dev/mapper/root resume_offset=<offset>
59. Make an addition to mkinitcpio.conf to support hibernation

        sudo vim /etc/mkinitcpio.conf
    Append `resume` in the `HOOKS` line so that it looks like:

            HOOKS=(base systemd microcode autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck resume)
    Save and quit the editor.
60. Rebuild UKIs with hibernation included

        sudo mkinitcpio -P
# Post-setup
Try booting into both Arch and Windows - refind should allow you to easily select either with the arrow keys, and Arch should auto-decrypt the LUKS volume with the help of the TPM (as should Windows Bitlocker, if that's in use). Next steps I'm not going to cover include: installing appropriate graphics drivers, choosing and installing a desktop environment or window manager, getting timeshift setup to take automatic BTRFS snapshots, adding pacman hooks to take snapshots before upgrading key packages (kernels, drivers, etc)... the rest it I'll leave to you. 

Final thoughts: I'm not a Linux longbeard... there may be inefficiencies or outright mistakes in the above instructions, but I've done my best to re-create what worked for me. Comments/corrections welcome... happy dual-booting!
