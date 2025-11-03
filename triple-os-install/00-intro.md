# How to add full installs of Windows, Ubuntu and FreeDOS to your Ventoy drive

## Feats of this guide
- Fully functional Windows, Ubuntu (or Debian) and DOS on one USB drive in addition to Ventoy
- All boot both physically and virtually
- All can receive and install updates
- All OS keep their data in easy to manage VHD files

## How it works
- Native boot is done via a custom `ventoy_grub.cfg`, accessed via F6 in the Ventoy boot menu
  - Windows VHD boot is provided by Ventoy and Microsoft
  - Debian VHD boot is scripted in this guide utilizing `initramfs-tools` or `dracut`
  - DOS VHD boot is provided by grub4dos (actively maintained fork)
- Virtual boot is done via VirtualBox (other Hypervisors should work as well)

## I only need `X`. Can I skip parts?
Yes. Just don't skip the first step where the partitioning is done.

## VirtualBox
[Download and install VirtualBox](https://www.virtualbox.org/). It is used extensively in this guide.

All VMs are recommended to have 4 CPU cores and 4 GiB memory minimum.

Enable Devices > Shared Clipboard > Bidirectional clipboard in all VMs for comfort.

Remove Hyper-V components (temporarily) from Windows for best performance.

## Can I use Hyper-V instead? VMWare Workstation? QEMU?
As long as fixed-size VHD files are produced, anything should work.

## Roadmap
- [Part 1: Install Ventoy, add partitions](01-ventoy.md)
- [Part 2: Install Windows 11 along Ventoy in a VHD file](02-windows.md)
- [Part 3: Install Ubuntu (or Debian) along Ventoy in VHD files](03-ubuntu.md)
- [Part 4: Install FreeDOS along Ventoy in VHD files](04-dos.md)
- [FAQ](99-faq.md)
