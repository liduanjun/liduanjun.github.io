---
title: arm服务器qemu支持qxl显示
date: 2024-05-08 09:45:03
tags:
---

## 问题

arm架构的服务器上，qemu不支持qxl显示，只支持virtio-gpu显示

## 解决办法

1. 修改 UEFI 代码

> 参考： https://www.kraxel.org/blog/2022/05/edk2-virt-quickstart/
> 源码： https://github.com/tianocore/edk2.git
> 补丁

```
diff --git a/ArmVirtPkg/ArmVirtQemu.dsc b/ArmVirtPkg/ArmVirtQemu.dsc
index 7e2ff33ad1..abb7828178 100644
--- a/ArmVirtPkg/ArmVirtQemu.dsc
+++ b/ArmVirtPkg/ArmVirtQemu.dsc
@@ -543,6 +543,7 @@
   OvmfPkg/QemuRamfbDxe/QemuRamfbDxe.inf
   OvmfPkg/VirtioGpuDxe/VirtioGpu.inf
   OvmfPkg/PlatformDxe/Platform.inf
+  OvmfPkg/QemuVideoDxe/QemuVideoDxe.inf
 
   #
   # USB Support
diff --git a/ArmVirtPkg/ArmVirtQemu.fdf b/ArmVirtPkg/ArmVirtQemu.fdf
index 764f652afd..7c3c788853 100644
--- a/ArmVirtPkg/ArmVirtQemu.fdf
+++ b/ArmVirtPkg/ArmVirtQemu.fdf
@@ -110,6 +110,7 @@ READ_LOCK_STATUS   = TRUE
   INF ArmVirtPkg/MemoryInitPei/MemoryInitPeim.inf
   INF ArmPkg/Drivers/CpuPei/CpuPei.inf
   INF MdeModulePkg/Core/DxeIplPeim/DxeIpl.inf
+  INF OvmfPkg/QemuVideoDxe/QemuVideoDxe.inf
 
 !if $(TPM2_ENABLE) == TRUE
   INF MdeModulePkg/Universal/PCD/Pei/Pcd.inf
```

2. 修改 qemu 代码

> 源码： https://github.com/qemu/qemu.git
> 补丁
```
diff --git a/hw/arm/Kconfig b/hw/arm/Kconfig
index 218b454e97..266b7820be 100644
--- a/hw/arm/Kconfig
+++ b/hw/arm/Kconfig
@@ -33,6 +33,7 @@ config ARM_VIRT
     select VIRTIO_MEM_SUPPORTED
     select ACPI_CXL
     select ACPI_HMAT
+    select QXL
 
 config CHEETAH
     bool
```

3. 修改linux内核 qxl 驱动代码

> 源码： https://github.com/torvalds/linux.git
> 补丁： 省略


