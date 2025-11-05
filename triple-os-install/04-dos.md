# Part 4: Install FreeDOS along Ventoy in VHD files

## Step 1: Create the FreeDOS VM
Download the latest [FreeDOS LiveCD](https://www.freedos.org/download/)

Run VirtualBox
- New virtual machine
- Name it VentDos
- For VM folder, use the `Ventoy` partition (VirtualBox will create a directory "VentDos")
  - This is not a typo, the `VENTFAT` partition is reserved to *share files* with DOS
  - Don't use `VentMore` either, because it is outside the 128 GiB limit
- Choose the FD##LIVE.iso install ISO
- Change virtual disk type to VHD, check "pre-allocate full size"
- Make it 1 GiB
- Finish, install FreeDOS
- When complete (`C:\>`), run `vol`
- Note output, for example: `FREEDOS2025`

## Step 2: Add `grub4dos` to `/ventoy/`
On host
- Download the latest grub4dos for legacy BIOS. \
  \
	This is mentally challenging, as both the [website](http://grub4dos.chenall.net/) and [github repo](https://github.com/chenall/grub4dos) of the current maintainer are in Chinese, and only the website hosts compiled binaries, and the website is horrible in many ways. Nevertheless, it seems legit and surprisingly up-to-date.\
  \
  Latest direct link at the time of writing: \
  http://dl.grub4dos.chenall.net/grub4dos-0.4.6a-2025-08-19.7z
- Extract `grub.exe` from the archive, place it on the `Ventoy` partition as `/ventoy/grub4dos.exe`

## Step 3: The custom grub menu
- Create a directory `ventoy` on the `Ventoy` partition
- Edit `/ventoy/ventoy_grub.cfg`, add this content:
	```sh
	# FreeDOS
	set freedos_vhd="/VentDos/VentDos.vhd"
	set freedos_vol="FREEDOS2025" # as noted before
	set shared_vol="VENTFAT" # shared FAT16 partition name
	# Show menuentry only if booted in compatible BIOS mode
	if [ "${grub_platform}_${grub_cpu}" = "pc_i386" ]; then
		menuentry "FreeDOS" {
			set opts="
				map --floppies=2;
				map ${freedos_vhd} (hd);
				vol ${shared_vol};
				map ()+1 (fd1);
				map --hook;
				map (fd1) (hd0);
				map --hook;
				vol ${freedos_vol};
				chainloader /kernel.sys
			"
			linux ${vtoy_iso_part}/ventoy/grub4dos.exe --skip-MBR-test --config-file=${opts}
		}
	else
		menuentry "FreeDOS [UNAVAILABLE IN EFI]" {
			echo "Native DOS requires 'Legacy Boot' or 'CSM'"
			sleep 5
		}
	fi
	```
- Boot in non-EFI mode (Legacy BIOS support required)

The FreeDOS VHD will appear as drive `C:`, the shared partition `VENTFAT` will appear as floppy drive `B:` - both are writable.

## A quirk to be aware of
Since grub4dos has limited functionality when it comes to exFAT, the modification time of the VHD file will not update when altered by DOS. Since size is fixed as well, the VHD looks untouched from the outside. This can be a problem if you use backup tools which skip files based on this metadata.

## Congratulations
If you successfully followed the guide, you have a very complete set of portable OS in your pocket and learned alot about the power of GRUB, grub4dos and how the Debian boot process can be modified in significant ways.

Visit the [FAQ](99-faq.md) for some extra information. Enjoy!
