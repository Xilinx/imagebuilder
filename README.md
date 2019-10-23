# ImageBuilder

Building an embedded virtualized system with anything more than one
Domain can be difficult, error prone and time consuming.

ImageBuilder, an Open Source collection of scripts (contributions
encouraged), changes all that.

ImageBuilder generates a U-Boot script that can be used to load all of
the binaries automatically and boot the full system fast. Given a
collection of binaries such as Xen, Dom0 and a number of Dom0-less
DomUs, ImageBuilder takes care of calculating all loading addresses,
editing device tree with the necessary information, and even
pre-configuring a disk image with kernels and rootfses. ImageBuilder
scripts can be used stand-alone, or from an ImageBuilder Container.

ImageBuilder has been tested on Xilinx ZynqMP MPSoC boards. An
up-to-date wikipage is also available at
[wiki.xenproject.org](https://wiki.xenproject.org/index.php?title=ImageBuilder).


## Stand-alone Usage: scripts/uboot-script-gen

The ImageBuilder script that generates a u-boot script to load all your
binaries for a Xen Dom0-less setup is `scripts/uboot-script-gen`.

To use it, first write a config file like `config`:

```
MEMORY_START="0x0"
MEMORY_END="0x80000000"

DEVICE_TREE="mpsoc.dtb"
XEN="xen"
DOM0_KERNEL="Image-dom0"
DOM0_RAMDISK="dom0-ramdisk.cpio"

NUM_DOMUS=2
DOMU_KERNEL[0]="zynqmp-dom1/Image-domU"
DOMU_RAMDISK[0]="zynqmp-dom1/domU-ramdisk.cpio"
DOMU_PASSTHROUGH_DTB[0]="zynqmp-dom1/passthrough-example-part.dtb"
DOMU_KERNEL[1]="zynqmp-dom2/Image-domU"
DOMU_RAMDISK[1]="zynqmp-dom2/domU-ramdisk.cpio"

UBOOT_SOURCE="boot.source"
UBOOT_SCRIPT="boot.scr"
```

Where:
- MEMORY_START and MEMORY_END specify the start and end of RAM.

- DEVICE_TREE specifies the DTB file to load.

- XEN specifies the Xen hypervisor binary to load. Note that it has to
  be a regular Xen binary, not a u-boot binary.

- DOM0_KERNEL specifies the Dom0 kernel file to load.

- DOM0_RAMDISK specifies the Dom0 ramdisk to use. Note that it should be
  a regular ramdisk cpio.gz file, not a u-boot binary.

- NUM_DOMUS specifies how many Dom0-less DomUs to load

- DOMU_KERNEL[number] specifies the DomU kernel to use.

- DOMU_RAMDISK[number] specifies the DomU ramdisk to use.

- DOMU_PASSTHROUGH_DTB[number] specifies the device assignment
  configuration, see xen.git:docs/misc/arm/passthrough.txt

Then you can invoke uboot-script-gen as follows:

```
$ bash ./scripts/uboot-script-gen -c /path/to/config-file -d . -t tftp
```

Where:
-c specifies the path to the config file to use
-d specifies the working directory (path in the config file are relative
   to it)
-t specifies the u-boot command to load the binaries. "tftp" and "sd"
   are shorthands for "tftpb" and "load scsi 0:1", but actually any
   arbitrary command can be used, for instance -t "fatload" is valid.


## Container Usage

ImageBuilder comes with a Dockerfile to build a container with all the
required scripts. The following chapters explain how to build it and use
it.


### `Dockerfile.image`

Build the container.

```
$ cd imagebuilder

$ docker build --force-rm --file Dockerfile.image -t imagebuilder .
```

### `/imagebuilder_sd` or `/imagebuilder_tftp`

#### TFTP

Run the following command to generate a uboot script to tftp all the
necessary binaries automatically:

```
$ docker run --rm --privileged=true -ti -v /tmp:/tmp -v /dev:/dev imagebuilder /imagebuilder_tftp
```

The generated files are in `/tmp/output`. At the u-boot prompt run:

```
tftpb 0xc00000 boot.scr; source 0xc00000
```

If you are using QEMU, you also need to manually setup the ip address.
Run this command instead:

```
setenv serverip 192.168.76.2; tftpb 0xc00000 boot.scr; source 0xc00000
```

#### SATA

Run the following command to generate an image to be written on disk:

```
$ docker run --rm --privileged=true -ti -v /tmp:/tmp -v /dev:/dev imagebuilder /imagebuilder_sd
```

The generated image is `/tmp/img/zynqmp.img`. Proceed to dd it to a
disk, or pass the file as an argument to QEMU (describing how to use
QEMU to emulate a SATA disk is out of scope for this document).

At the u-boot prompt you can boot automatically with the following
command:

```
scsi scan; load scsi 0:1 0xc00000 boot.scr; source 0xc00000
```

### Add additional DomUs

Assuming that you have the kernel and ramdisk of another DomU already in
`PACKAGE.md` format, you can configure Imagebuilder to start it
automatically at boot by making the following changes:

- edit `Dockerfile.image`, add the domU package to the FROM lines
- edit `Dockerfile.image`, add the domU package to the COPY lines
- edit `config`, adding another DomU (NUM_DOMUS, DOMU_KERNEL and DOMU_RAMDISK)

Rebuild Imagebuilder and rerun imagebuilder_sd/tftp.
