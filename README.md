[update-readmes]   Mode: rewrite — migrating to template structure...
# qcom-deb-images

[![Built with Ona](https://ona.com/build-with-ona.svg)](https://app.ona.com/#https://github.com/Interested-Deving-1896/qcom-deb-images)

<!-- AI:start:what-it-does -->
This project provides build scripts for generating Debian-based Linux images tailored for Qualcomm hardware. It addresses the need for reproducible and configurable image creation, supporting use cases such as development, testing, and deployment on Qualcomm platforms. It is primarily used by developers and system integrators working with Qualcomm SoCs.
<!-- AI:end:what-it-does -->

## Architecture

<!-- AI:start:architecture -->
The project builds Debian-based images for Qualcomm platforms using `debos`. Key components include:

1. **Makefile**: Defines build targets for UFS and SD card images, leveraging `debos` with configurable options like `FAKEMACHINE_BACKEND` and `DEBOS_OPTS`. It also supports testing via `pytest`.
2. **Debos Recipes**: YAML files in `debos-recipes/` specify the steps for creating the root filesystem (`qualcomm-linux-debian-rootfs.yaml`) and disk images (`qualcomm-linux-debian-image.yaml`).
3. **CI Workflows**: Located in `.github/workflows/`, these automate builds, tests, and static checks for various configurations and scenarios.
4. **Scripts and Overlays**: The `scripts/` directory contains helper scripts, while `overlay-debs/` provides additional Debian packages for customization.
5. **Kernel Configurations**: The `kernel-configs/` directory holds configuration files for building custom kernels.

Directory structure:
```plaintext
.
├── .github/
│   └── workflows/          # CI workflow definitions
├── debos-recipes/          # YAML recipes for image creation
├── kernel-configs/         # Kernel configuration files
├── overlay-debs/           # Custom Debian packages
├── scripts/                # Helper scripts
├── Makefile                # Build and test targets
├── README.md               # Project documentation
└── LICENSE.txt             # Licensing information
```

Components interact via the Makefile, which orchestrates `debos` to generate images based on the recipes and integrates with CI workflows for automation.
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
- `build-daily.yml`: Triggers daily builds of Debian images using the Makefile targets. No secrets required.
- `build-on-pr.yml`: Runs image builds and tests for pull requests. No secrets required.
- `build-on-push.yml`: Executes builds and tests on push events to main branches. No secrets required.
- `build-overlay-deb.yml`: Builds overlay Debian packages. Requires `GH_TOKEN` for repository access.
- `debos.yml`: Runs `debos` commands to generate images and rootfs artifacts. No secrets required.
- `lava-schema-check.yml`: Validates LAVA job schema files. No secrets required.
- `lava-test.yml`: Executes LAVA tests for hardware validation. Requires `LAVA_API_TOKEN` for LAVA server access.
- `linux-daily-linux-next.yml`: Builds Linux images daily using the `linux-next` branch. No secrets required.
- `linux-daily-qcom-next.yml`: Builds Linux images daily using the `qcom-next` branch. No secrets required.
- `linux-weekly-mainline.yml`: Builds Linux images weekly using the mainline kernel. No secrets required.
- `qcom-preflight-checks.yml`: Runs preflight checks for Qualcomm-specific configurations. No secrets required.
- `static-checks.yml`: Performs static analysis on Python scripts. No secrets required.
- `test-pr.yml`: Runs unit tests for pull requests. No secrets required.
- `u-boot.yml`: Builds U-Boot bootloader images. No secrets required.
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
[@lool](https://github.com/lool) (397 commits)  
[@basak-qcom](https://github.com/basak-qcom) (27 commits)  
[@lumag](https://github.com/lumag) (20 commits)  
[@mwasilew](https://github.com/mwasilew) (11 commits)  
[@doanac](https://github.com/doanac) (7 commits)  
[@b49020](https://github.com/b49020) (3 commits)  
[@Interested-Deving-1896](https://github.com/Interested-Deving-1896) (1 commit)  
[@mynameistechno](https://github.com/mynameistechno) (1 commit)  
[@mattface](https://github.com/mattface) (1 commit)  
[@njjetha](https://github.com/njjetha) (1 commit)  
[@ndechesne](https://github.com/ndechesne) (1 commit)  
[@SuMere](https://github.com/SuMere) (1 commit)  

*Note: This repository is a mirror. Please refer to the upstream source for additional contributions.*
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
