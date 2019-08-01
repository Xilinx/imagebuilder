# imagebuilder

## `Dockerfile.image`

```
$ cd imagebuilder

$ docker build --force-rm --file Dockerfile.image -t imagebuilder .
```

## `/imagebuilder_sd` or `/imagebuilder_tftp`

### TFTP

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

### SATA

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
