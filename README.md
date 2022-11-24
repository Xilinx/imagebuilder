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
pre-configuring a disk image with kernels and rootfses.

ImageBuilder has been tested on Xilinx ZynqMP MPSoC boards. An
up-to-date wikipage is also available at
[wiki.xenproject.org](https://wiki.xenproject.org/index.php?title=ImageBuilder).


## scripts/uboot-script-gen

The ImageBuilder script that generates a u-boot script to load all your
binaries for a Xen Dom0-less setup is `scripts/uboot-script-gen`.

To use it, first write a config file like `config`:

```
MEMORY_START="0x0"
MEMORY_END="0x80000000"
LOAD_CMD="tftpb"
BOOT_CMD="booti"

DEVICE_TREE="mpsoc.dtb"
XEN="xen"
XEN_CMD="console=dtuart dtuart=serial0 dom0_mem=1G dom0_max_vcpus=1 bootscrub=0 vwfi=native sched=null"
PASSTHROUGH_DTS_REPO="git@github.com:Xilinx/xen-passthrough-device-trees.git device-trees-2021.2/zcu102"
DOM0_KERNEL="Image-dom0"
DOM0_CMD="console=hvc0 earlycon=xen earlyprintk=xen clk_ignore_unused"
DOM0_RAMDISK="dom0-ramdisk.cpio"
DOM0_MEM=1024
DOM0_VCPUS=1

NUM_DT_OVERLAY=1
DT_OVERLAY[0]="host_dt_overlay.dtbo"

NUM_DOMUS=2
DOMU_KERNEL[0]="zynqmp-dom1/Image-domU"
DOMU_PASSTHROUGH_PATHS[0]="/axi/ethernet@ff0e0000 /axi/serial@ff000000"
DOMU_CMD[0]="console=ttyPS0 earlycon console=ttyPS0,115200 clk_ignore_unused rdinit=/sbin/init root=/dev/ram0 init=/bin/sh"
DOMU_RAMDISK[0]="zynqmp-dom1/domU-ramdisk.cpio"
DOMU_COLORS[0]="6-14"
DOMU_KERNEL[1]="zynqmp-dom2/Image-domU"
DOMU_CMD[1]="console=ttyAMA0 clk_ignore_unused rdinit=/sbin/init root=/dev/ram0 init=/bin/sh"
DOMU_RAMDISK[1]="zynqmp-dom2/domU-ramdisk.cpio"
DOMU_MEM[1]=512
DOMU_VCPUS[1]=1

BITSTREAM=download.bit

NUM_BOOT_AUX_FILE=2
BOOT_AUX_FILE[0]="BOOT.BIN"
BOOT_AUX_FILE[1]="uboot.cfg"

UBOOT_SOURCE="boot.source"
UBOOT_SCRIPT="boot.scr"
APPEND_EXTRA_CMDS="extra.txt"
FDTEDIT="imagebuilder.dtb"
FIT="boot.fit"
FIT_ENC_KEY_DIR="dir/key"
FIT_ENC_UB_DTB="uboot.dtb"
```

Where:
- MEMORY_START and MEMORY_END specify the start and end of RAM.

- LOAD_CMD specifies the u-boot command used to load the binaries. This
  can be left out of the config and be (over)written by the -t CLI
  argument. It has to be set either in the config file or CLI argument
  though.

- BOOT_CMD specifies the u-boot command used to boot the binaries.
  By default, it is 'booti'. The acceptable values are 'booti', 'bootm'
  and 'bootefi' and 'none'. If the value is 'none', the BOOT_CMD is not
  added to the boot script, and the addresses for the Xen binary and the
  DTB are stored in 'host_kernel_addr' and 'host_fdt_addr' u-boot
  env variables respectively, to be used manually when booting.

- DEVICE_TREE specifies the DTB file to load.

- XEN specifies the Xen hypervisor binary to load. Note that it has to
  be a regular Xen binary, not a u-boot binary.

- XEN_COLORS specifies the colors (cache coloring) to be used for Xen
  and is in the format startcolor-endcolor

- XEN_CMD specifies the command line arguments used for Xen.  If not
  set, the default one will be used.

- PASSTHROUGH_DTS_REPO specifies the git repository and/or the directory
  which contains the partial device trees. This is optional. However, if
  this is specified, then DOMU_PASSTHROUGH_PATHS[number] need to be specified.
  uboot-script-gen will compile the partial device trees which have
  been specified in DOMU_PASSTHROUGH_PATHS[number].

- DOM0_KERNEL specifies the Dom0 kernel file to load.
  For dom0less configurations, the parameter is optional.

- DOM0_MEM specifies the amount of memory for Dom0 VM in MB. The default
  is 1024. This is only applicable when XEN_CMD is not specified.

- DOM0_VCPUS specifies the number of VCPUs for Dom0. The default is 1. This is
  only applicable when XEN_CMD is not specified.

- DOM0_COLORS specifies the colors (cache coloring) to be used for dom0
  and is in the format startcolor-endcolor

- DOM0_CMD specifies the command line arguments for Dom0's Linux
  kernel.  If "root=" isn't set, imagebuilder will try to determine it.
  If not set at all, the default one is used.

- DOM0_RAMDISK specifies the Dom0 ramdisk to use. Note that it should be
  a regular ramdisk cpio.gz file, not a u-boot binary.

- NUM_DT_OVERLAY specifies the number of host device tree overlays to be
  added at boot time in u-boot

- DT_OVERLAY[number] specifies the path to the hosts device tree overlays
  to be added at boot time in u-boot

- LOPPER_PATH specifies the path to lopper.py script, the main script in the
  Lopper repository (https://github.com/devicetree-org/lopper). This is
  optional. However, if this is specified, then DOMU_PASSTHROUGH_PATHS[number]
  needs to be specified. uboot-script-gen will invoke lopper to generate the
  partial device trees for devices which have been listed in
  DOMU_PASSTHROUGH_PATHS[number]. This option is currently in experimental state
  as the corresponding lopper changes are still in an early support state.

- LOPPER_CMD specifies the command line arguments for lopper's extract assist.
  This is optional and only applicable when LOPPER_PATH is specified. Only to be
  used to specify which nodes to include (using -i <node_name>) and which
  nodes/properties to exclude (using -x <regex>). If not set at all, the default
  one is used applicable for ZynqMP MPSoC boards.

- NUM_DOMUS specifies how many Dom0-less DomUs to load

- DOMU_KERNEL[number] specifies the DomU kernel to use.

- DOMU_CMD[number] specifies the command line arguments for DomU's Linux
  kernel. If not set, then "console=ttyAMA0" is used.

- DOMU_RAMDISK[number] specifies the DomU ramdisk to use.

- DOMU_PASSTHROUGH_PATHS[number] specifies the passthrough devices (
  separated by spaces). It adds "xen,passthrough" to the corresponding
  dtb nodes in xen device tree blob.
  This option is valid in the following cases:

  1. When PASSTHROUGH_DTS_REPO is provided.
  With this option, the partial device trees (corresponding to the
  passthrough devices) from the PASSTHROUGH_DTS_REPO, are compiled
  merged and used as DOMU[number] device tree blob.
  Note it assumes that the names of the partial device trees will match
  to the names of the devices specified here.

  2. When LOPPER_PATH is provided.
  With this option, the partial device trees (corresponding to the
  passthrough devices) are generated by the lopper and then compiled and merged
  by ImageBuilder to be used as DOMU[number] device tree blob.

  3. When DOMU_NOBOOT[number] is provided. In this case, it will only
  add "xen,passthrough" as mentioned before.

- DOMU_PASSTHROUGH_DTB[number] specifies the passthrough device trees
  blob. This option is used when DOMU_PASSTHROUGH_PATHS[number] is not
  specified by the user.
  NOTE that with this option, user needs to manually set xen,passthrough
  in xen.dtb.

- DOMU_MEM[number] is the amount of memory for the VM in MB, default 512MB

- DOMU_VCPUS[number] is the number of vcpus for the VM, default 1

- DOMU_COLORS[number] specifies the colors (cache coloring) to be used
  for the domain and is in the format startcolor-endcolor

- DOMU_NOBOOT[number]: if specified, the DomU is not started
  automatically at boot as dom0-less guest. It can still be created
  later from Dom0.

- DOMU_STATIC_MEM[number]="baseaddr1 size1 ... baseaddrN sizeN"
  if specified, indicates the host physical address regions
  [baseaddr, baseaddr + size) to be reserved to the VM for static allocation.

- DOMU_DIRECT_MAP[number] can be set to 1 or 0.
  If set to 1, the VM is direct mapped. The default is 1.
  This is only applicable when DOMU_STATIC_MEM is specified.

- DOMU_ENHANCED[number] can be set to 1 or 0, default is 1 when Dom0 is
  present. If set to 1, the VM can use PV drivers. Older Linux kernels
  might break.

- DOMU_CPUPOOL[number] specifies the id of the cpupool (created using
  CPUPOOL[number] option, where number == id) that will be assigned to domU.

- DOMU_DRIVER_DOMAIN[number] if set to 1 the domain is a driver domain.
  Set driver_domain in xl config file. This option is only available for
  the disk_image script.

- LINUX is optional but specifies the Linux kernel for when Xen is NOT
  used.  To enable this set any LINUX\_\* variables and do NOT set the
  XEN variable.

- LINUX_CMD is optional but specifies the baremetal Linux kernel CMD.

- LINUX_RAMDISK is optional but specifies the ramdisk to use when
   booting baremetal Linux.

- BITSTREAM is optional but specifies the bitstream to program the FPGA
  with in u-boot when booting.  Currently only a single bitstream is
  supported.

- NUM_BOOT_AUX_FILE: is optional but if specified tell how many extra
  files to include in the first partition with disk_image.   Useful for
  things like uboot config files, firmware files or other such files.

- BOOT_AUX_FILE[number]: a list of file(s) to be included in the first
  partition.

- UBOOT_SOURCE and UBOOT_SCRIPT specify the output. They are optional
  as you can pass -o FILENAME to uboot-script-gen as a command line
  parameter

- APPEND_EXTRA_CMDS: is optional and specifies the path to a text file
  containing extra u-boot commands to be added to the boot script before
  the boot command. Useful for running custom fixup commands.

- FDTEDIT is an optional and is off by default.  Specifies the output
  modified dtb, used for reference and fdt_std.

- FIT is an optional and is off by default.  Specifies using a fit image
  for booting rather than individual files.

- FIT_ENC_KEY_DIR is optional but specifies the directory and hint used
  for signing a FIT image.  The CLI arguments overwrite what's in the
  config.  See the -u option below for more information.

- FIT_ENC_UB_DTB is optional but specifies the u-boot dtb to modify and
  include the public key in.  This can only be used with
  FIT_ENC_KEY_DIR.  See the -u option below for more information.

- CPUPOOL[number]="cpu@1,...,cpu@N scheduler"
  specifies the list of cpus' node names (separated by commas) and the scheduler
  to be used to create boot-time cpupool. If no scheduler is set, the Xen
  default one will be used.

- NUM_CPUPOOLS specifies the number of boot-time cpupools to create.

Then you can invoke uboot-script-gen as follows:

```
$ bash ./scripts/uboot-script-gen -c /path/to/config-file -d . -t tftp -o bootscript
```

Where:\
-c specifies the path to the config file to use\
-d specifies the "root" directory (paths in the config file are relative
   to it), this is not a working directory (any output file locations
   are specified in the config and any temporary files are in /tmp)\
-t specifies the u-boot command to load the binaries. "tftp", "sd", "usb"
   and "scsi" are shorthands for "tftpb", "load mmc 0:1", "fatload usb 0:1"
   and "load scsi 0:1", but actually any arbitrary command can be used,
   for instance -t "fatload" is valid.  The only special commands are:
   fit, which generates a FIT image using a script, and fit_std, which
   produces a standard style of fit image without a script, but has
   issues with dom0less configurations and isn't recommended. \
-o specifies the output filename for the uboot script and its source.\
-k specifies the key directory for signing images in a FIT image and the
   hint.  The hint is the name of the crt and key files minus the
   suffix (<hint>.key, <hint>.crt).  This is optional and but enables
   signature for the fit or fit_std -t options.\
-u specifies the U-boot control dtb.  This is an optional argument but
   can only be used  in combination with the -k option.  This adds the
   public key into the dtb.  Then one can add this dtb back into the
   u-boot bin or elf.\

### Signed FIT images

Signed FIT images are a way to sign images with asymmetrical keys. While 
making the FIT image, images are signed with a private key; then during
boot U-Boot uses a public key in its control dtb to verify the 
signatures. Some of the U-Boot config options needed are:
CONFIG_FIT_SIGNATURE=y\
CONFIG_RSA=y\
CONFIG_LEGACY_IMAGE_FORMAT=n\

Once U-boot is built, then take the control dtb, supply it to
Imagebuilder when building a signed image, then use it when booting.
For generating the keys and other documentation, see:\
u-boot/doc/uImage.FIT/signature.txt\


## scripts/disk\_image

The ImageBuilder script that generates a disk image file to load on a SD
or SATA drive.  This creates multiple partitions: 1 boot partition where
the boot files from working directory (-c option) are, and 1 additional
partition for each Dom0/DomU cpio archive to write to disk.

disk\_image will write to disk as separate partition each file specified
as follows:

- DOM0_ROOTFS specifies the Dom0 rootfs to use.

- DOMU_ROOTFS[number] specifies the DomU rootfs to use.

The provided rootfs should not be a u-boot binary. The supported
formats are:

- cpio.gz
- tar.gz

After you've generated the u-boot scripts using the uboot-script-gen
script, disk_image is run as follows:

```
$ sudo bash ./scripts/disk_image -c /path/to/config-file -d . \
                                 -w /path/to/tmp/dir          \
                                 -o /path/to/output/disk.img  \
                                 -t "sd"
```

Where:\
-c specifies the path to the config file to use\
-d specifies the working directory (paths in the config file are relative
   to it)\
-w specifies the temporary working directory that the script uses for
   building the disk image, and if not set, one is created in /tmp\
-o specifies the output disk image file name\
-a specifies whether the disk image size is to be aligned to the nearest
   power of two\
-t specifies the u-boot command to load the binaries. "tftp", "sd", "usb"
   and "scsi" are shorthands for "tftpb", "load mmc 0:1", "fatload usb 0:1"
   and "load scsi 0:1", but actually any arbitrary command can be used,
   for instance -t "fatload" is valid.


disk\_image supports these additional parameters on the config file:

- GPT is optional and select the usage of a GPT partition table (sgdisk
  is required)


disk_image also generates on the fly a xl config file for each domU and
adds them to the dom0 rootfs partition under /etc/xen. It makes it
easier to start those domUs from dom0.
