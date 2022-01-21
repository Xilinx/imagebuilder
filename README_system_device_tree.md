# ImageBuilder with System Device Tree

Please refer to README.md to know more about ImageBuilder

ImageBuilder can be used with System Device Tree to generate the
scripts, binaries, etc required to boot Xen, Dom0 and DomU kernels on
a hardware platform.
The System Device Tree contains the hardware information as well as
domain information.
Refer https://gitenterprise.xilinx.com/stefanos/system-device-tree/blob/master/system-device-tree.dts
for the system device tree example.

## scripts/lop-xen.dts

ImageBuilder derives its configuration from "domains" node in System
Device Tree, e.g.:

xen: domain@2 {
    compatible = "openamp,domain-v1","openamp,hypervisor-v1";
    cpus = <&cpus_a72 0x3 0x00000002>;
    memory = <0x0 0x500000 0x0 0x7fb00000>;

    dom0: domain@3 {
        compatible = "openamp,domain-v1","xen,domain-v2";
        cpus = <&cpus_a72 0x3 0x00000001>;
        memory = <0x0 0x501000 0x0 0x3faff000>;
    };

    linux1: domain@4 {
        compatible = "openamp,domain-v1","xen,domain-v2";
        cpus = <&cpus_a72 0x3 0x00000001>;
        memory = <0x0 0x501000 0x0 0x3faff000>;
        access = <&mmc0 0x0>;
    };

    linux2: domain@5 {
        compatible = "openamp,domain-v1","xen,domain-v2";
        cpus = <&cpus_a72 0x3 0x00000001>;
        memory = <0x0 0x40000000 0x0 0x40000000>;

        firewallconfig = <&linux1 1 0>;
    };
};

From the above snippet, ImageBuilder can deduce the number of domUs,
count of virtual CPUs for Dom0, amount of memory for Dom0, count of
virtual CPUs for each DomU, amount of memory for each DomU and device
assigned to each  DomU.

lop-xen.dts is the lopper dts file to generate the ImageBuilder config
file (containing the above information) from System Device Tree.

To use it, one needs to checkout the following two repositories:

```
git clone git://github.com/devicetree-org/lopper
git clone https://gitenterprise.xilinx.com/stefanos/system-device-tree.git
```

Note: One needs to add a 'dom0' label (as shown in the previous snippet)
for the node which represents dom0.

And then run the following command from the "lopper" directory:

```
python3 lopper.py  -f --enhanced -i <path_to>/imagebuilder/scripts/lop-xen.dts \
        <path_to>/system-device-tree/system-device-tree-xen.dts > config
```

For the dts snippet shown before, the following 'config' file will be
generated:

NUM_DOMUS=2
DOM0_VCPUS = 2
DOM0_MEM = 1018
DOMU_VCPUS[0] = 2
DOMU_MEM[0] = 1018
DOMU_PASSTHROUGH_PATHS[0] = "/bus@f1000000/sdhci@f1050000"
DOMU_VCPUS[1] = 2
DOMU_MEM[1] = 1024

Refer to README.md for a description of each of these options.
