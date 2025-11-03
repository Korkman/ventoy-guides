# Part 2: Install Windows 11 along Ventoy in a VHD file

NOTE: For slow USB drives or connectivity, it is recommended to perform this part on a local disk and then move the VM directory to the USB drive.

## Step 1: Create the Windows VM
Run VirtualBox
- New virtual machine
- Name it "VentWin"
- For VM folder, use the "VentWin" partition (VirtualBox will create a directory "VentWin")
- Choose the Windows 11 install ISO
- Select desired OS Edition (Windows 11 Pro is a good choice)
- Use unattended setup
- Set preferred username, password, host and domain name
- Install guest additions, they won't interfere when booted natively
- Change virtual disk type to VHD, check "pre-allocate full size"
- For size, use the available space reported on `VentWin` minus 1 GiB
- Finish
- Creating the pre-allocated VHD will take a while
- When prompted, press a key to boot from CD/DVD (reset the VM if you failed)
- Unattend the installation

## Step 2: Prepare Windows to be booted from Ventoy
Recent versions of Windows 11 by default pre-encrypt the disk, which needs to be undone (future Ventoy may support encryption)
- Right click start menu, choose `System`
- Click `Privacy & security`
- Click `Device encryption`, turn it off
- Wait for decryption to complete
- Shutdown the VM

## Step 3: The custom grub menu
On the host
- Create a directory `/ventoy` on the `Ventoy` partition
- [Download the latest `ventoy_vhdboot.img.gz`](https://github.com/ventoy/vhdiso/releases) ([documentation](https://www.ventoy.net/en/plugin_vhdboot.html) for reference)
- Unzip `ventoy_vhdboot.img` into `/ventoy`.
- Edit or create /ventoy/ventoy_grub.cfg and add this
  ```sh
  # Windows
  set win11_vhd_path="/VentWin/VentWin.vhd"
  set win11_vhd_partname="VentWin"
  menuentry "Windows 11" {
      if search --no-floppy --set win11_vhd_part --label $win11_vhd_partname; then
          vhdboot_common_func "($win11_vhd_part)$win11_vhd_path"
      else
          echo "$win11_vhd_partname not found"
      fi
  }
  ```

USB boot to verify the custom Ventoy menu works (access via F6).

## (Optional) Step 4: Recommended Windows customization

- Have [UniGetUI](https://www.marticliment.com/unigetui/) install your favorite tools, create a `.ubundle` file \
  ... and / or have portable apps on the VentMore partition
- It's highly recommended to [block Windows Updates](https://www.sordum.org/9470/windows-update-blocker-v1-8/) so they run only when appropriate (prefer doing updates when booted virtually)

## Up next: Ubuntu
Continue with [Part 3: Install Ubuntu (or Debian) along Ventoy in VHD files](03-ubuntu.md)
