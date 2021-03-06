---
title: Integrating with U-Boot
taxonomy:
    category: docs
---

In order to support rootfs rollback, Mender requires integration with U-Boot.
This page explains how to integrate U-Boot in your existing Yocto Project
build environment with Mender.

## How to integrate with U-Boot

When a device boots, it needs to decide from where to boot, and this depends on
which partitions have received which updates. Mender comes with special boot
code prepared for U-Boot which selects the boot partition
automatically, as well as handling the rollback logic.

In order to utilize this, when building with Yocto Project you need to inherit either
the `mender-full` class, or the `mender-uboot` class (the former enables all
required features for Mender and implicitly inherits the latter). For example in
your `local.conf`:

```
INHERIT += "mender-full"
```

This enables the Mender U-Boot boot code.

However, the are so many ways to boot different devices, that there is no way this
integration can be fully automatic. Therefore Mender provides special
integration points for your existing boot code to hook into so that Mender can
be a part of the normal boot process. These integration points, along with
certain required U-Boot features and configuration options, allow Mender to do
safe, automatic updates of the device, with rollback support.

## U-Boot features

A number of U-Boot features need to be enabled for Mender to work correctly, and
these should be enabled in the board support headers in U-Boot, under
`include/configs`.

1. `CONFIG_BOOTCOUNT_LIMIT`: This is required for rollback support to work. For
   example:

   ```
   #define CONFIG_BOOTCOUNT_LIMIT
   ```

2. `CONFIG_BOOTCOUNT_ENV`: This will store the boot counter in the U-Boot
   environment. This means that other `CONFIG_BOOTCOUNT_` features should be
   turned off.

3. `CONFIG_ENV_IS_IN_FAT`: This will store the U-Boot environment file on the
   FAT-based boot partition. Again, other `CONFIG_ENV_IS_IN_` features should be
   turned off.

4. `FAT_ENV_INTERFACE`, `FAT_ENV_DEVICE`, `FAT_ENV_PART` and
   `FAT_ENV_DEVICE_AND_PART` should be removed from the configuration if they
   are present. They will be automatically defined by Mender.


## Integration points

These are the current integration points:

1. `mender_setup`: This is an environment script that should be run at the very
   beginning of the boot process. It will not perform any boot steps, however it
   may modify, and potentially save, the U-Boot environment.

   For example, if your current `bootcmd` looks like this:

   ```
   bootcmd=run mmcboot
   ```

   it should be changed to:

   ```
   bootcmd=run mender_setup; run mmcboot
   ```

2. `mender_uboot_root`: This is an environment variable that contains the
   description of the device currently set to boot. Whenever a U-Boot command is
   issued that needs to access the current boot partition, this variable should
   be referenced.

   For example, if you have a script, `loadimage`, that loads the kernel from
   the filesystem, using `mmc` as the device and a `${bootpart}` variable
   reference as the partition to load from:

   ```
   loadimage=load mmc ${bootpart} ${loadaddr} ${bootdir}/${bootfile}
   ```

   it should be changed into:

   ```
   loadimage=load ${mender_uboot_root} ${loadaddr} ${bootdir}/${bootfile}
   ```

   Note that `mmc` is included in the `${mender_uboot_root}` string; it is a
   complete description of the device and partition to load from.

3. `mender_kernel_root`: This is essentially the same as the previous variable,
   but is a string tailored to the Linux kernel instead of U-Boot. This should
   be used as the `root` argument to the kernel. For example:

   ```
   bootargs=console=${console},${baudrate} root=${mmcroot}
   ```

   should be changed to:

   ```
   bootargs=console=${console},${baudrate} root=${mender_kernel_root}
   ```

4. `mender_altbootcmd`: This integration point is only needed if your setup is
   already making use of U-Boot's `altbootcmd` functionality. If not currently
   in use, this step can be skipped.

   If `altbootcmd` is being used, one first needs to disable Mender's built-in
   `altbootcmd`. To do this, the `MENDER_NO_DEFAULT_ALTBOOTCMD` define should be
   added to the board configuration header in U-Boot (inside
   `include/configs`).

   Then, at the beginning of `altbootcmd`, the call `run mender_altbootcmd`
   should be added. Like `mender_setup`, this will not perform any boot steps,
   but it may modify and potentially save the environment. Afterwards the
   `mender_uboot_root` and `mender_kernel_root` variables will refer to the
   correct partitions, taking into account the potential rollback that may
   happen because `altbootcmd` was called. After the desired alternate boot
   steps have been performed, one can either call `bootcmd` to perform a normal
   boot using the new partitions, or one can perform a different type of boot
   sequence and refer to the Mender variables directly.


## Boot configuration

There are also a few other details that need to be in place for Mender to work.

1. In a Mender based configuration, the kernel is loaded from the rootfs
   partition, not from the boot partition. This is in order to make a complete
   upgrade possible, including the kernel. Usually, in a boot partition, the
   kernel is stored in the root, but on a rootfs partition it is usually stored
   in `/boot`. Therefore, paths that refer to the location of the kernel need to
   be updated to point to this location. This is usually the case for the device tree
   and initrd files as well, if the kernel has those. For instance:

   ```
   uimage=uImage
   fdt_file=uImage.dtb
   ```

   should be changed to:

   ```
   uimage=boot/uImage
   fdt_file=boot/uImage.dtb
   ```

2. Because the kernel and associated files are loaded from a rootfs partition,
   in the majority of cases it will be an ext3 or ext4 partition. If the
   existing boot code for the board uses the `fatload` command to load the
   kernel and/or any associated files, it will need to be changed, since the
   rootfs is usually not a FAT partition. We recommend that it is replaced
   simply with `load`, since it will work in both cases, but it can also be
   replaced with either `ext2load` or `ext4load` if desired.


## Forks of U-boot

If the project is using a fork of U-Boot, some additional steps may be
required:

1. Mender has a dependency on `u-boot`, but the project's U-Boot likely has
   another name, therefore it is important to mark the project's fork as a
   component that provides `u-boot`. The example below shows how to add the
   needed directives in the `.bb` file of the U-Boot fork.

   ```
   PROVIDES += "u-boot"
   RPROVIDES_${PN} += "u-boot"
   ```

2. In the machine section of the board in question, the actual u-boot
   implementation must be selected using `PREFERRED_PROVIDER`, like this:

   ```
   PREFERRED_PROVIDER_u-boot = "u-boot-my-fork"
   ```

   Many machine configurations will probably have this in their setup already.

3. The recipe needs to include `u-boot-mender.inc`, in order to incorporate the
   patches needed for Mender to work:

   ```
   require recipes-bsp/u-boot/u-boot-mender.inc
   ```

### u-boot-fw-utils

Mender depends on userspace tools provided by `u-boot-fw-utils`. This is a
standard component in Yocto, but if using a U-Boot fork, this may need to be
added manually. If it doesn't already exist, the most straightforward approach
is to start with the `meta/recipes-bsp/u-boot/u-boot-fw-utils_*.bb` recipe found
in Yocto, and then make the necessary changes in the same fashion as done for
the main `u-boot` recipe. The two recipes should use identical source code
(e.g. the same patches applied).

Like with `u-boot`, `u-boot-fw-utils` should contain sections to define what it
provides, and that it is the preferred provider.

1. In the recipe, there should be:

   ```
   PROVIDES += "u-boot-fw-utils"
   RPROVIDES_${PN} += "u-boot-fw-utils"
   ```

2. And in the machine section of the board:

   ```
   PREFERRED_PROVIDER_u-boot-fw-utils = "u-boot-fw-utils-my-fork"
   ```

3. And finally, `u-boot-mender.inc` needs to included using `require`, as for
   `u-boot`.
