# Part 3: Install Ubuntu (or Debian) along Ventoy in VHD files

NOTE: this does *not* use the Ventoy Linux vDisk Boot Plugin. See the [FAQ](99-faq.md#why-not-use-the-ventoy-linux-vdisk-boot-plugin) why.

## Step 1: Create the Ubuntu VM
Run VirtualBox
- New virtual machine
- Name it VentUbuntu
- For VM folder, use the "VentMore" partition (VirtualBox will create a directory "VentUbuntu")
- Use unattended setup for convenience
- Set preferred username, password, host and domain name
- Do not auto-install guest additions - at the time of writing the script is broken and will cause an error after installation, guest integration is mostly preinstalled anyways
- Change virtual disk type to VHD, check "pre-allocate full size"
- Make it 30 GiB or bigger in size
- Finish, wait for the install to complete, shutdown the VM

## Step 2: Move the bootloader to `VentUbuntuBoot.vhd`
Run VirtualBox
- Add a virtual disk: 1 GiB `VentUbuntuBoot.vhd`, pre-allocated, located along `VentUbuntu.vhd`.
- Re-order the disks to be
	- VentUbuntuBoot.vhd on port 0
	- VentUbuntu.vhd on port 1

At this point, the VM is only bootable with the VirtualBox boot menu
- Boot the VM, press F12, select disk 2 to boot

In the booted VM in a terminal
- `lsblk` # Verify you see the new disk sda = 1 GiB
- `sudo parted /dev/sda` # create boot partitions
  ```
  mklabel gpt
  mkpart bios_boot 1MiB 3MiB
  set 1 bios_grub on
  mkpart boot 3MiB 100%
  ```
- `sudo e2label /dev/sdb2 VentUbuntuRoot` # For easier referencing, label the rootfs
- `sudo mkfs.ext4 -L VentUbuntuBoot /dev/sda2` # Format the new home for `/boot`
- `sudo mkdir -p /tmp/mntBoot && sudo mount LABEL=VentUbuntuBoot /tmp/mntBoot` # Mount it
- `sudo cp -ax /boot/* /tmp/mntBoot` # Copy the current `/boot`
- `ls /tmp/mntBoot` # Verify the filesystem holds `vmlinuz`, `initrd.img`
- `sudo editor /etc/fstab` # Add the following line to fstab
  ```
  /dev/disk/by-label/VentUbuntuBoot /boot ext4 defaults,x-systemd.automount 0 2
  ```
- `sudo mv /boot /boot.old` # Rename the old `/boot` so it is never considered for updates
- `sudo mkdir /boot && sudo chattr +i /boot` # Create an immutable dir for mountpoint
- `sudo systemctl daemon-reload && sudo systemctl restart boot.automount` # Load the automounter
- `ls /boot` # Verify the automount works and `vmlinuz`, `initrd.img` are present
- `sudo update-grub` # Update the GRUB config
- `sudo grub-install /dev/sda` # Update the GRUB bootloader

Reboot the VM to verify it boots fine (no need to switch disk with F12)

## Step 3: Open VHD files at boot
We add scripts to open the VHD files in the initial ramdisk which is directly loaded by GRUB. The standard boot process can pick up from there.

Determine what tooling your distro chose to install (default since Ubuntu 25.10 is Dracut):
- `sudo which -s dracut && echo "Dracut found" || echo "Dracut not found, assuming initramfs-tools"`
- [Skip to Dracut if it was found](#open-vhd-files-at-boot-variant-for-dracut)

### Open VHD files at boot: Variant for initramfs-tools
Make `kpartx` available in initramfs:
- `sudo apt install kpartx`
- `sudo editor "/etc/initramfs-tools/hooks/VentUbuntuVHD"`
  ```sh
  #!/bin/sh
  PREREQ=""
  . /usr/share/initramfs-tools/hook-functions

  copy_exec /sbin/kpartx /sbin
  ```
- `sudo chmod +x "/etc/initramfs-tools/hooks/VentUbuntuVHD"`

Make exFAT available in initramfs:
- `sudo editor "/etc/initramfs-tools/modules"`
  ```
  # Required for VHD losetup
  exfat
  ```

This initramfs "top" phase script will mount the exFAT partitions `Ventoy` and `VentMore` and losetup the VHD files
- `sudo editor "/etc/initramfs-tools/scripts/local-top/VentUbuntuVHD"`
  ```sh
  #!/bin/sh

  case $1 in
  prereqs)
    exit 0
    ;;
  esac

  echo -n "Waiting for boot files to be available ..."
  while [ ! -e "/dev/disk/by-label/VentMore" ]; do
    if [ -e "/dev/disk/by-label/VentUbuntuBoot" ]; then
      echo " no need to losetup VHDs, continue boot"
      exit 0
    fi
    sleep 1
    echo -n "."
  done
  echo ""

  echo "Mount VentMore ..."
  mkdir -p "/mnt/VentMore"
  mount -t exfat -o uid=1000,gid=1000 "/dev/disk/by-label/VentMore" "/mnt/VentMore"

  echo "Mount Ventoy ..."
  mkdir -p "/mnt/Ventoy"
  mount -t exfat -o uid=1000,gid=1000 "/dev/disk/by-label/Ventoy" "/mnt/Ventoy"

  echo "losetup VHD files ..."
  loboot=$(losetup -f)

  # Expecting the /boot VHD on either partition
  if [ -e "/mnt/VentMore/VentUbuntu/VentUbuntuBoot.vhd" ]
  then
    losetup "$loboot" "/mnt/VentMore/VentUbuntu/VentUbuntuBoot.vhd"
  else
    losetup "$loboot" "/mnt/Ventoy/VentUbuntu/VentUbuntuBoot.vhd"
  fi
  kpartx -a "$loboot"
  
  loroot=$(losetup -f)
  losetup "$loroot" "/mnt/VentMore/VentUbuntu/VentUbuntu.vhd"
  kpartx -a "$loroot"

  # From this point on, the standard scripts can see and mount / and /boot
  ```
- `sudo chmod +x "/etc/initramfs-tools/scripts/local-top/VentUbuntuVHD"`

The following "bottom" phase script will move the extfs mount from initramfs to rootfs
- `sudo editor "/etc/initramfs-tools/scripts/local-bottom/VentUbuntuVHD"`
  ```sh
  #!/bin/sh
  case $1 in
  prereqs)
          exit 0
          ;;
  esac

  # Move the mounts so they can be re-used
  mount --move "/mnt/VentMore" "/root/mnt/VentMore"
  mount --move "/mnt/Ventoy" "/root/mnt/Ventoy"
  ```
- `sudo chmod +x "/etc/initramfs-tools/scripts/local-bottom/VentUbuntuVHD"`
- `sudo mkdir "/mnt/VentMore" && sudo chattr +i "/mnt/VentMore"` # Create destinations
- `sudo mkdir "/mnt/Ventoy" && sudo chattr +i "/mnt/Ventoy"`
- `sudo update-initramfs -u -k all` # Update the initramfs

For a clean shutdown, the VHD housing filesystems must be closed. `initramfs-tools` [provides no infrastructure](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=778849) for this, so we use this rather hacky approach to force all filesystems read-only. Note that rootfs **already is** read-only at this stage and the only realistically affected fs should be the ones backing our loop devices, which are then clean for poweroff.

- `sudo editor "/lib/systemd/system-shutdown/VentUbuntuVHD.shutdown"`
  ```sh
  #!/bin/sh

  # force leftover filesystems into read-only
  echo s > /proc/sysrq-trigger
  echo u > /proc/sysrq-trigger
  ```
- `sudo chmod +x "/lib/systemd/system-shutdown/VentUbuntuVHD.shutdown"`

Done. [Skip to step 4](#step-4-customize-grub-inside-and-outside-vm-boot-mode).

### Open VHD files at boot: Variant for Dracut
Create a dracut module:
- `sudo mkdir /usr/lib/dracut/modules.d/99ventubuntu`
- `sudo editor /usr/lib/dracut/modules.d/99ventubuntu/module-setup.sh`
  ```bash
  #!/bin/bash

  check() {
      return 0
  }

  depends() {
      echo "rootfs-block"
      # return 0 installs module by default
      return 0
  }

  installkernel() {
      hostonly='' instmods -c exfat
  }

  install() {
      inst_hook initqueue/settled 10 "$moddir/ventubuntu-vhd.sh"
      inst_hook pre-pivot 10 "$moddir/ventubuntu-mntmove.sh"
  }

  ```
- `sudo chmod +x /usr/lib/dracut/modules.d/99ventubuntu/module-setup.sh`

Script mounting the exfat filesystems:
- `sudo editor /usr/lib/dracut/modules.d/99ventubuntu/ventubuntu-vhd.sh`
  ```sh
  #!/bin/sh

  # user and group IDs for the mounted USB volumes
  MNT_UID=1000
  MNT_GID=1000

  # mount present USB volumes
  usb_mount() {
    if [ -e "$1" ] && [ ! -e "$2" ]; then
        mkdir -p "$2"
        mount -o uid=$MNT_UID,gid=$MNT_GID "$1" "$2"
    fi
  }

  usb_mount "/dev/disk/by-label/Ventoy" "/mnt/Ventoy"
  usb_mount "/dev/disk/by-label/VentMore" "/mnt/VentMore"

  # search and losetup VHD files
  vhd_search() {
    if [ ! -e "$1" ]; then
        # blockdevice is missing, search for VHD
        if [ -e "/mnt/VentMore/$2" ]; then
            losetup -Pf "/mnt/VentMore/$2"
            need_shutdown
        elif [ -e "/mnt/Ventoy/$2" ]; then
            losetup -Pf "/mnt/Ventoy/$2"
            need_shutdown
        fi
    fi
  }

  vhd_search "/dev/disk/by-label/VentUbuntu" "VentUbuntu/VentUbuntu.vhd"
  vhd_search "/dev/disk/by-label/VentUbuntuBoot" "VentUbuntu/VentUbuntuBoot.vhd"

  ```
- `sudo chmod +x /usr/lib/dracut/modules.d/99ventubuntu/ventubuntu-vhd.sh`

Script to move the mounts into the rootfs:
- `sudo editor /usr/lib/dracut/modules.d/99ventubuntu/ventubuntu-mntmove.sh`
  ```sh
  #!/bin/sh

  mntmove() {
      if [ -e "$1" ]; then
          mount --bind "$1" "${NEWROOT}$1"
      fi
  }

  mntmove "/mnt/Ventoy"
  mntmove "/mnt/VentMore"
  ```
- `sudo chmod +x /usr/lib/dracut/modules.d/99ventubuntu/ventubuntu-mntmove.sh`
- `sudo mkdir "/mnt/Ventoy" "/mnt/VentMore"` # Create destinations
- `sudo chattr +i "/mnt/Ventoy" "/mnt/VentMore"`
- `sudo update-initramfs -u -k all` # Update the initramfs

## Step 4: Customize GRUB inside and outside VM boot mode
Inside the VM, we simplify the GRUB search, too, so `update-grub` cannot get confused:
- `sudo editor "/etc/grub.d/07-VentUbuntu"`
  ```sh
  #!/bin/sh
  set -e

  cat << 'EOF'
  # Ubuntu
  set ubuntu_cmdline="root=LABEL=VentUbuntuRoot ro net.ifnames=0 biosdevname=0"
  set ubuntu_cmdline_recovery="$ubuntu_cmdline recovery nomodeset dis_ucode_ldr"
  search --no-floppy --set ubuntu_bootfs --label VentUbuntuBoot

  menuentry "VentUbuntu" {
    set root=($ubuntu_bootfs)
    linux	/vmlinuz $ubuntu_cmdline
    initrd	/initrd.img
  }
  menuentry "VentUbuntu (recovery mode, old kernel)" {
    set root=($ubuntu_bootfs)
    linux	/vmlinuz.old $ubuntu_cmdline_recovery
    initrd	/initrd.img.old
  }
  EOF
  ```
- `sudo chmod +x "/etc/grub.d/07-VentUbuntu"`
- `sudo update-grub` # Update the grub config on the 1 GiB disk

Reboot the VM to verify the custom boot menu works. Shutdown the VM.

On host:
- Create a directory `ventoy` on the `Ventoy` partition
- Edit or create `/ventoy/ventoy_grub.cfg` and add this:
  ```sh
  # Ubuntu
  set ubuntu_cmdline="root=LABEL=VentUbuntuRoot ro net.ifnames=0 biosdevname=0"
  set ubuntu_cmdline_recovery="$ubuntu_cmdline recovery nomodeset dis_ucode_ldr"
  set ubuntu_vhd_path="VentUbuntu/VentUbuntuBoot.vhd"
  set ubuntu_vhd_partname="VentMore"
  search --no-floppy --set ubuntu_vhd_part --label $ubuntu_vhd_partname
  menuentry "VentUbuntu" {
    loopback loop9 ($ubuntu_vhd_part)/$ubuntu_vhd_path
    set root=(loop9,2)
    linux	/vmlinuz $ubuntu_cmdline
    initrd	/initrd.img
  }
  menuentry "VentUbuntu (recovery mode, old kernel)" {
    loopback loop9 ($ubuntu_vhd_part)/$ubuntu_vhd_path
    set root=(loop9,2)
    linux	/vmlinuz.old $ubuntu_cmdline_recovery
    initrd	/initrd.img.old
  }
  ```

USB boot to verify the custom Ventoy menu works (access via F6).

## (Optional) Step 5: Move `VentUbuntuBoot.vhd` to the `Ventoy` partition
If you need to boot Ubuntu on a BIOS which can only access the first partition, you can move `VentUbuntuBoot.vhd` and adjust the Ubuntu part of the `ventoy_grub.cfg`. Say you create a matching directory `VentUbuntu`, these two lines must match:
```sh
set ubuntu_vhd_path="VentUbuntu/VentUbuntuBoot.vhd"
set ubuntu_vhd_partname="Ventoy"
```

The VirtualBox VM unfortunately becomes less portable in this case because the location of `VentUbuntuBoot.vhd` is no longer relative to the `.vbox`file but an absolute path. A workaround is to copy over the `VentUbuntuBoot.vhd` whenever needed and only temporarily make this change. Or have the entire `VentUbuntu` directory reside on the `Ventoy` partition, taking up a significant amount of the 120 GiB partition.

## Up next: FreeDOS
Continue with [Part 4: Install FreeDOS along Ventoy in VHD files](04-dos.md)
