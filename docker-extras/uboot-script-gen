#!/bin/bash

offset=$((2*1024*1024))
filesize=0

function add_device_tree_kernel()
{
    local path=$1
    local addr=$2
    local size=$3

    echo "fdt mknod $path module$addr" >> $UBOOT_SOURCE
    echo "fdt set $path/module$addr compatible \"multiboot,kernel\" \"multiboot,module\"" >> $UBOOT_SOURCE
    echo "fdt set $path/module$addr reg <0x0 "$addr" 0x0 "$size">" >> $UBOOT_SOURCE
    echo "fdt set $path/module$addr bootargs \"console=ttyAMA0\"" >> $UBOOT_SOURCE
}

function add_device_tree_ramdisk()
{
    local path=$1
    local addr=$2
    local size=$3

    echo "fdt mknod $path module$addr" >> $UBOOT_SOURCE
    echo "fdt set $path/module$addr compatible \"multiboot,ramdisk\" \"multiboot,module\"" >> $UBOOT_SOURCE
    echo "fdt set $path/module$addr reg <0x0 "$addr" 0x0 "$size">" >> $UBOOT_SOURCE
}

function add_device_tree_passthrough()
{
    local path=$1
    local addr=$2
    local size=$3

    echo "fdt mknod $path module$addr" >> $UBOOT_SOURCE
    echo "fdt set $path/module$addr compatible \"multiboot,device-tree\" \"multiboot,module\"" >> $UBOOT_SOURCE
    echo "fdt set $path/module$addr reg <0x0 "$addr" 0x0 "$size">" >> $UBOOT_SOURCE
}

function device_tree_editing()
{
    local device_tree_addr=$1

    echo "fdt addr $device_tree_addr" >> $UBOOT_SOURCE
    echo "fdt resize 1024" >> $UBOOT_SOURCE
    echo "fdt set /chosen \#address-cells <0x2>" >> $UBOOT_SOURCE
    echo "fdt set /chosen \#size-cells <0x2>" >> $UBOOT_SOURCE
    echo "fdt set /chosen xen,xen-bootargs \"console=dtuart dtuart=serial0 dom0_mem=700M dom0_max_vcpus=1 bootscrub=0 serrors=forward vwfi=native sched=null\"" >> $UBOOT_SOURCE
    echo "fdt mknod /chosen dom0" >> $UBOOT_SOURCE
    echo "fdt set /chosen/dom0 compatible \"xen,linux-zimage\" \"xen,multiboot-module\"" >> $UBOOT_SOURCE
    echo "fdt set /chosen/dom0 reg <0x0 "$dom0_kernel_addr" 0x0 "$dom0_kernel_size">" >> $UBOOT_SOURCE
    if test "$LOAD_CMD" = "tftpb"
    then
        echo "fdt set /chosen/dom0 bootargs \"console=hvc0 earlycon=xen earlyprintk=xen root=/dev/ram0\"" >> $UBOOT_SOURCE
        if test $dom0_ramdisk_addr != "-"
        then
            echo "fdt mknod /chosen dom0-ramdisk" >> $UBOOT_SOURCE
            echo "fdt set /chosen/dom0-ramdisk compatible \"xen,linux-initrd\" \"xen,multiboot-module\"" >> $UBOOT_SOURCE
            echo "fdt set /chosen/dom0-ramdisk reg <0x0 "$dom0_ramdisk_addr" 0x0 "$dom0_ramdisk_size">" >> $UBOOT_SOURCE
        fi
    else
        echo "fdt set /chosen/dom0 bootargs \"console=hvc0 earlycon=xen earlyprintk=xen root=/dev/sda2\"" >> $UBOOT_SOURCE
    fi

    i=0
    while test $i -lt $NUM_DOMUS
    do
        echo "fdt mknod /chosen domU$i" >> $UBOOT_SOURCE
        echo "fdt set /chosen/domU$i compatible \"xen,domain\"" >> $UBOOT_SOURCE
        echo "fdt set /chosen/domU$i \#address-cells <0x2>" >> $UBOOT_SOURCE
        echo "fdt set /chosen/domU$i \#size-cells <0x2>" >> $UBOOT_SOURCE
        echo "fdt set /chosen/domU$i memory <0x0 0x40000>" >> $UBOOT_SOURCE
        echo "fdt set /chosen/domU$i cpus <0x1>" >> $UBOOT_SOURCE
        echo "fdt set /chosen/domU$i vpl011 <0x1>" >> $UBOOT_SOURCE
        add_device_tree_kernel "/chosen/domU$i" ${domU_kernel_addr[$i]} ${domU_kernel_size[$i]}
        if test "${domU_ramdisk_addr[$i]}"
        then
            add_device_tree_ramdisk "/chosen/domU$i" ${domU_ramdisk_addr[$i]} ${domU_ramdisk_size[$i]}
        fi
        if test "${domU_passthrough_dtb_addr[$i]}"
        then
            add_device_tree_passthrough "/chosen/domU$i" ${domU_passthrough_dtb_addr[$i]} ${domU_passthrough_dtb_size[$i]}
        fi
        i=$(( $i + 1 ))
    done
}

function add_size()
{
    local filename=$1
    local size=`stat --printf="%s" $filename`
    memaddr=$(( $memaddr + $size + $offset - 1))
    memaddr=$(( $memaddr & ~($offset - 1) ))
    memaddr=`printf "0x%X\n" $memaddr`
    filesize=$size
}

function load_file()
{
    local filename=$1

    echo "$LOAD_CMD $memaddr $filename" >> $UBOOT_SOURCE
    add_size $filename
}

function check_file_type()
{
    local filename=$1
    local type="$2"

    file $filename | grep "$type" &> /dev/null
    if test $? != 0
    then
        echo Wrong file type "$filename". It shold be "$type".
    fi
}

function check_compressed_file_type()
{
    local filename=$1
    local type="$2"

    file $filename | grep "gzip compressed data" &> /dev/null
    if test $? == 0
    then
        local tmp=`mktemp`
        cat $filename | gunzip > $tmp
        filename=$tmp
    fi
    check_file_type $filename "$type"
}

. config

rm -f $UBOOT_SOURCE $UBOOT_SCRIPT
memaddr=$(( $MEMORY_START + $offset ))
memaddr=`printf "0x%X\n" $memaddr`
uboot_addr=$memaddr
# 2MB are enough for a uboot script
memaddr=$(( $memaddr + $offset ))
memaddr=`printf "0x%X\n" $memaddr`

check_compressed_file_type $XEN "MS-DOS executable"
xen_addr=$memaddr
load_file "$XEN"

check_compressed_file_type $DOM0_KERNEL "MS-DOS executable"
dom0_kernel_addr=$memaddr
load_file $DOM0_KERNEL
dom0_kernel_size=$filesize

if test "$DOM0_RAMDISK" && [[ $LOAD_CMD = "tftpb" ]]
then
    check_compressed_file_type $DOM0_RAMDISK "cpio archive"
    dom0_ramdisk_addr=$memaddr
    load_file "$DOM0_RAMDISK"
    dom0_ramdisk_size=$filesize
else
    dom0_ramdisk_addr="-"
fi

i=0
while test $i -lt $NUM_DOMUS
do
    check_compressed_file_type ${DOMU_KERNEL[$i]} "MS-DOS executable"
    domU_kernel_addr[$i]=$memaddr
    load_file ${DOMU_KERNEL[$i]}
    domU_kernel_size[$i]=$filesize
    if test "${DOMU_RAMDISK[$i]}"
    then
        check_compressed_file_type ${DOMU_RAMDISK[$i]} "cpio archive"
        domU_ramdisk_addr[$i]=$memaddr
        load_file ${DOMU_RAMDISK[$i]}
        domU_ramdisk_size[$i]=$filesize
    fi
    if test "${DOMU_PASSTHROUGH_DTB[$i]}"
    then
        check_compressed_file_type ${DOMU_PASSTHROUGH_DTB[$i]} "Device Tree Blob"
        domU_passthrough_dtb_addr[$i]=$memaddr
        load_file ${DOMU_PASSTHROUGH_DTB[$i]}
        domU_passthrough_dtb_size[$i]=$filesize
    fi
    i=$(( $i + 1 ))
done

check_file_type $DEVICE_TREE "Device Tree Blob"
device_tree_addr=$memaddr
load_file $DEVICE_TREE
device_tree_editing $device_tree_addr

# disable device tree reloation
echo "setenv fdt_high 0xffffffffffffffff" >> $UBOOT_SOURCE
echo "booti $xen_addr - $device_tree_addr" >> $UBOOT_SOURCE
mkimage -A arm64 -T script -C none -a $uboot_addr -e $uboot_addr -d $UBOOT_SOURCE "$UBOOT_SCRIPT" &> /dev/null

memaddr=$(( $MEMORY_END - $memaddr - $offset ))
if test $memaddr -lt 0
then
    echo Error, not enough memory to load all binaries
    exit 1
fi

echo "Generated uboot script $UBOOT_SCRIPT, to be loaded at address $uboot_addr:"
echo "$LOAD_CMD $uboot_addr $UBOOT_SCRIPT; source $uboot_addr"
