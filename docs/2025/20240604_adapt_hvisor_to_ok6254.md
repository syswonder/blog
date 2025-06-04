# hvisor适配ok6254开发板

时间：2025/6/4

作者：侯云龙


# 0. ok6254开发板相关知识


[01_AM62X简介](https://forlinx-book.yuque.com/wl9002/ligiet/uf6t5hneigb2nes0)-核心板

[03_OK62xx-C嵌入式开发平台介绍](https://forlinx-book.yuque.com/wl9002/ligiet/vx43ckb0bazphd2y)-开发板接口相关

----

[06_开发板系统烧写](https://forlinx-book.yuque.com/wl9002/ligiet/edhhragnmlykha6y)-烧录

---

[04_相关代码编译](https://forlinx-book.yuque.com/wl9002/ligiet/uvhqnayx5nzz1nrz)-Linux编译/设备树获取

# 1. hvisor对开发板的适配使用

参考文档[hvisor 硬件适配开发手册 - hvisor 手册](https://hvisor.syswonder.org/chap01/PlatformDev.html)

[hvisor如何适配新板子——以aarch64 rk3568为例](https://kindly-tellurium-6bf.notion.site/hvisor-aarch64-rk3568-1c11786f530480928d00e93f9f7801c7)

## 1.1 hvisor-uart

hvisor通过调试串口输出调试信息

`src/device/uart/ok62xx_uart.rs`开发板的调试串口输出

`src/device/uart/mod.rs`中添加该串口相关信息





## 1.2.hvisor-其他配置

```
platform/aarch64/ok62xx-c
├── board.rs 
├── cargo
│   ├── config.template.toml
│   └── features
├── configs
│   ├── virtio_cfg.json
│   └── zone1-linux.json
├── image
│   └── dts
│       ├── Makefile
│       ├── zone0.dts
│       ├── zone1.dts
│       └── zone1-linux.dts
├── linker.ld
└── platform.mk
```

board.rs：添加设备树地址/内核加载地址/内核入口地址/RAM映射区域/IO映射区域/ROOT Liunx irq/gicd,gicr,gicc,gich,gicv,gits的基地址和大小

gic相关地址从设备树中获取

**features：对应的具体cargo features，uart**

virtio_cfg.json：见后续启动zone1

zone1-linux.json：见后续启动zone1

zone0.dts：zone0设备树-基本上是原有设备树

zone1.dts：zone1设备树-精简设备树

linker.ld：hvisor链接文件，修改相关启动地址

platform.mk：修改hvisor地址，zone0kernel，dtb地址

## 1.3 加载启动hvisor

通过uboot将sd卡中hvisor.bin和OK6254-C.dtb设备树文件加载到内存并启动（或可通过tftp传输）

查看sd卡中文件

```
mmc list
mmc dev 1
mmcinfo
fatls mmc 1:1
```

加载hvisor并启动

```
load mmc 1:1 0x80000000 OK6254-C.dtb
load mmc 1:1 0x80400000 hvisor.bin
md 0x80000000 0x100
md 0x80400000 0x100
bootm 0x80400000 - 0x80000000
```

## 1.4 hvisor适配开发板过程中遇到的问题汇总

1. hvisor无法使用串口输出：确定串口驱动类型  或  编写裸板程序，确定串口输出所使用的uart寄存器等，再编写hvisor的串口输出的相关代码详细流程如下：用C语言写一个裸板程序，用交叉编译工具链编译链接，使用uboot将裸板程序加载到内存，然后go启动运行；能够成功通过串口输出调试信息则将其转成rust语言并添加`src/device/uart/ok62xx_uart.rs`，参考[UART 16550的使用-CSDN博客](https://blog.csdn.net/qq_45226456/article/details/142097210)





# 2. 通过hvisor启动zone0

## 2.1 zone0镜像和设备树
zone0镜像为厂商提供原有镜像，通过sd卡加载到内存，后经hvisor启动

加载hvisor镜像设备树，zone0设备树，镜像，并启动hvisor 

```
load mmc 1:1 0x90000000 OK6254-C-linux.dtb
load mmc 1:1 0x90400000 Image
load mmc 1:1 0x80000000 OK6254-C.dtb
load mmc 1:1 0x80400000 hvisor.bin
bootm 0x80400000 - 0x80000000
```

## 2.2 基于设备树文件修改board.rs使其启动zone0

board.rs：添加设备树地址/内核加载地址/内核入口地址/RAM映射区域/IO映射区域/ROOT Liunx irq/gicd,gicr,gicc,gich,gicv,gits的基地址和大小


相关代码见
[board.rs](https://github.com/ohhhHwH/hvisor/blob/dev/platform/aarch64/ok62xx-c/board.rs)




## 2.2 hvisor启动zone0中遇见的问题汇总

1.多核启动问题

ROOT_ZONE_CPUS启动CPU 0b1111代表启动四个核

```rust
pub const ROOT_ZONE_CPUS: u64 = (1 << 0) | (1 << 1) |  (1 << 2)|  (1 << 3);
```

2.gic初始化问题-gic相关地址从设备树中获取

```dts
interrupt-controller@1800000 {
			compatible = "arm,gic-v3";
			#address-cells = <0x02>;
			#size-cells = <0x02>;
			ranges;
			#interrupt-cells = <0x03>;
			interrupt-controller;
			reg = <0x00 0x1800000 0x00 0x10000 0x00 0x1880000 0x00 0xc0000>;
			interrupts = <0x01 0x09 0x04>;
			phandle = <0x01>;

			msi-controller@1820000 {
				compatible = "arm,gic-v3-its";
				reg = <0x00 0x1820000 0x00 0x10000>;
				socionext,synquacer-pre-its = <0x1000000 0x400000>;
				msi-controller;
				#msi-cells = <0x01>;
				phandle = <0x63>;
			};
		};
```

3.内存覆盖问题

出现异常内存访问，需要在broad.rs的ROOT_ZONE_MEMORY_REGIONS中将其内存范围进行映射

4.linux启动过程中卡死

对比正常启动liunx的logo信息，确定没有启动的设备或异常访问的内存范围

5.内核vfs挂载失败

设备树中chosen选择相应的挂载位置

# 3. 在zone0中使用hvisor-tool启动zone1


## 3.1 编译hvisor-tool

[Hvisor 管理工具 - hvisor 手册](https://hvisor.syswonder.org/chap04/subchap04/ManageTools.html)


[hvisor-tool/README-zh.md at main · syswonder/hvisor-tool](https://github.com/syswonder/hvisor-tool/blob/main/README-zh.md)


## 3.2 zone0设备树修改

**zone0.dts**

添加设备树节点

```
hvisor_virtio_device {
		compatible = "hvisor";
		interrupt-parent = <0x01>;
		interrupts = <0x00 0x20 0x01>;
	};
```

添加reserved-memory使，这块地址不被建表

```
reserved-memory {
		#address-cells = <0x02>;
		#size-cells = <0x02>;
		ranges;

		nonroot@b0000000 {
			no-map;
			reg = <0x00 0xb0000000 0x00 0x20000000>;
		};
}
```


## 3.3 hvisor-tool启动相关配置文件修改

**virtio_cfg.json**

id号；memory_region内存映射区域；devicesvirtio设备树

**zone1-liunx.json**

映射zone1需要的io设备、virtio设备内存地址和ram区域

## 3.4 zone1设备树裁剪

精简zone1设备树，保留必要的设备节点,并保证zone0所占用的设备/地址不冲突。

裁剪后设备树见:
[ok6254-zone1.dts](https://github.com/ohhhHwH/hvisor/blob/dev/platform/aarch64/ok62xx-c/image/dts/ok6254-zone1.dts)


添加virtio blk等virtio设备节点

```
 // virtio blk
	virtio_mmio@a003c00 {
		dma-coherent;
		interrupt-parent = <0x01>;
		interrupts = <0x0 0x2e 0x1>;
		reg = <0x0 0xa003c00 0x0 0x2000>;  
		compatible = "virtio,mmio";
	};
```

----

## 3.5 zone1启动过程中所遇到的问题

1.hvisor-tool加载hvisor.ko内核模块运行提示没有virtio-device：没有添加设备树节点

2.root linux运行./hvisor提示glibc版本不匹配：hvisor-tool未指定正确的文件系统路径

3.linux启动过程中卡死：对比正常启动liunx的logo信息，确定没有启动的设备或异常访问的内存范围

4.zone1启动导致zone0的mmc卡死：zone1占用了zone0的中断控制器导致zone0无法收到中断

5.zone1启动后物理串口设备未成功启动：zone1的物理串口使用的时钟被zone0占用，需要修改为fix clock

6./bin/sh后未正常输出：zone1.json中未正确配置相关中断，sh程序需要串口的相关中断才能正常运行。












