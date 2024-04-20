# 矽望社区技术报告节选

## 在hvisor上使用Virtio磁盘和网络设备

时间：2024年4月15日

作者：李国玮

摘要：介绍如何在hvisor上使用Virtio-blk和Virtio-net设备。

[正文](2024/20240415_Virtio_devices_tutorial.md)

## RISC-V APLIC 线中断总结

**时间：** 2024年3月11日

**作者：** 刘景宇

摘要：介绍了RISC-V架构中断控制器APLIC，主要包含直接传递中断方式（即非MSI）的部分。

[正文](2024/20240311_APLIC.md)

## 在RuxOS上支持c++

**时间：** 2024年2月29日

**作者：** 徐金阳

摘要：介绍了在RuxOS上解决编译时c++头文件找不到，以及链接时c++标准库符号找不到的方法，从而可以在RuxOS上编译运行c++应用程序。

[正文](2024/20240229_Support_c++_on_RuxOS.md)

## NXP OK8MP 开发板启动Jailhouse

**时间：** 2024年2月23日

**作者：** 杨竣轶

**摘要：** 介绍了在NXP OK8MP开发板上启动Jailhouse的详细过程与配置

[正文](2023/20240223_NXP_Boot_Jailhouse_Tutorial.md)

## RukOS中9pfs的实现与使用

**时间：** 2023年12月6日

**作者：** 吴政

**摘要：** 主要介绍了RukOS的9pfs模块的使用方式，以及它的具体实现原理。

[正文](2023/20231113_how-9pfs-is-integrated-to-rukos.md)

## 在 rukos 上支持 nginx

**时间：** 2023年11月28日

**作者：** 刘昊文

**摘要：** 介绍了 如何在rukos上运行nginx

[正文](2023/20231128_nginx_run.md)

## 在 rukos 上支持 musl libc

**时间：** 2023年11月15日

**作者：** 晏巨广

**摘要：** 介绍了 musl libc 的调用链和集成过程中遇到的困难，以及 APP 的测试结果

[正文](2023/20231115_MUSL_on_Rukos.md)

## C程序的启动过程

**时间：** 2023年11月9日

**作者：** 刘昊文 吴政 陈正宁 朱若海

**摘要：** 介绍了从系统到用户函数的c程序的启动过程，以及在启动时的栈的情况

[正文](2023/20231109_cstart.md)

## ArceOS 运行 Redis

**时间：** 2023年7月30日

**作者：** 晏巨广

**摘要:** 阐述了 `ArceOS` 兼容 `Redis` 过程中遇到的问题，以及在 qemu 和真机上的运行方法。

[正文](2023/20230730_Redis_On_ArceOS.md)

## 使用qemu测试sysHyper步骤

**时间：** 2023年7月23日

**作者：** 李柯樾

**摘要：** 讲述在qemu上编译测试sysHyper的步骤

[正文](2023/20230723_Demonstration_in_QEMU.md)

## Linux中的Memory HotPlug机制

**时间：** 2023年7月21日

**作者：** 杨竣轶

**摘要：** 讲述Linux中的Memory Hotplug机制的原理与实现

[正文](2023/20230721_Linux_Memory_Hotplug.md)


## 移植 ArceOS 到 树莓派4 上的问题和解决方法

**时间：** 2023年6月1日

**作者：** 袁世平

**摘要：** 这篇文章主要讲述了如何在树莓派4上运行 ArceOS, 涉及到的内容是树莓派4在各个方面与qemu-virt 之间的差异, 需要对代码进行的修改,包括启动、串口、中断、多核、存储（Ramdisk）等方面。

[正文](2023/20230601_How-to-run-ArceOS-on-raspi4.md)


## 如何让jailhouse运行在模拟ARM AArch64的QEMU上

**时间：** 2023年4月21日

**作者：** 杨竣轶

**摘要：** 讲述如何从零编译QEMU、Linux内核、构建ubuntu安装镜像、编译安装jailhouse的全过程。

[正文](2023/20230421_ARM64-QEMU-jailhouse.md)

## 在Arm64 Qemu上运行Jailhouse

**时间：** 2023年4月21日

**作者：** 杨竣轶

**摘要：** 讲述如何从零编译QEMU、Linux内核、构建ubuntu安装镜像、编译安装jailhouse的全过程。

[正文](2023/20230421_ARM64-QEMU-jailhouse.md)


## 使用gdb调试jailhouse步骤

**时间：** 2023年4月14日

**作者：** 李柯樾

**摘要：** 如何使用gdb调试jailhouse的step by step 说明。

[正文](2023/20230414_gdb_debug_jailhouse.md)









