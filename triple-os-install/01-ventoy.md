# Part 1: Install Ventoy, add partitions

## Step 1: Planning the layout
All tasks should be given room to breathe in their respective file systems
- Ventoy ISOs: \
  100 GiB is plentiful and convenient \
  30 GiB is clausthrophobic but [compatible with more BIOS](99-faq.md#why-is-the-ventoy-partition-120-gib)
- Windows 11: Recommended minimum of 80 GiB (initial usage < 20 GiB)
- Ubuntu Linux: Recommended minimum of 30 GiB (initial usage < 10 GiB)
- FreeDOS: Recommended minimum of \
  64 MiB for plain install (usage is < 30 MiB) \
  512 MiB for full install (usage is < 325 MiB)
- FAT16 partition: 1 GiB for quickly sharing files with DOS (and maybe some dumb devices)

Thus a USB drive with 256 GiB or more capacity is recommended when including Windows, 64 GiB when excluded.

USB write speeds range from 4 to 4.000 MiB/s. If you go shopping, check reviews on write speeds first. My personal experience with a PNY Pro Elite V2 has been very good (around 500 MiB/s writes).

Example partition layout plan for 256 GiB:
  - "Ventoy" 120 GiB exFAT for ISOs and small VHDs (1 GiB DOS VHD, 1 GiB Ubuntu `/boot`)
  - "VENTFAT" 1 GiB FAT16 (share files with DOS)
  - "VentWin" 80 GiB exFAT for Windows VHD
  - "VentMore" 55 GiB for Ubuntu VHD and other files (ISOs accessed via F2)

Example using a 2 TiB USB NVMe:
  - "Ventoy" 120 GiB
  - "VENTFAT" 1 GiB
  - "VentWin" 200 GiB
  - "VentMore" 1.4 TiB

Example using 64 GiB:
  - "Ventoy" 30 GiB
  - "VENTFAT" 1 GiB
  - "VentMore" 33 GiB

Adjust to your preferences. The guide will assume the 256 GiB example.

## Step 2: ventoy2disk
Have [Ventoy](https://github.com/ventoy/Ventoy) installed.

Run ventoy2disk
- Select the correct destination
- Select partition style GPT
- Enter partition configuration
  - Keep exFAT filesystem
  - Preserve X GiB of space, X = total available - 120 (Ventoy partition size)
- Install
- Double check the `Ventoy` partition is about 120 GiB in size

## Step 3: Disk Management

Run Disk Management or any other capable partitioning tool (Disk Genius, gparted)
- Note how Ventoy consists of two partitions by default
- Never change size or location of these two partitions
- Add partition "VENTFAT", 1 GiB, format as FAT (in fact FAT16, not FAT32)
- Add partition "VentWin", 80 GiB, format as exFAT
- Add partition "VentMore", all remaining space (minus overprovisioning, if you're into that), format as exFAT

## Up next: Windows
Continue with [Part 2: Install Windows 11 along Ventoy in a VHD file](02-windows.md)
