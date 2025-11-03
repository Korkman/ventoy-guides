# FAQ

## General

### Why is the Ventoy partition 120 GiB?
Bad BIOS implementations cannot access USB storage beyond 128 GiB. There are even worse cases, so if you don't need 120 GiB for ISOs, make it even smaller, like 60 or 30 GiB. You can still store more ISOs on the VentMore partition and access them with F2 (Browse) in the Ventoy boot menu, albeit without advanced features like persistence. Note that the DOS VHD files and the placement of the VENTFAT partition are also kept within this area because DOS relies on BIOS for storage access.

### Can I have multiple installs of the same OS?
Yes, just choose a unique prefix for use in your `ventoy_grub.cfg`. All lines setting variables set them globally when not inside a menuentry. They are outside of menuentries so more variants of menuentries take up less lines to create.

### My Ventoy boot menu gets cluttered with unbootable VHD files!
Set a directory `/ISOs` to be the root for actual ISOs.

Create or edit `/ventoy/ventoy.json` on the `Ventoy` partition:
```json
{
  "control": [
    { "VTOY_DEFAULT_SEARCH_ROOT": "/ISOs" }
  ]
}  
```

## Windows

### Why have a full portable Windows install and not, say, a WinPE ISO?
Windows PE based ISOs are great and you should definitely priorise having one over a full install. The full install has some advantages, though. See this table on how they complement each other.

| Feature | WinPE ISO | Full install |
| ------- | --------- | ------------ |
| Application compatibility | Limited | Very good |
| Customization | Rebuild | Live |
| Drivers | Rebuild, limited | Live |
| Portability | Good | Bad |
| Bluescreens | Less | More |

### Why a dedicated partition for Windows?
Windows does weird things to the filesystem on which the VHD resides. This is to limit the blast radius.

### Why a pre-allocated VHD?
So Windows does less weird things to the filesystem on which the VHD resides. Specifically, dynamic size VHDs will "expand" to their full size on boot and revert back when shutdown. This may fail and accumulate significant unusable and unrecoverable space on the filesystem.

### Why exFAT for storing the VHD files?
Because it has less overhead and so journalling takes place only once (the virtual disk is NTFS formatted).

### About Windows Activation, licensing
I honestly do not know if this use-case is covered by presumably existing Windows licenses present on owned host systems. The portable Windows install will activate with digital licenses stored in host BIOS, which is actually unintended. When I find a way to prevent this from happening, I'll add it to the guide. Staying unactivated is entirely fine for a portable rescue system.

### Can I use Bitlocker?
At the time of writing the answer is no. The Ventoy provided `ventoy_vhdboot.img` (dated 2021) will fail to unlock the drive. The parts are there (it should be possible), but no password prompt appears.

### Is this Windows To Go?
No. Windows To Go was a dedicated enterprise product from MS and was discontinued in 2019.

### 80 GiB! Does it have to be so big?
Don't be fooled by the initial usage of about 15 GiB - it will grow. You can try [Tiny11](https://github.com/ntdevlabs/tiny11builder) to shave off a few GiB, but it will still be hefty compared to WinPE ISOs.

### Can I use NTFS drive compression inside the VHD?
Yes. To do so:
- Right-click drive C: in the booted Windows
- Select `Properties`
- Check `Use drive compression ...`
- Apply to all files, click `Skip all` when uncompressable files are encountered

### A Windows Update keeps failing and is rolled back
Windows Feature Updates tend to fail. To fix them, perform an inplace reinstall which will keep your installed software and config:
- Boot Windows in VM
- On host, download Windows install ISO and select for VM DVD drive
- Disconnect VM network
- In VM, unblock Windows Updates and run `D:\sources\setupprep.exe /product server`, assuming `D:` is your DVD drive
- Select not to download updates now

### The Ventoy boot menu locks up and nothing happens
Watch the disk activity LED if present. It may actually be booting. If this is on new hardware, the display driver might be missing and Windows will queue a driver download. If network is unavailable, shut down cleanly via a short press on the power button. Open Windows Update, which should resume to install the driver, in VirtualBox VM boot or on a different physical host.

### Other caveats?
- Encountering new hardware triggers automatic driver installs. This can cause reboots which have to be attended because the Ventoy boot menu has to be answered on every one.
- Some rare software titles require secure boot and TPM, and probably won't like the TPM keys to change. In other words: Software which deliberately sabotages portability still won't be portable.

## Ubuntu

### Why Ubuntu?
Most Debian-based distro should work the same. Ubuntu just happens to be a very popular one and is therefore used for the guide. When using MX Linux, remember it defaults to SysV init so the systemd specific instructions will fail (they are optional in this instance).

### Why so complicated?
Debian wasn't made to boot from VHD files. There are other ways to install and integrate it with Ventoy, but this method aims to be bootable both virtually and physically and to be easy to maintain (migrate to new USB disks, etc.).

### Why not use the Ventoy Linux vDisk Boot Plugin?
The key phrase in the vDisk documentation is:

| You need to run the vtoyboot script again after the update or there is a certain probability that the vDisk will not boot next boot

My guide specifically uses the tooling available in Debian so updates are reliable and do not require any further tinkering once setup.

### Why pre-allocated VHDs?
So they can be attached with losetup. The loop driver does not actually support the VHD format. We're exploiting the fact that the VHD metadata is at the end of the file, which is otherwise just a 1:1 mapping.

### Why split off /boot?
Two reasons. One is to support LUKS encryption and the other is so you can move it to the smaller `Ventoy` partition. If you encounter a BIOS which can't access beyond 128 GiB, that will allow booting Ubuntu anyways.

### Why exFAT for storing the VHD files?
Because it has less overhead and so journalling takes place only once (the virtual disk is ext4 formatted).

### Can I add encryption?
Yes. The easiest is of course home directory encryption, or containers like VeraCrypt, Cryptomator. For full disk encryption (relatively speaking), do not use unattended setup and create a dedicated, unencrypted /boot partition.

### Caveats
- It *may* break when boot components of Debian change drastically, most notably `grub2`, `update-grub` and the `initramfs-tools`

## FreeDOS

### Why the VENTFAT partition?
For easy sharing of files with DOS, like a BIOS image for flashing or similar. The partition placement is still within the 128 GiB BIOS limit, so it should be accessible on most hosts. As mentioned earlier, the Ventoy partition can be made even smaller (6 GiB anyone?) for maximum compatibility.

### Where is the shared partition VENTFAT?
"Floppy" drive B:. That's a large floppy, yeah.

### Can I install MS-DOS instead?
Yes. Install it as usual in the VM first. Adapt the guide: change `kernel.sys` to `io.sys`. Also update the volume name `FREEDOS2025` accordingly. Then attempt physical boot.

### Can I install Windows 9x instead?
We're entering "Why stop here?" territory. In theory, yes, but I've had no luck. And ended up corrupting the Ventoy bootloader (Win 9x bypassing the mapping to access the drive directly?).

If you want to give it a shot: For FAT32, grub4dos cannot use `vol` to search to boot partition. You will have to swap it with a `uuid` search. Bear in mind booting win9x on physical (or even virtual) hardware may need significant driver gathering and installation, and will be less portable. If tempted, prefer Windows 98 SE as it doesn't need FIX95CPU patching for fast CPUs.
