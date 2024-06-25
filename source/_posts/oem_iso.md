---
title: UOS镜像定制脚本
date: 2020-11-27 09:45:03
tags:
---

# 注意！！！

如果有内核deb包，定制内核名，基本镜像中的内核包名，宿主机内核包名要一致，否则会定制失败

```bash
#!/bin/bash
#set -x

#基本镜像，指需要被定制的镜像
#定制镜像，指被定制加工完成的镜像
#[需要设置的部分]
#基本镜像的下载链接
ISO_SOURCE_URL=https://cdimage.uniontech.com/iso-v20/uniontechos-desktop-20-professional-1022_arm64.iso

#基本镜像存放文件夹的路径
ISO_SOURCE_PATH=cdimage

#基本镜像的文件名
ISO_SOURCE_NAME=uniontechos-desktop-20-professional-1022_arm64.iso

#定制镜像要保存的文件夹路径
ISO_DEST_PATH=cdimage

#定制镜像要保存的文件名
ISO_DEST_NAME=uniontechos-desktop-20-professional-1022_arm64-oem.iso

#定制的deb包路径
DEBS_PATH=debs

#定制的deb包(内核)路径，如果没有内核包请留空
DEBS_KERNEL_PATH=debs_kernel

#定制的deb包(内核) release 和 version，没有的话留空
# 例如，下行的 release 为 4.19.0-arm64-desktop，version 为 4.19.90-2001
# linux-image-4.19.0-arm64-desktop_4.19.90-2001_arm64.deb
# 因为宿主机的内核release必须和要定制的镜像的内核release一致，所以这里直接取宿主机内核release
DEBS_KERNEL_RELEASE=`uname -r`
DEBS_KERNEL_VERSION=4.19.90-2001

#[不需要设置的部分]
#临时文件夹:ISO解压后存放的路径
TEMP_ISO=iso

#临时文件夹:filesystem.squashfs解压后存放的路径
TEMP_ROOTFS=rootfs

USER=$(env | grep ^USER | cut -d "=" -f 2)
if [ $USER != "root" ];then
    echo "请用root权限执行此脚本!"
    exit
fi

if [ $# -eq 0 ];then
echo "用法:
    1)$0 config 配置环境
    2)将内核deb包放入 $DEBS_KERNEL_PATH 中去
    3)将其它deb包放入 $DEBS_PATH 中去
    4)$0 build 定制镜像"
    exit
fi

#[只运行一次的命令]
if [ $# -eq 1 ] && [ $1 == "config" ];then
    echo "====开始配置OEM环境"

    #安装定制oem所需的依赖软件
    echo "====安装定制oem所需的依赖软件"
    apt install -y squashfs-tools dpkg-dev xorriso wget
    
    #准备各个文件夹
    mkdir -m 777 -pv $ISO_SOURCE_PATH $ISO_DEST_PATH $DEBS_PATH $DEBS_KERNEL_PATH
    echo "文件夹创建成功"
    echo "请把内核deb包放在$DEBS_KERNEL_PATH中"
    echo "请把其它deb包放在$DEBS_PATH"

    #下载基本镜像
    echo "====下载基本镜像$ISO_SOURCE_NAME到$ISO_SOURCE_PATH目录中去"
    if [ ! -e $ISO_SOURCE_PATH/$ISO_SOURCE_NAME ];then
        wget --no-check-certificate $ISO_SOURCE_URL -O $ISO_SOURCE_PATH/$ISO_SOURCE_NAME
    fi
    
    exit
fi

if [ $# -eq 1 ] && [ $1 == "build" ];then
    echo "====开始制作OEM镜像"

    # 内核deb包文件有四个:image和headers为必须安装，libc为推荐安装，dbg为可选安装
    if [ -d $DEBS_KERNEL_PATH ];then
        if [ -z $DEBS_KERNEL_RELEASE ] || [ -z $DEBS_KERNEL_VERSION ];then
            echo "$DEBS_KERNEL_RELEASE 或者 $DEBS_KERNEL_VERSION 配置有误"
            exit
        fi

        DEBS_LINUX_IMAGE=linux-image-${DEBS_KERNEL_RELEASE}_${DEBS_KERNEL_VERSION}_arm64.deb
        DEBS_LINUX_HEADERS=linux-headers-${DEBS_KERNEL_RELEASE}_${DEBS_KERNEL_VERSION}_arm64.deb
        DEBS_LINUX_LIBC=linux-libc-dev_${DEBS_KERNEL_VERSION}_arm64.deb
        DEBS_LINUX_IMAGE_DBG=linux-image-${DEBS_KERNEL_RELEASE}-dbg_${DEBS_KERNEL_VERSION}_arm64.deb
        
        if [ ! -e $DEBS_KERNEL_PATH/$DEBS_LINUX_IMAGE ] || [ ! -e $DEBS_KERNEL_PATH/$DEBS_LINUX_HEADERS ];then
            echo "内核deb包文件有四个:image和headers为必须安装，libc为推荐安装，dbg为可选安装"
            echo "如果你不想安装内核deb包，请删除 $DEBS_KERNEL_PATH 文件夹"
            exit
        fi
    fi

    #卸载dev和proc文件系统
    echo "====卸载dev和proc文件系统"
    umount $TEMP_ROOTFS/proc
    umount $TEMP_ROOTFS/sys
    umount $TEMP_ROOTFS/dev/pts
    umount $TEMP_ROOTFS/dev
    umount $TEMP_ROOTFS/mnt
    
    #拷贝iso中所有文件到iso目录中
    echo "====解压ISO文件到临时目录"
    umount /mnt
    mount $ISO_SOURCE_PATH/$ISO_SOURCE_NAME /mnt
    rm -rf $TEMP_ISO
    mkdir -pv $TEMP_ISO
    cp -r /mnt/. $TEMP_ISO
    rm -rf $TEMP_ROOTFS
    mkdir -pv $TEMP_ROOTFS
    
    #解压filesystem.squashfs到fs目录
    echo "====解压filesystem.squashfs文件"
    unsquashfs -f -d $TEMP_ROOTFS /mnt/live/filesystem.squashfs
    
    #挂载本机/dev和/proc文件系统到rootfs目录下
    echo "====挂载本机/dev和/proc文件系统到rootfs目录下"
    mount --bind /proc/ $TEMP_ROOTFS/proc/
    mount --bind /sys/ $TEMP_ROOTFS/sys/
    mount --bind /dev/ $TEMP_ROOTFS/dev/
    mount --bind /dev/pts $TEMP_ROOTFS/dev/pts
    
    #chroot到$TEMP_ROOTFS系统并安装相关的deb(内核)包,安装完后退出rootfs系统
    echo "====安装内核deb包"
    if [ -n $DEBS_KERNEL_PATH ];then
        umount $TEMP_ROOTFS/mnt/
        mount --bind $DEBS_KERNEL_PATH/ $TEMP_ROOTFS/mnt/

        #安装 linux-image
        if [ -e $DEBS_KERNEL_PATH/$DEBS_LINUX_IMAGE ];then
                chroot $TEMP_ROOTFS /bin/sh -c "sudo apt install /mnt/$DEBS_LINUX_IMAGE"
        fi

        #安装 linux-image-dbg
        if [ -e $DEBS_KERNEL_PATH/$DEBS_LINUX_IMAGE_DBG ];then
                chroot $TEMP_ROOTFS /bin/sh -c "sudo apt install /mnt/$DEBS_LINUX_IMAGE_DBG"
        fi

        #安装 linux-headers
        if [ -e $DEBS_KERNEL_PATH/$DEBS_LINUX_HEADERS ];then
                chroot $TEMP_ROOTFS /bin/sh -c "sudo apt install /mnt/$DEBS_LINUX_HEADERS"
        fi

        #安装 linux-libc-dev
        if [ -e $DEBS_KERNEL_PATH/$DEBS_LINUX_LIBC ];then
                chroot $TEMP_ROOTFS /bin/sh -c "sudo apt install /mnt/$DEBS_LINUX_LIBC"
        fi
    fi

    #chroot到$TEMP_ROOTFS系统并安装相关的deb包,安装完后退出rootfs系统
    echo "====安装其它deb包"
    if [ -d $DEBS_PATH ];then
        umount $TEMP_ROOTFS/mnt/
        mount --bind $DEBS_PATH/ $TEMP_ROOTFS/mnt/
	chroot $TEMP_ROOTFS /bin/sh -c "sudo apt install /mnt/*.deb"
    fi

    #制作安装器vmlinuz文件和initrd文件
    echo "====制作安装器vmlinuz文件和initrd文件"
    cp $TEMP_ROOTFS/boot/vmlinuz-$DEBS_KERNEL_RELEASE $TEMP_ISO/live/vmlinuz
    cp $TEMP_ROOTFS/boot/initrd.img-$DEBS_KERNEL_RELEASE $TEMP_ISO/live/initrd.img

    #卸载dev和proc文件系统
    echo "====卸载dev和proc文件系统"
    umount $TEMP_ROOTFS/proc
    umount $TEMP_ROOTFS/sys
    umount $TEMP_ROOTFS/dev/pts
    umount $TEMP_ROOTFS/dev
    umount $TEMP_ROOTFS/mnt
    
    #将$TEMP_ROOTFS压缩为filesystem.squashfs
    echo "====将$TEMP_ROOTFS压缩为filesystem.squashfs"
    rm $TEMP_ISO/live/filesystem.squashfs
    mksquashfs $TEMP_ROOTFS $TEMP_ISO/live/filesystem.squashfs -comp xz
    
    #将iso目录压缩为iso镜像
    echo "====将rootfs压缩为$TEMP_ISO"
    xorriso -as mkisofs -r -J -c boot.cat -boot-load-size 4 -boot-info-table \
    -eltorito-alt-boot -no-emul-boot -V "uos 20" -file_name_limit 250 -o  \
    $ISO_DEST_PATH/$ISO_DEST_NAME $TEMP_ISO
    
    umount /mnt
    rm -rf $TEMP_ROOTFS $TEMP_ISO

    echo "====定制完成"
    echo "定制镜像存放于：$ISO_SOURCE_PATH/$ISO_DEST_NAME"
    exit
fi
```