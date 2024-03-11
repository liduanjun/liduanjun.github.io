---
title: 内核中使用dmi区分设备
date: 2024-03-11 15:14:07
tags:
---
## 什么是 DMI

DMI（Desktop Management Interface,DMI）就是帮助收集电脑系统信息的管理系统，DMI信息的收集必须在严格遵照SMBIOS规范的前提下进行。SMBIOS（System Management BIOS）是主板或系统制造者以标准格式显示产品管理信息所需遵循的统一规范。SMBIOS和DMI是由行业指导机构Desktop Management Task Force(DMTF)起草的开放性的技术标准，其中DMI设计适用于任何的平台和操作系统。

DMI充当了管理工具和系统层之间接口的角色。它建立了标准的可管理系统更加方便了电脑厂商和用户对系统的了解。DMI的主要组成部分是Management Information Format(MIF)数据库。这个数据库包括了所有有关电脑系统和配件的信息。通过DMI，用户可以获取序列号、电脑厂商、串口信息以及其它系统配件信息。

## 使用dmidecode获取DMI信息

dmidecode命令 可以让你在Linux系统下获取有关硬件方面的信息。dmidecode的作用是将DMI数据库中的信息解码，以可读的文本方式显示。由于DMI信息可以人为修改，因此里面的信息不一定是系统准确的信息。dmidecode遵循SMBIOS/DMI标准，其输出的信息包括BIOS、系统、主板、处理器、内存、缓存等等。

### 安装 dmidecode
```
# sudo apt install dmidecode
# dmidecode -h
Usage: dmidecode [OPTIONS]
Options are:
 -d, --dev-mem FILE     Read memory from device FILE (default: /dev/mem)
 -h, --help             Display this help text and exit
 -q, --quiet            Less verbose output
 -s, --string KEYWORD   Only display the value of the given DMI string
 -t, --type TYPE        Only display the entries of given type
 -H, --handle HANDLE    Only display the entry of given handle
 -u, --dump             Do not decode the entries
     --dump-bin FILE    Dump the DMI data to a binary file
     --from-dump FILE   Read the DMI data from a binary file
     --no-sysfs         Do not attempt to read DMI data from sysfs files
     --oem-string N     Only display the value of the given OEM string
 -V, --version          Display the version and exit
```

### 使用 dmidecode
```
# sudo dmidecode
SMBIOS 2.5 present.

Handle 0x0001, DMI type 4, 40 bytes
Processor Information
        Socket Designation: Node 1 Socket 1
        Type: Central Processor
        Family: Xeon MP
        Manufacturer: Intel(R) Corporation
        id: C2 06 02 00 FF FB EB BF
        Signature: Type 0, Family 6, Model 44, Stepping 2
        Flags:
                FPU (Floating-point unit on-chip)
                VME (Virtual mode extension)
                DE (Debugging extension)
                PSE (Page size extension)
                TSC (time stamp counter)
                MSR (Model specific registers)
                PAE (Physical address extension)
                MCE (Machine check exception)
                CX8 (CMPXCHG8 instruction supported)
                APIC (On-chip APIC hardware supported)
                SEP (Fast system call)
                MTRR (Memory type range registers)
                PGE (Page global enable)
                MCA (Machine check architecture)
                CMOV (Conditional move instruction supported)
                PAT (Page attribute table)
                PSE-36 (36-bit page size extension)
                CLFSH (CLFLUSH instruction supported)
                DS (Debug store)
                ACPI (ACPI supported)
                MMX (MMX technology supported)
                FXSR (FXSAVE and FXSTOR instructions supported)
                SSE (Streaming SIMD extensions)
                SSE2 (Streaming SIMD extensions 2)
                ss (Self-snoop)
                HTT (Multi-threading)
                TM (Thermal monitor supported)
                PBE (Pending break enabled)
        Version: Intel(R) Xeon(R) CPU           E5620  @ 2.40GHz
        Voltage: 1.2 V
        External Clock: 5866 MHz
        Max Speed: 4400 MHz
        Current Speed: 2400 MHz
        Status: Populated, Enabled
        Upgrade: ZIF Socket
        L1 Cache Handle: 0x0002
        L2 Cache Handle: 0x0003
        L3 Cache Handle: 0x0004
        Serial Number: Not Specified
        Asset Tag: Not Specified
        Part Number: Not Specified
        Core Count: 4
        Core Enabled: 4
        Thread Count: 8
        Characteristics:
                64-bit capable

Handle 0x0055, DMI type 4, 40 bytes
Processor Information
        Socket Designation: Node 1 Socket 2
        Type: Central Processor
        Family: Xeon MP
        Manufacturer: Not Specified
        ID: 00 00 00 00 00 00 00 00
        Signature: Type 0, Family 0, Model 0, Stepping 0
        Flags: None
        Version: Not Specified
        Voltage: 1.2 V
        External Clock: 5866 MHz
        Max Speed: 4400 MHz
        Current Speed: Unknown
        Status: Unpopulated
        Upgrade: ZIF Socket
        L1 Cache Handle: Not Provided
        L2 Cache Handle: Not Provided
        L3 Cache Handle: Not Provided
        Serial Number: Not Specified
        Asset Tag: Not Specified
        Part Number: Not Specified
        Characteristics: None
```

## 在内核中使用dmi区分设备

有时候需要在内核中区分不同的设备，比如不同厂商设置的最大背光亮度值是不一样的，
有的厂商设置为100，有的设置为10，这时候可以用dmi来区分不同设备，针对性地进行设置。

```
//callback 函数
static int video_set_bqc_offset(
	const struct dmi_system_id *d)
{
	if (hw_changes_brightness == -1)
		hw_changes_brightness = 1;
	return 0;
}

//特例表，如果当前设备匹配到了这个表，则调用 callback 函数进行特殊处理
//有好几个匹配项目，一般用 DMI_BOARD_VENDOR ， DMI_BOARD_NAME 等
//其它的见内核源码 include/linux/mod_devicetable.h
static const struct dmi_system_id video_dmi_table[] = {
	{
	 .callback = video_set_bqc_offset,
	 .ident = "Acer Aspire 5720",
	 .matches = {
		DMI_MATCH(DMI_BOARD_VENDOR, "Acer"),
		DMI_MATCH(DMI_PRODUCT_NAME, "Aspire 5720"),
		},
	},
    {}
};

//在合适的位置（比如 init 函数里）使用此函数去检查特例表。
dmi_check_system(video_dmi_table);
```