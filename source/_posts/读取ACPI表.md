---
title: 读取ACPI表
date: 2024-03-08 16:20:02
tags:
---

## 什么是ACPI

ACPI是高级配置和电源接口的缩写。本意是要定义一套标准的、体系架构无关的接口用于实现对硬件的检测、配置和电源管理。由Intel，微软，东芝等诸多公司参与指定的行业规范，在X86和ARM系统中都有使用。

## 应用如何读取ACPI

```
sudo apt install acpica-tools
```

* iasl: compiles ASL (ACPI Source Language) into AML (ACPI Machine Language), suitable for inclusion as a DSDT in system firmware.It also can disassemble AML, for debugging purposes.
* acpibin: performs basic operations on binary AML files (e.g., comparison, data extraction)
* acpidump: write out the current contents of ACPI tables
* acpiexec: simulate AML execution in order to debug method definitions
* acpihelp: display help messages describing ASL keywords and op-codes
* acpisrc: manipulate the ACPICA source tree and format source files for specific environments
* acpixtract: extract binary ACPI tables from acpidump output (see also the pmtools package)

### 列出所有ACPI的表
```
root@vm:~# acpidump -s
ACPI: SSDT 0x0000000000000000 0000CA (v01 BOCHS  VMGENID  00000001 BXPC 00000001)
ACPI: APIC 0x0000000000000000 000090 (v01 BOCHS  BXPC     00000001 BXPC 00000001)
ACPI: WAET 0x0000000000000000 000028 (v01 BOCHS  BXPC     00000001 BXPC 00000001)
ACPI: DSDT 0x0000000000000000 003DD4 (v01 BOCHS  BXPC     00000001 BXPC 00000001)
ACPI: FACP 0x0000000000000000 000074 (v01 BOCHS  BXPC     00000001 BXPC 00000001)
ACPI: HPET 0x0000000000000000 000038 (v01 BOCHS  BXPC     00000001 BXPC 00000001)
ACPI: FACS 0x0000000000000000 000040
```

### 生成某个表的二进制文件
```
root@vm:~# acpidump -n DSDT -b
root@vm:~# ls
dsdt.dat
```

### 把某个表的二进制文件转换成可读文件
```
root@vm:~# iasl -d dsdt.dat

Intel ACPI Component Architecture
ASL+ Optimizing Compiler/Disassembler version 20190509
Copyright (c) 2000 - 2019 Intel Corporation

File appears to be binary: found 4349 non-ASCII characters, disassembling
Binary file appears to be a valid ACPI table, disassembling
Input file dsdt.dat, Length 0x3DD4 (15828) bytes
ACPI: DSDT 0x0000000000000000 003DD4 (v01 BOCHS  BXPC     00000001 BXPC 00000001)
Pass 1 parse of [DSDT]
Pass 2 parse of [DSDT]
Parsing Deferred Opcodes (Methods/Buffers/Packages/Regions)

Parsing completed
Disassembly completed
ASL Output:    dsdt.dsl - 144810 bytes
```

### 读某个表的可读文件

节选自 dsdt.dsl 文件

```
Scope (_SB)
{
    Device (PCI0)
    {
        Name (_HID, EisaId ("PNP0A03") /* PCI Bus */)  // _HID: Hardware ID
        Name (_ADR, Zero)  // _ADR: Address
        Name (_UID, Zero)  // _UID: Unique ID
        Method (EDSM, 5, Serialized)
        {
            If ((Arg2 == Zero))
            {
                Local0 = Buffer (One)
                    {
                            0x00                                             // .
                    }
                If ((Arg0 != ToUUID ("e5c937d0-3553-4d7a-9117-ea4d19c3434d") /* Device Labeling Interface */))
                {
                    Return (Local0)
                }

                If ((Arg1 < 0x02))
                {
                    Return (Local0)
                }

                Local0 [Zero] = 0x81
                Return (Local0)
            }

            If ((Arg2 == 0x07))
            {
                Local0 = Package (0x02)
                    {
                        Zero, 
                        ""
                    }
                Local1 = DerefOf (Arg4 [Zero])
                Local0 [Zero] = Local1
                Return (Local0)
            }
        }
    }
}
```

## 内核如何读取ACPI

```
	/*
	 * "num-pins" is the total number of interrupt pins implemented in
	 * this mbigen instance, and mbigen is an interrupt controller
	 * connected to ITS  converting wired interrupts into MSI, so we
	 * use "num-pins" to alloc MSI vectors which are needed by client
	 * devices connected to it.
	 *
	 * Here is the DSDT device node used for mbigen in firmware:
	 *	Device(MBI0) {
	 *		Name(_HID, "HISI0152")
	 *		Name(_UID, Zero)
	 *		Name(_CRS, ResourceTemplate() {
	 *			Memory32Fixed(ReadWrite, 0xa0080000, 0x10000)
	 *		})
	 *
	 *		Name(_DSD, Package () {
	 *			ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
	 *			Package () {
	 *				Package () {"num-pins", 378}
	 *			}
	 *		})
	 *	}
	 */
	ret = device_property_read_u32(&pdev->dev, "num-pins", &num_pins);
	if (ret || num_pins == 0)
		return -EINVAL;
```

