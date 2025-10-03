# UMIDIGI BISON 2020 Super Partition Resizing

## MediaTek and GPT signing

Many Qualcomm devices have their GPTs protected by signatures even after their bootloaders have been unlocked. This is presumably done to reduce the attack surface on their bootloaders. The boot process has to chain-load several stages and eventually read the bootloader unlock flag/data, all of which requires parsing the GPT. This means that the GPT is parsed before the bootloader knows if it has been unlocked, and thus can either always check its signature or never.

Qualcomm chose the user-hostile solution of locking down the GPT, while MediaTek apparently chose the more friendly avenue of trying to write robust GPT parsing code and patching it should vulnerabilities become known. AFAIK, locked MediaTek boot chains will not fail if you alter the GPT, but you do your own research.

## UMIDIGI BISON 2020

[This device](https://www.gsmarena.com/umidigi_bison-13110.php) is compatible with PHH-based ARM64 AB GSI builds up to Android 14, such as those by [Andy Yan](https://sourceforge.net/projects/andyyan-gsi/files/lineage-21-pre-qpr2-td/) and [Iodé](https://iode.tech/documentation/gsi/). Unfortunately, the 128 GB device comes with a meager 3.5 GB `super` partition that is way too small to flash most builds, even after deleting the `product` partition.


But for this MT6771-based device we have [firmware and SP Flash Tool](https://community.umidigi.com/forum.php?mod=viewthread&tid=27617), plus the fantastic [`mtkclient`](https://github.com/bkerler/mtkclient), both of which can recover from bricks, so I decided to give resizing `super` a try.

Note that if you want to use SP Flash Tool after resizing `super`, you should use the included patched scatter file and your choice of patched non-empty `super` image.

## How to resize `super`

We want to enlarge `super` from to 3.5 GB to 8 GB, making space for it by moving and shrinking `cache` and `userdata`. The vestigial `cache` will be shrank to 64 MB, and `userdata` will occupy the rest of the eMMC.

1. Flash the latest stock ROM linked above using the SPFT.
2. Unlock the bootloader (OEM unlock).
3. Make a backup and flash the patched GPT.
4. Reformat `cache` and `userdata`.
5. Resize `super`.
6. Flash your desired GSI.

Repo files with `stock` in their name refer to the OEM partition scheme, while files named `patched` refer to the modified 8 GB `super` partition scheme.

## Recovery, fastboot, fastbootd

At any time you can reboot into recovery by holding down `VOL+` and then `POWER`; release both keys once the BISON boot logo shows. When the robot pic shows, hold `POWER`, click `VOL+`, and release `POWER` to access the recovery menu. From there you can reboot to fastboot (choose option "Reboot to bootloader") or enter user-space fastbootd (choose option "Enter fastboot"). **Do not confuse fastboot with fastbootd:** fastboot is part of the bootloader and runs without a Linux kernel, while fastbootd is a Linux user-space daemon process. From fastboot you can reboot back to recovery with `sudo fastboot reboot-recovery`.

## Make a backup and flash the patched GPT

In the `gpt` directory:

```sh
# Install mtkclient, connect your phone as per instructions and test:
mtk printgpt

# Read your current GPT (2 copies, primary and secondary):
./gpt-read

# Verify the MD5 hashes of your read files match those of the stock copies in this repo.
# DO NOT PROCEED IF MD5 HASHES DO NOT MATCH !!!

# It is recommended that you now make a full backup of the 128 GB eMMC of your phone.
# The process takes over 2 hours. If you choose to do it, compress it and store it somewhere:
mtk rf full-emmc-backup.img

# Write the patched GPT:
./gpt-write

# Re-read the GPT files and verify that MD5s now match the patches copies:
./gpt-read

```

## Reformat `cache` and `userdata`

Reboot to recovery, and from there to reboot to bootloader/fastboot.

```sh
# In fastboot (fastbootd will not format cache):
fastboot -w
```

## Resize `super`

You can use the artifacts attached to the releases page of this repo (if you trust me), or you can:
1. Compile `lpmake`, `lpdump` and `lpunpack`.
2. Use `super/super-stock` to unpack the contents of the stock `super` partition.
3. Use the various `super/patched-*` options build your own artifacts.

Then you can flash any of the non-empty images via fastboot (`super-patched-vendor-only.img` is recommended for GSI use):
```sh
# In fastboot:
fastboot erase super
fastboot flash super-patched-*.img
```

Or you can flash the empty image (included in the repo for your convenience) via fastbootd:
```sh
# In fastbootd:

# Wipe and apply the empty dynamic partition layout:
fastboot wipe-super super-patched-empty.img

# Flash the stock vendor partition required for GSI use:
fastboot flash vendor vendor.img
```

## Flash your desired GSI

PHH-based ARM64 AB GSI builds up to Android 14 should work on this device:

```sh
# In fastbootd:

# You probably want to nuke product if you are coming from stock:
fastboot resize-logical-partition product 0

# Wipe system and flash your desired GSI:
fastboot resize-logical-partition system 0
fastboot flash system lineage-21.0-20250912-UNOFFICIAL-arm64_bgN-signed.img
```
