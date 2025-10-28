# 矽望社区技术报告 - 2025

## DesignWare PCIe iATU介绍

时间： 2025/10/22

作者： 徐仲锴

摘要： 以NXP i.MX8MP的DWC驱动为例，介绍DesignWare PCIe iATU地址转换单元的功能。

[正文](20251022_DesignWare_PCIe_iATU_Notes.md)

## Hvisor在OK6254-C上的启动流程

时间： 2025/09/11

作者： 侯云龙

摘要：如何在OK6254-C上启动hvisor

[正文](20250911_OK6254-C_Run_hvisor.md)

## Genesis使用教程

时间： 2025/08/28

作者： 赵龙淳

摘要：如何安装使用genesis，以及常见问题说明。

[正文](20250828_genesis_tutorial.md)

## hvisor适配Phytium Pi的流程

时间：2025/08/27

作者：包子旭

摘要：介绍hvisor适配新板子Phytium Pi的流程

[正文](20250827_Adapt_Hvisor_to_phytium_pi.md)

## 龙芯 3A6000 PCIe 虚拟化调试笔记

时间：2025/8/20

作者：韩喻泷

摘要：包括龙芯 3A6000 上的 PCIe 虚拟化与直通、Guest 时钟中断、PREEMPT_RT 相关调试笔记

[正文](20250820_3A6000_PCIe_Debug_Notes.md)

## EIC7700x SoC上的L3 Cache Controller介绍

时间：2025/7/22

作者：刘景宇

摘要：介绍ESWIN的EIC7700x Soc上的L3 Cache Controller及其虚拟化的简要实现介绍

[正文](20250722_Cache_Contoller.md)

## Hvisor 在 Megrez 上的适配(zone0 启动)

时间：2025/7/10

作者：刘景宇

摘要：描述了Hvisor在Megrez上启动zone0的过程，为zone1的启动以及其他risc-v开发板的移植提供参考

[正文](20250710_Megrez_Start_Zone0.md)

## RuxOS伪终端的实现与其在sshd上的应用

时间：2025/6/20

作者：石全

摘要：RuxOS伪终端的实现与其在sshd上的应用

[正文](20250620_SSHD_Support_for_RuxOS.md)

## 在RuxOS中支持FUSE用户空间文件系统

时间：2025/6/18

作者：郑元昊

摘要：介绍在RuxOS中如何运行FUSE以及具体设计与实现

[正文](20250618_FUSE_In_RuxOS.md)

## hvisor适配ok6254的流程

时间：2025/6/4

作者：侯云龙

摘要：介绍hvisor适配新板子ok6254的流程

[正文](20250604_Adapt_Hvisor_to_ok6254.md)

## 在rk3588上通过hvisor启动64位zephyr

时间：2025/5/31

作者：李国玮

摘要：介绍zephyr，以及如何将zephyr移植到rk3588上，并通过hvisor将zephyr作为non root zone启动。

[正文](20250531_Zephyr_on_hvisor.md)

## 在NXP上使用Hvisor运行ruxos

时间：2025/5/06

作者：刘昊文

摘要：介绍如何在NXP OK8MP开发板上通过hvisor将ruxos系统运行在non root zone上

[正文](20250506_Hvisor_Rux.md)

## hvisor如何适配新板子——以aarch64 rk3568为例

时间：2025/4/3

作者：陈星宇

摘要：介绍hvisor如何适配新板子，以aarch64 rk3568为例

[正文](20250403_How_to_Adapt_Hvisor_to_a_New_Board--A_Case_Study_of_AArch64_RK3568.md)


## 在NXP上通过RPMsg实现Linux与Xiuos通信

时间：2025/2/17

作者：李国玮

摘要：介绍如何在nxp imx8mp板子上，通过rpmsg实现Linux与Xiuos的通信

[正文](20250217_RPMSG_on_NXP.md)

## RuxOS伪终端的实现与其在sshd上的应用

时间：2025/6/20

作者：石全

摘要：介绍在RuxOS中如何实现伪终端并且介绍伪终端在ssh中的作用

[正文](20250620_SSHD_Support_for_RuxOS.md)

## 在Riscv-Ruxos中完善缺页中断与进程级fork功能

时间：2025/7/17

作者：曾俊

摘要：介绍在Riscv64架构下为Ruxos完善缺页中断与进程级fork的功能，同时提供Riscv64架构下对Busybox的支持

[正文](20250717_Pagefault_and_Fork_in_Riscv_for_Ruxos.md)