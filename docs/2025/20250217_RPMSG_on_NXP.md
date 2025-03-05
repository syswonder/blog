# 在NXP上通过RPMsg实现Linux与Xiuos通信
时间：2025/2/17

作者：李国玮

为了在NXP imx8mp上实现运行在A核上的Linux与运行在M核上的Xiuos通过RPMsg进行通信，需要依次进行如下几个步骤：

1. 将Xiuos移植到imx8mp开发板上
2. 将RPMsg-lite移植到Xiuos
3. 为Linux启用RPMsg功能，确保Linux上的应用程序可以通过RPMsg驱动程序进行通信
4. 为Linux与Xiuos编写进行通信的应用程序

## 移植xiuos到nxp

移植xiuos主要涉及两个目录：

APP_Framework/Applications：存放应用程序

Ubiquitous/XiZi_IIoT/board：存放不同开发板的BSP

![无标题](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/2024/无标题.png)

移植过程：

1. 运行裸机hello world程序：从nxp厂商提供的BSP中，找到一个裸机hello world程序对应的所有源文件以及编译链接流程，编译完成后通过uboot在M核上启动。

2. 将裸机hello world程序对接到xiuos：将裸机程序的BSP移植到board目录，将print hello world作为应用程序运行。主要是编译链接会出各种问题。

   移植过程中，重点需要关注的外设是uart。因为xiuos采用总线机制统一各种驱动接口，因此裸机程序的BSP中的uart驱动需要和xiuos进行对接，对接时需要编写connect_uart.c，该文件如何编写可参考其他board的类似文件。

3. 为xiuos实现rpmsg功能：将rpmsg-lite移植到xiuos，并写一个利用rpmsg与A核通信的应用程序。

rpmsg-lite是nxp为M核开发的一个轻量级rpmsg框架，应用程序利用rpmsg-lite即可通信。移植rpmsg-lite的主要工作：

1. rpmsg_env_bm.c提供了内存分配等函数，需要替换成xiuos的内存分配等函数。
2. rpmsg底层用到了virtio实现数据传输和通知，A核会通过中断通知M核接收数据，需要利用xiuos的中断注册函数注册对应的中断处理函数。

## 为linux启用rpmsg功能

主线linux对rpmsg支持不够完善，需要打个linux论坛上提供的[patch](https://lwn.net/Articles/743115/)，以便应用程序可以通过对/dev/rpmsg_ctrl0和/dev/rpmsg0进行操作，实现rpmsg的收发。

> 打完patch后，执行*make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- menuconfig*，
>
> 选中Device Drivers > Rpmsg drivers中的所有内容。

设备树中需要加入rpmsg所需的节点：

```
reserved-memory {
    #address-cells = <0x02>;
    #size-cells = <0x02>;
    ranges;
    // vring0和vring1分别是两个virtqueue，用于rpmsg的收发
    vdev0vring0@55000000 {
        compatible = "shared-dma-pool";
        reg = <0x00 0x55000000 0x00 0x8000>;
        no-map;
        phandle = <0x09>;
    };

    vdev0vring1@55008000 {
        compatible = "shared-dma-pool";
        reg = <0x00 0x55008000 0x00 0x8000>;
        no-map;
        phandle = <0x0a>;
    };
    // IO buffer
    vdevbuffer@55400000 {
        compatible = "shared-dma-pool";
        reg = <0x00 0x55400000 0x00 0x100000>;
        no-map;
        phandle = <0x0b>;
    };

    rsc-table {
        reg = <0x00 0x550ff000 0x00 0x1000>;
        no-map;
    };

    rpmsg@0x55800000 {
        no-map;
        reg = <0x00 0x55800000 0x00 0x800000>;
    };

    imx8mp-cm7 {
        compatible = "fsl,imx8mp-cm7";
        rsc-da = <0x55000000>;
        clocks = <0x03 0x55>;
        mbox-names = "tx\0rx\0rxdb";
        mboxes = <0x08 0x00 0x01 0x08 0x01 0x01 0x08 0x03 0x01>;
        memory-region = <0x09 0x0a 0x0b>;
        status = "okay";
    };
};

rpmsg {
    compatible = "fsl,imx8mq-rpmsg";
    mbox-names = "tx\0rx\0rxdb";
    mboxes = <0x08 0x00 0x01 0x08 0x01 0x01 0x08 0x03 0x01>;
    status = "okay";
    vdev-nums = <0x01>;
    reg = <0x00 0x55000000 0x00 0x10000>;
    memory-region = <0x0b>;
};
// message unit，即mailbox。A核和M核对mu操作时，即可发送中断给对方
mu@30aa0000 {
    compatible = "fsl,imx8mp-mu\0fsl,imx6sx-mu";
    reg = <0x30aa0000 0x10000>;
    interrupts = <0x00 0x58 0x04>;
    clocks = <0x03 0xd5>;
    clock-names = "mu";
    #mbox-cells = <0x02>;
    phandle = <0x08>;
};
```

由于xiuos和linux会并行运行在nxp上，xiuos使用uart4，因此linux的设备树不能涉及对uart4有任何操作的节点，否则linux启动后xiuos输出会有问题。需要修改设备树：

```c
// 保证uart4为disabled状态
serial@30a60000 {
    compatible = "fsl,imx8mp-uart\0fsl,imx6q-uart";
    reg = <0x30a60000 0x10000>;
    interrupts = <0x00 0x1d 0x04>;
    clocks = <0x03 0xfe 0x03 0xfe>;
    clock-names = "ipg\0per";
    dmas = <0x22 0x1c 0x04 0x00 0x22 0x1d 0x04 0x00>;
    dma-names = "rx\0tx";
    status = "disabled";
};

// 还需要删除下面的节点，避免linux对uart4的pinctl进行修改，
// uart4grp {
// 	fsl,pins = <0x238 0x498 0x600 0x00 0x08 0x49 0x23c 0x49c 0x00 0x00 0x00 0x49>;
// 	phandle = <0x29>;
// };
// 特别地，uart1不能在pinctrl-0中引用uart4grp
serial@30890000 {
    compatible = "fsl,imx8mp-uart\0fsl,imx6q-uart";
    reg = <0x30890000 0x10000>;
    interrupts = <0x00 0x1b 0x04>;
    clocks = <0x03 0xfc 0x03 0xfc>;
    clock-names = "ipg\0per";
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <0x28>;
};

```

## linux与xiuos通信样例程序

* nxp上A核与M核的地址空间布局

M7核地址空间的0x4000_0000 -- 0xBFFF_FFFF和A53的是共享的，地址也是相同。

> 参考手册 i.MX 8M Plus Applications Processor Reference Manual。

* 通信流程图

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/2024/rpmsg示例程序.drawio.png" alt="rpmsg示例程序.drawio" style="zoom: 67%;" />

* 运行步骤

目前支持在M核上运行rpmsg例程，与A核上的Linux应用程序通过RPMsg通信。具体方式如下：

1. 通过uboot在M核上运行XiUOS：

   ```
   fatload mmc 1:1 0x80000000 XiZi-imx8mp.bin
   dcache flush
   bootaux 0x80000000
   ```

   然后在letter shell中输入CreateRPMsgTask，按下回车，启动rpmsg例程。例程将等待A核上的Linux初始化RPMsg通信通道。

2. 通过uboot在A核上启动Linux，Linux启动后，执行示例程序。

3. 在Linux端可观察到：

   ![image-20250305145941655](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/2024/image-20250305145941655.png)

4. 在XiUOS上可观察到：

   ![image-20250305150018190](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/2024/image-20250305150018190.png)

* 程序代码

xiuos中的rpmsg代码可见：[xiuos](https://www.gitlink.org.cn/xuos/xiuos/tree/prepare_for_master/Ubiquitous%2FXiZi_IIoT%2Fboard%2Fimx8mp%2Frpmsg_remote.c)。

Linux端的rpmsg应用程序可见：[hvisor-tool](https://github.com/syswonder/hvisor-tool/blob/main/tools/rpmsg_demo.c)。