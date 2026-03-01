# Bootshim篇
编辑与`2026-03-01`  编辑者: [hxzbaka](https://github.com/hxzbaka) Ver:1.0

## 概述
Bootshim 是一个小型引导程序，用于在 UNISOC（SPRD）设备上通过现有的 bootloader（cboot/zboot）启动 UEFI 固件。它充当跳板，将 UEFI 镜像重定位到其预期基址，然后跳转执行。

## 工作原理
1. **Bootloader 加载组合镜像**  
   Bootloader（cboot 或 zboot）读取 `boot` 分区，该分区包含 `bootshim` + `UEFI`（实际固件）的拼接数据，并将其加载到一般内存地址 `0x80080000`执行，从 bootshim 入口开始

2. **重定位判断**  
   Bootshim 比较其当前运行时地址（加载位置）与期望的 UEFI 基址（编译时定义的 `UEFI_BASE`）
   - 如果两者相同，说明镜像已在正确位置，bootshim 直接跳转到 UEFI。  
   - 如果不同，bootshim 将整个 UEFI 负载（大小为 `UEFI_SIZE`）从当前位置复制到 `UEFI_BASE`

3. **复制循环**  
   使用 `ldp`/`stp` 指令以 16 字节为单位高效复制

4. **跳转到 UEFI**  
   成功重定位（或无需重定位）后，bootshim 通过绝对跳转指令转移到 `UEFI_BASE`，将控制权交给 UEFI 固件

## 注意
> 你必须解析你的设备dts以确保UEFI固件访问的内存范围不会被使用！！！

## 适配设备
- 1.解析设备dtb(转换为dts)


# 设备适配指南（UEFI + BootShim）

> ⚠ 注意  
> 你必须解析设备 DTS，确保 UEFI 固件访问的内存范围 **不会与系统已使用内存冲突**！！！

---

# 一、解析设备 DTB → DTS

## 1. 提取设备 boot.img

从设备固件中提取 `boot.img`。

可以使用任意 Android 解包工具，例如：
- Android Image Kitchen
- magiskboot
- unpackbootimg

解包 `boot.img` 后，会得到：

- kernel
- ramdisk
- dtb
- extra

⚠ 某些SPRD设备中，DTB 存放在 `extra` 中。

---

## 2. 去掉厂商索引

在任意 Linux 发行版终端使用工具

```
./_dt <extra>
```

去掉厂商索引后，会得到类似：

```
Boardname_id_1.dtb
```

---

## 3. 转换 DTB → DTS

使用 dtc 工具：

```
dtc -I dtb -O dts -o board.dts Boardname_id_1.dtb
```

---

# 二、分析内存结构

打开生成的 `board.dts`。

---

## 1. 查找 memory 节点

例如：

```
memory@80000000 {
    device_type = "memory";
    reg = <0x00 0x80000000 0x00 0x80000000>;
};
```

### reg 格式说明

```
reg = <高32位 起始地址  高32位 大小>;
```

示例：

```
reg = <0x00 0x80000000 0x00 0x80000000>;
```

表示：

- 起始地址：0x80000000
- 大小：0x80000000（2GB 逻辑映射）

⚠ 注意：

- kernel 映射内存 ≠ 实际物理 RAM
- 例如：
  - 逻辑映射：2GB
  - 实际物理：512MB

必须结合 reserved-memory 判断。

---

## 2. 查找 reserved-memory 节点

例如：

```
reserved-memory {
    #address-cells = <0x02>;
    #size-cells = <0x02>;
    ranges;

    mpu0-dump@877ff000 {
        reg = <0x00 0x877ff000 0x00 0x1000>;
    };

    sipc-mem@87800000 {
        reg = <0x00 0x87800000 0x00 0x3d0000>;
    };

    wcn-mem@88000000 {
        reg = <0x00 0x88000000 0x00 0x300000>;
    };

    ...
};
```

---

# 三、计算安全内存区域

观察：

```
memory:         0x80000000 ~ 0xA0000000
第一个 carveout: 0x877FF000
```

计算：

```
0x877FF000 - 0x80000000
= 0x077FF000
≈ 120MB
```

说明低 120MB 为“干净区”（未被 reserved-memory 占用）

---

## 安全区间

```
0x80000000 ~ 0x877FF000
≈ 120MB
```

---

# 四、BootShim 跳转地址选择


- 避开 reserved-memory
- 避开 U-Boot 释放 kernel 的地址
- 避开常规 ARM64 kernel 加载地址

常见情况：

```
0x80080000  → 常见 64 位 kernel 地址
```

因此推荐：

```
0x81000000
```

作为：

```
BootShim 跳转地址
UEFI_BASE
FD_BASE
```

---

## EDK2修改
平台DEC文件（例如:SC9832EPkg.dec)
```
[FD.SC9832EPKG_UEFI]
BaseAddress   = 0x81000000|gArmTokenSpaceGuid.PcdFdBaseAddress  # The base address of the Firmware in NOR Flash.
Size          = 0x00200000|gArmTokenSpaceGuid.PcdFdSize         # The size in bytes of the FLASH Device
ErasePolarity = 1
```
BaseAddress即FD_BASE
Size即FD_SIZE
要与Bootshim中保持相同

设备DSC文件
```
[PcdsFixedAtBuild.common]
  # System Memory (512M)
  gArmTokenSpaceGuid.PcdSystemMemoryBase|0x80000000
  gArmTokenSpaceGuid.PcdSystemMemorySize|0x20000000
  gEmbeddedTokenSpaceGuid.PcdPrePiStackBase|0x81000000
  gEmbeddedTokenSpaceGuid.PcdPrePiStackSize|0x00040000      # 256K stack
  gSC9832EPkgTokenSpaceGuid.PcdUefiMemPoolBase|0x81040000         # DXE Heap base address
  gSC9832EPkgTokenSpaceGuid.PcdUefiMemPoolSize|0x00100000         # UefiMemorySize, DXE heap size
  ```


## 构建方法
构建过程通常与 UEFI 构建系统集成。两个关键宏需要传递给汇编器：

- `UEFI_BASE` – UEFI 期望运行的物理地址（通常与 `FD_BASE` 相同）
- `UEFI_SIZE` – UEFI 固件二进制的大小（通常为 `FD_SIZE`）

## 构建指令
`make UEFI_BASE=0x81000000 UEFI_SIZE=0x00200000`大小和FD_SIZE一致，通常为2MB

## 使用
在产出的UEFI固件前拼接BootShim的二进制执行数据作为kernel替换进boot镜像中刷入

`cat BootShim.bin UEFI_FD.bin > kernel`