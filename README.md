[update-readmes]   Mode: rewrite — migrating to template structure...
# qcom-deb-images

[![Built with Ona](https://ona.com/build-with-ona.svg)](https://app.ona.com/#https://github.com/Interested-Deving-1896/qcom-deb-images)

<!-- AI:start:what-it-does -->
_Description pending._
<!-- AI:end:what-it-does -->

## Architecture

<!-- AI:start:architecture -->
_Architecture documentation pending._
<!-- AI:end:architecture -->

## Install

<!-- Add installation instructions here. This section is yours — the AI will not modify it. -->

```bash
git clone https://github.com/Interested-Deving-1896/qcom-deb-images.git
cd qcom-deb-images
```

## Usage


To build flashable assets for all supported boards, follow these steps:

1. (optional) build U-Boot for the RB1
    ```bash
    scripts/build-u-boot-rb1.sh
    ```

1. (optional) build a local Linux kernel deb from mainline with a recommended config fragment
    ```bash
    scripts/build-linux-deb.py kernel-configs/qemu-boot.config kernel-configs/qcom-imsdk.config kernel-configs/systemd-boot.config

    # or from linux-next:
    scripts/build-linux-deb.py --linux-next kernel-configs/qemu-boot.config kernel-configs/qcom-imsdk.config kernel-configs/systemd-boot.config
    ```

1. build tarballs of the root filesystem and DTBs
    ```bash
    debos debos-recipes/qualcomm-linux-debian-rootfs.yaml

    # (optional) if you've built a local kernel, copy it to local-debs/ and run
    # this instead:
    #debos -t localdebs:local-debs/ debos-recipes/qualcomm-linux-debian-rootfs.yaml
    ```

1. build disk and filesystem images from the root filesystem tarball
    ```bash
    # the default is to build a UFS image
    debos debos-recipes/qualcomm-linux-debian-image.yaml

    # (optional) if you want SD card images or support for eMMC boards, run
    # this as well:
    debos -t imagetype:sdcard debos-recipes/qualcomm-linux-debian-image.yaml
    ```

1. build flashable assets from downloaded boot binaries, the DTBs, and pointing at the UFS/SD card disk images
    ```bash
    debos debos-recipes/qualcomm-linux-debian-flash.yaml

    # (optional) if you've built U-Boot for the RB1, run this instead:
    #debos -t u_boot_rb1:u-boot/rb1-boot.img debos-recipes/qualcomm-linux-debian-flash.yaml

    # (optional) build only a subset of boards:
    #debos -t target_boards:qcs615-ride,qcs6490-rb3gen2-vision-kit debos-recipes/qualcomm-linux-debian-flash.yaml
    ```

1. enter Emergency Download Mode (see section below) and flash the resulting images with QDL
    ```bash
    # for RB3 Gen2 Vision Kit or UFS boards in general
    cd flash_qcs6490-rb3gen2-vision-kit_ufs
    qdl --storage ufs prog_firehose_ddr.elf rawprogram[0-9].xml patch[0-9].xml

    # for RB1 or eMMC boards in general
    cd flash_qrb2210-rb1_emmc
    qdl --allow-missing --storage emmc prog_firehose_ddr.elf rawprogram[0-9].xml patch[0-9].xml
    ```

### Debos tips

By default, debos will try to pick a fast build backend. It will prefer to use its KVM backend (`-b kvm`) when available, and otherwise a UML environment (`-b uml`). If none of these work, a solid backend is QEMU (`-b qemu`). Because the target images are arm64, building under QEMU can be really slow, especially when building from another architecture such as amd64.

To build large images, the debos resource defaults might not be sufficient. Consider raising the default debos memory and scratchsize settings. This should provide a good set of minimum defaults:
```bash
debos --fakemachine-backend qemu --memory 1GiB --scratchsize 4GiB debos-recipes/qualcomm-linux-debian-image.yaml
```

### Options for debos recipes

A few options are provided in the debos recipes; for the root filesystem recipe:
- `localdebs`: path to a directory with local deb packages to install (NB: debos expects relative pathnames)
- `xfcedesktop`: install an Xfce desktop environment; default: console only environment

For the image recipe:
- `dtb`: override the firmware provided device tree with one from the Linux kernel, e.g. `qcom/qcs6490-rb3gen2.dtb`; default: don't override
- `imagetype`: either `ufs` (the default) or `sdcard`; UFS images are named disk-ufs.img and use 4096-byte sectors and SD card images are named disk-sdcard.img and use 512-byte sectors
- `imagesize`: set the output disk image size; default: `4GiB`

For the flash recipe:
- `u_boot_rb1`: prebuilt U-Boot binary for RB1 in Android boot image format -- see below (NB: debos expects relative pathnames)
- `target_boards`: comma-separated list of board names to build (default: `all`). Accepted values are the board names defined in the flash recipe, e.g. `qcs615-ride`, `qcs6490-rb3gen2-vision-kit`, `qcs8300-ride`, `qcs9100-ride-r3`, `qrb2210-rb1`.

Note: Boards whose required device tree (.dtb) is not present in `dtbs.tar.gz` are automatically skipped during flash asset generation.

Deprecated flash options:
- `build_qcs615`, `build_qcm6490`, `build_qcs8300`, `build_qcs9100`, `build_rb1`: these per-family/per-board toggles are deprecated and will be removed. Use `target_boards` instead to select which boards to build.

Here are some example invocations:
```bash
# build the root filesystem with Xfce
debos -t xfcedesktop:true debos-recipes/qualcomm-linux-debian-rootfs.yaml

# build an image where systemd overrides the firmware device tree with the one
# for RB3 Gen2
debos -t dtb:qcom/qcs6490-rb3gen2.dtb debos-recipes/qualcomm-linux-debian-image.yaml

# build an SD card image
debos -t imagetype:sdcard debos-recipes/qualcomm-linux-debian-image.yaml

# build flash assets for a subset of boards
# (see flash recipe for accepted board names)
debos -t target_boards:qcs615-ride,qcs6490-rb3gen2-vision-kit debos-recipes/qualcomm-linux-debian-flash.yaml
```

### Flashing tips

The `disk-sdcard.img` disk image can simply be written to an SD card, albeit most Qualcomm boards boot from internal storage by default. With an SD card, the board will use boot firmware from internal storage (eMMC or UFS) and do an EFI boot from the SD card if the firmware can't boot from internal storage.

For UFS boards, if there is no need to update the boot firmware, the `disk-ufs.img` disk image can also be flashed on the first LUN of the internal UFS storage with [qdl](https://github.com/linux-msm/qdl). Create a `rawprogram-ufs.xml` file as follows:
```xml
<?xml version="1.0" ?>
<data>
  <program SECTOR_SIZE_IN_BYTES="4096" file_sector_offset="0" filename="disk-ufs.img" label="image" num_partition_sectors="0" partofsingleimage="false" physical_partition_number="0" start_sector="0"/>
</data>
```
Put the board in "emergency download mode" (EDL; see next section) and run:
```bash
qdl --storage ufs prog_firehose_ddr.elf rawprogram-ufs.xml
```
Make sure to use `prog_firehose_ddr.elf` for the target platform, such as this [version from the QCM6490 boot binaries](https://softwarecenter.qualcomm.com/download/software/chip/qualcomm_linux-spf-1-0/qualcomm-linux-spf-1-0_test_device_public/r1.0_00058.0/qcm6490-le-1-0/common/build/ufs/bin/QCM6490_bootbinaries.zip) or this [version from the RB1 rescue image](https://artifacts.codelinaro.org/artifactory/clo-549-96boards-backup/96boards/rb1/linaro/rescue/23.12/rb1-bootloader-emmc-linux-47528.zip).

### Emergency Download Mode (EDL)

In EDL mode, the board will receive a flashing program over its USB type-C cable, and that program will receive data to flash on the internal storage. This is a lower level mode than fastboot which is implemented by a higher-level bootloader.

To enter EDL mode:
1. remove power to the board
1. remove any cable from the USB type-C port
1. on some boards, it's necessary to set some DIP switches
1. press the `F_DL` button while turning the power on
1. connect a cable from the flashing host to the USB type-C port on the board
1. run qdl to flash the board

NB: It's also possible to run qdl from the host while the board is not connected, then start the board directly in EDL mode.

## Configuration

<!-- Document configuration options here. This section is yours — the AI will not modify it. -->

## CI

<!-- AI:start:ci -->
_CI documentation pending._
<!-- AI:end:ci -->

## Mirror chain

<!-- AI:start:mirror-chain -->
This repo is maintained in [`Interested-Deving-1896/qcom-deb-images`](https://github.com/Interested-Deving-1896/qcom-deb-images) and mirrored through:

```
Interested-Deving-1896/qcom-deb-images  ──►  OpenOS-Project-OSP/qcom-deb-images  ──►  OpenOS-Project-Ecosystem-OOC/qcom-deb-images
```

Changes flow downstream automatically via the hourly mirror chain in
[`fork-sync-all`](https://github.com/Interested-Deving-1896/fork-sync-all).
Direct commits to OSP or OOC are detected and opened as PRs back to `Interested-Deving-1896`.
<!-- AI:end:mirror-chain -->

## Contributors

<!-- AI:start:contributors -->
_Contributors pending._
<!-- AI:end:contributors -->

## Origins

<!-- AI:start:origins -->
_Original project — no upstream fork._
<!-- AI:end:origins -->

## Resources

<!-- AI:start:resources -->
_No additional resource files found._
<!-- AI:end:resources -->

## License

<!-- AI:start:license -->
<!-- License not detected — add a LICENSE file to this repo. -->
<!-- AI:end:license -->
