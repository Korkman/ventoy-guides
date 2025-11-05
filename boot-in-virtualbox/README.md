# How to boot USB drives in VirtualBox

## Table of contents
- [USB boot in EFI mode](#virtual-usb-efi-boot)
- [USB boot in BIOS mode](#virtual-usb-bios-boot)
  - [Warning for Windows and macOS users](#warning-for-windows-and-macos-users)
  - [USB boot in BIOS mode on Windows](#virtual-usb-bios-boot-on-windows)
  - [USB boot in BIOS mode on Linux](#virtual-usb-bios-boot-on-linux)
  - [USB boot in BIOS mode on macOS](#virtual-usb-bios-boot-on-macos)

## Virtual USB EFI boot

Run VirtualBox
- Create new VM
- Name it "USB EFI boot"
- Do not select any install media
- Select Microsoft Windows 11 for OS (this presets a Secure Boot enabled EFI machine)
- 4 GiB of RAM, half your CPU cores
- Select no hard drive
- Finish
- Plug in your USB drive
- Edit the VM, go to USB section
- Verify USB 3.0 is available and selected
  - If not, cancel the dialog and install the extension pack from the [VirtualBox downloads page](https://www.virtualbox.org/wiki/Downloads)
- Add a filter for an existing device ("plus" symbol)
  - Identifying the USB drive from a list of USB vendor and product IDs / names can be tricky, especially with generic names. When in doubt eject the drive, screenshot the list, insert the drive again and see what was added.
- At the time of writing, Windows should be booted with VBoxSVGA graphics while Linux uses the newer VMSVGA
- OK

Now when you run this VM, VirtualBox will virtually unplug the drive from the host and pass it entirely to the VM. This is often too slow. If VirtualBox displays "No bootable option or device was found", **just reboot the VM** (Hostkey+Del) to retry with the now attached drive.

**On first boot** Secure Boot will require you to [enroll the Ventoy key](https://www.ventoy.net/en/doc_secure.html) as usual.

Closing the VM will reattach the USB drive to the host.

## Virtual USB BIOS boot

The VirtualBox BIOS implementation unfortunately doesn't provide USB boot. Hence we have to jump some hoops to pass the entire drive into the guest as a virtual hard drive. Yes, it's going to be the classic VMDK hack.

### Warning for Windows and macOS users
On Windows and macOS there is no way to link the virtual disk to the USB drive by ID. Instead it will be linked to "the n-th disk connected since startup". Obviously this can cause accidential mix-ups. Booting the wrong disk is, generally speaking, bad for your data.

**Always check your physical disk number still matches in Disk Management / diskutil.**

Also, the **size of the physical drive** is persisted in the VMDK and must match or **risk of data corruption is high**.

**So you need to keep one VMDK file for each USB drive, multiplied by physical drive number encountered.**

Example naming scheme:
```
usb-pny-pro-elite-512g-phy3.vmdk
usb-pny-pro-elite-512g-phy4.vmdk
usb-corsair-force-mp600-2tb-phy3.vmdk
usb-corsair-force-mp600-2tb-phy4.vmdk
```

On Linux we can take advantage of `/dev/disk/by-id` so mix-ups are prevented.

### Virtual USB BIOS boot on Windows
Read [warning](#warning-for-windows-and-macos-users) first. Then go on.

#### Create the VMDK
- Create a local directory for storing VMDK files, say `C:\VM Disks`
- Plug in your USB drive
- Right-click start menu, click Disk Management
- Note the **current** disk number of your USB drive
- We will be using "3" as example, which translates to `\\.\PhysicalDrive3`
- Open `cmd` or `powershell`, run
  ```powershell
  cd "C:\Program Files\Oracle\VirtualBox\"
  
  .\VBoxManage.exe createmedium disk \
  --filename "C:\VM Disks\usb-pny-pro-elite-512g-phy3.vmdk" \
  --format VMDK --variant RawDisk \
  --property RawDrive=\\.\PhysicalDrive3
  ```
#### Attach the VMDK
Run VirtualBox
- Create new VM
- Name it "USB BIOS boot"
- Do not select any install media
- Select Debian 64-bit for OS
- 2 GiB of RAM, half your CPU cores
- ***No EFI***
- Select the created VMDK as hard drive
- Finish

#### Administrator privileges
If you attempt to start the VM now, you will probably learn that without Administrator privileges, it will fail with `VERR_ACCESS_DENIED`.

To succeed, both VirtualBox and the `VBoxSDS` service (which is started on-demand and inherits privileges from VirtualBox) need to run with Administrator privileges. To avoid confusion caused by this, we set VirtualBox to always run as Administrator.

- Exit VirtualBox
- The service `VBoxSDS` will remain running in background with the privileges granted at start. Stop it now in `Task Manager` > `Services`.
- Change properties of the shortcut you use to start VirtualBox. In `Compatibility`, tick `Run this program as Administrator`

#### Disk also needs to be offlined, read-only cleared
Starting the VM will appear to work, but the disk is write-protected, with a cache in place to fool you for a while into thinking it's not.

Run `diskpart.exe`. Assuming your physical disk number is `3`, send these commands:
```
select disk 3
offline disk
attr disk clear readonly
```

And behold, the VirtualBox VM runs fine when started.

You may have noticed your disk is now invisible in `explorer.exe` because it is offline. Shutdown the VM and in `diskpart`
```
select disk 3
online disk
```

The USB drive reappears in `explorer.exe`

You will have to offline and online your USB drive manually as needed.

### Virtual USB BIOS boot on Linux

#### Create the VMDK
- Plug in your USB drive
- Open a terminal
- Identify your drive in this list. It should start with `usb-`:
  ```sh
  ls -al /dev/disk/by-id
  ```
  *Example: `usb-Force_MP_600_0123456789ABC-0:0`*
- This example creates `usb-corsair-force-mp600-2tb.vmdk` in your home directory \
  from `/dev/disk/by-id/usb-Force_MP_600_0123456789ABC-0:0`
  ```sh
  sudo VBoxManage createmedium disk \
  --filename "~/usb-corsair-force-mp600-2tb.vmdk" \
  --format VMDK --variant RawDisk \
  --property "RawDrive=/dev/disk/by-id/usb-Force_MP_600_0123456789ABC-0:0"
  ```

#### Attach the VMDK
Before using the VMDK, make sure no filesystems from the USB drive are mounted. Usually this does the trick:
```
sudo umount /dev/disk/by-id/usb-Force_MP_600_0123456789ABC-0:0-*
```
Better double-check with `lsblk` no USB partitions remain mounted.

On Linux, too, VirtualBox needs higher privileges to access raw block devices. Just run VirtualBox with `sudo`.

Run VirtualBox
- `sudo VirtualBox`
- Create new VM
- Name it "USB Phy boot"
- Do not select any install media
- Select Debian 64-bit for OS
- 2 GiB of RAM, half your CPU cores
- ***No EFI***
- Select the created VMDK as hard drive
- Finish

### Virtual USB BIOS boot on macOS
Read [warning](#warning-for-windows-and-macos-users) first. Then go on.

#### Create the VMDK
- Plug in your USB drive
- Open a terminal
- Identify your drive in this list. It should have the tag "external":
  ```sh
  diskutil list
  ```
  *Example: `/dev/disk6 (external, physical)`*
- Unmount all volumes, then create `~/usb-corsair-force-mp600-2tb-phy6.vmdk` in your home directory from `/dev/disk6`:
  ```sh
  diskutil unmountDisk /dev/disk6
  sudo VBoxManage createmedium disk \
  --filename "~/usb-corsair-force-mp600-2tb-phy6.vmdk" \
  --format VMDK --variant RawDisk \
  --property "RawDrive=/dev/disk6"
  ```

#### Attach the VMDK
Run VirtualBox
- `sudo VirtualBox`
- Create new VM
- Name it "USB Phy boot"
- Do not select any install media
- Select Debian 64-bit for OS
- 2 GiB of RAM, half your CPU cores
- ***No EFI***
- Select the created VMDK as hard drive
- Finish

