# A container package

Imagebuilder takes multiple `package` containers as input and produces a
runnable u-boot script or a runnable disk image as output. This document
describes the package format.

A package is a container containing all the relevant binaries for a
given component under the following path:

```
/home/builder/output-<name>/
```

For instance, for DomUs a package containes the kernel, rootfs,
additional configuration. As an example, zynqmp-dom1-package contains
the following files:

```
/home/builder/output-zynqmp-dom1/Image-domU
/home/builder/output-zynqmp-dom1/domU-ramdisk.cpio
/home/builder/output-zynqmp-dom1/passthrough-example-part.dtb
```

Which are the kernel, ramdisk, passthrough configuration respectively.
