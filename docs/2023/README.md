# 矽望社区技术报告 - 2023


## 使用 Jtag 在树莓派上调试 ArceOS

**时间：** 2023年6月1日

**作者：** [袁世平](robert_yuan@pku.edu.cn)

**摘要：** 这篇文章主要讲述连接树莓派需要准备的硬件和软件，同时讲述了连接的方法和调试的方法，对于移植部分只是基于非常浅的讲述了最基本的串口地址的修改，以及调试所需要的对内核起始地址的修改。

[正文](Raspi4-debug-with-jtag.md)

## 移植 ArceOS 到 树莓派4 上的问题和解决方法

**时间：** 2023年6月1日

**作者：** [袁世平](robert_yuan@pku.edu.cn)

**摘要：** 这篇文章主要讲述了如何在树莓派4上运行 ArceOS, 涉及到的内容是树莓派4在各个方面与qemu-virt 之间的差异, 需要对代码进行的修改,包括启动、串口、中断、多核、存储（Ramdisk）等方面。

[正文](How-to-run-ArceOS-on-raspi4.md)


## 如何让jailhouse运行在模拟ARM AArch64的QEMU上

**时间：** 2023年4月21日

**作者：** 杨竣轶

**摘要：** 讲述如何从零编译QEMU、Linux内核、构建ubuntu安装镜像、编译安装jailhouse的全过程。

[正文](20230421_ARM64-QEMU-jailhouse.md)


## ARM AArch64的中断虚拟化

**时间：** 2023年4月21日

**作者：** 丁韶峰

**摘要：** 介绍ARM GIC及其虚拟化相关知识。

[正文](20230421_gicv.md)


## 使用gdb调试jailhouse步骤

**时间：** 2023年4月14日

**作者：** 李柯樾

**摘要：** 如何使用gdb调试jailhouse的step by step 说明。

[正文](20230414_gdb_debug_jailhouse.md)


## ARM SMMU 

**时间：** 2023年4月14日

**作者：** 陈星宇

**摘要：** 介绍ARM 中 SMMU 的相关知识及jailhouse的相关设计。

[正文](20230414_ARM_SMMU.md)
