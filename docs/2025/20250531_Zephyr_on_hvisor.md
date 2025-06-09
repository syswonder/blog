# 在rk3588上通过hvisor启动64位zephyr
时间：2025/5/31

作者：李国玮

本文档主要介绍zephyr，以及如何将zephyr移植到rk3588上，并通过hvisor将zephyr作为non root zone启动。目前只支持启动64位zephyr，未来会启动32位，请关注本文档更新。

## zephyr介绍

zephyr是一个RTOS，支持A核、M核。目前官方仓库支持rk3568，但不支持3588。zephyr文档很全，可参考：

Introduction：https://docs.zephyrproject.org/latest/introduction/index.html

zephyr支持的所有板子：https://docs.zephyrproject.org/latest/boards/index.html#

Beyond the start guide：https://docs.zephyrproject.org/latest/develop/beyond-GSG.html#beyond-gsg

### west

[west](https://github.com/zephyrproject-rtos/west)是一个用于管理多个仓库的超级版Git工具。它通过一个west.yml文件来进行仓库的初始化和更新。默认情况下该文件使用[west.yml](https://github.com/zephyrproject-rtos/zephyr/blob/main/west.yml)。

west官方文档：https://docs.zephyrproject.org/latest/develop/west/index.html

### 板级支持

如何支持一个新板子：https://docs.zephyrproject.org/latest/hardware/porting/board_porting.html#board-porting-guide

RK3568：https://docs.zephyrproject.org/latest/boards/firefly/roc_rk3568_pc/doc/index.html

### kernel

#### 设备树解析

Linux的DTS会被编译为DTB，然后在启动时由Bootloader传递给kernel。但Zephyr运行在性能较差的嵌入式平台上，故不可能专门运行一个解析器去读DTB。

 因此，DTS实际上实在编译时被Zephyr的构建系统（一套python脚本）变成了头文件，这个头文件的位置是：

```
${project_folder}/build/zephyr/include/generated/devicetree_generated.h
```

 了解即可，实际开发不需要查看这个头文件。

这个头文件生成的流程：在dts目录下，除了有设备树描述设备信息以外，还有一个bindings子目录，是各个设备节点的元数据，描述设备树节点的内容。在编译时，设备树中compatible和bingdings中compatible相同节点的进行匹配，然后利用bindings将其转换为devicetree_generated.h。具体可以参考：https://docs.zephyrproject.org/latest/build/dts/howtos.html#dt-create-devices



* C代码如何使用设备树

DeviceTree最终会用来生成`devicetree_generated.h`头文件，包含了DeviceTree中的所有信息。自然而然的，我们会想到要在C/C++代码中访问这些信息。

>  注意，由于DeviceTree中节点名称、属性名称允许使用的字符集是比C语言变量命名所允许的字符集更广泛的，因此，Zephyr规定，在C语言中访问DeviceTree的内容时，名称内的字母全部都变成**小写字母**、且特殊符号都变成**下划线**。
>
>  例如`zephyr,user`变为`zephyr_user`；`my-gpio`变为`my_gpio`。

 我们无需关心`devicetree_generated.h`文件本身的内容，因为它不是给人看的，需要使用一套宏函数来将其读出。在需要操作DeviceTree的文件中包含以下头文件：

```c
#include <zephyr/devicetree.h>
```

#### threads

线程一般执行在内核态，应用程序和内核都可以使用线程。

> An **execution mode**, which can either be supervisor or user mode. By default, threads run in supervisor mode and allow access to privileged CPU instructions, the entire memory address space, and peripherals. User mode threads have a reduced set of privileges. This depends on the [`CONFIG_USERSPACE`](https://docs.zephyrproject.org/latest/kconfig.html#CONFIG_USERSPACE) option. See [User Mode](https://docs.zephyrproject.org/latest/kernel/usermode/index.html#usermode-api).
> 一般不会运行在user mode，除非有需要提高安全性的隔离需求，将线程运行在user mode。

* 优先级

数值越低，优先级越高。根据优先级分有两种线程：

- A *cooperative thread* has a negative priority value. Once it becomes the current thread, a cooperative thread remains the current thread until it performs an action that makes it unready.
- A *preemptible thread* has a non-negative priority value. Once it becomes the current thread, a preemptible thread may be supplanted at any time if a cooperative thread, or a preemptible thread of higher or equal priority, becomes ready. 此外，调度器每个时间片会检查preemptible threads，如果有同等优先级的，则切换线程。

优先级的应用场景：

Use cooperative threads for device drivers and other performance-critical work.

Use cooperative threads to implement mutually exclusion without the need for a kernel object, such as a mutex.

Use preemptive threads to give priority to time-sensitive processing over less time-sensitive processing.

##### main thread

内核的主线程，用于初始化，并跳转到应用程序的main函数。优先级默认最高。

#### memory management

如果开启CONFIG_MMU，则自动开启虚拟内存机制。虚拟内存的起始地址KERNEL_VM_BASE，和设备树中指定的zephyr,sram的起始地址一致。Image的text和data，和物理内存一般是恒等映射。

所以sram可以视为和dram一样的东西。linker.ld指定了text段的起始地址，等于物理内存的起始地址。

#### device model

https://docs.zephyrproject.org/latest/kernel/drivers/index.html

* uart

在.defconfig中，定义了CONFIG_UART_NS16550。在cmakelists.txt中，由于定义了CONFIG_UART_NS16550，会将uart_ns16550.c加入到源代码。RK3588 SoC 使用的 UART 控制器与 NS16550 兼容。

## zephyr移植

由于zephyr支持rk3568，而3568和3588比较相似，因此参考3568的板级支持以及Soc支持，即可移植到3588上。这里给出移植过程中遇到的一些难点以及解决方案。

### uart3乱码

zephyr运行在non root时，root linux使用uart2作为终端输出，zephyr使用uart3进行输出。要让zephyr可以正确使用uart3，首先要在设备树中加入uart3对应的节点，并在板级支持的.defconfig中，定义CONFIG_UART_NS16550。在cmakelists.txt中，由于定义了CONFIG_UART_NS16550，会将uart_ns16550.c加入到源代码。(RK3588 SoC 使用的 UART 控制器与 NS16550 兼容)

当我遇到输出乱码时，首先想到的是波特率没设对。经过前期验证，uart3要想输出，必须指定波特率为115200（通过root linux可以验证）。在zephyr uart驱动源码中，有函数：

```c
uart_ns16550_init-->
ret = uart_ns16550_configure(*dev*, &data->*uart_config*);
```

在uart_config这里包含了波特率的设置，而这个结构体是通过下面创建的：

```c
#define UART_NS16550_COMMON_DEV_DATA_INITIALIZER(n)                                  \
		.uart_config.baudrate = DT_INST_PROP_OR(n, current_speed, 0),        \
		.uart_config.parity = UART_CFG_PARITY_NONE,                          \
		.uart_config.stop_bits = UART_CFG_STOP_BITS_1,                       \
		.uart_config.data_bits = UART_CFG_DATA_BITS_8,                       \
		.uart_config.flow_ctrl =                                             \
			COND_CODE_1(DT_INST_PROP_OR(n, hw_flow_control, 0),          \
				    (UART_CFG_FLOW_CTRL_RTS_CTS),                    \
				    (UART_CFG_FLOW_CTRL_NONE)),                      \
		IF_ENABLED(DT_INST_NODE_HAS_PROP(n, dlf),                            \
			(.dlf = DT_INST_PROP_OR(n, dlf, 0),))			     \
		DEV_DATA_ASYNC(n)		
```

可以看到，baudrate属性是直接读取设备树上的current-speed属性，但是，uart节点中还包含`clock-frequency = <24000000>;`属性，该属性在驱动中的使用位置为：

```c
uart_ns16550_configure-->
set_baud_rate(dev, cfg->baudrate, pclk); // clock_frequency作为pclk参数传入set_baud_rate函数
```

这个属性的意义是什么呢？原来，设置uart的波特率，不是直接将波特率写入一个寄存器，而是配置两个除数寄存器 —— `DLL` 和 `DLM`。如何得知要写入的两个除数寄存器的值？

首先，UART 波特率由下式决定：
$$
\text{baud\_rate} = \frac{\text{pclk}}{16 \times (\text{DLM} \ll 8 + \text{DLL})}
$$
其中：

- `pclk`：UART 的输入时钟频率，来自外部时钟源，通常为 24 MHz、48 MHz 等；
- `16`：UART 内部固定的分频因子；
- `DLL` / `DLM`：波特率除数低/高字节，合起来是一个 16 位的除数。也就是说，本质上DLL和DLM是一个除数divisor的高位和低位。

如果我们知道波特率和时钟频率，那么可以反推得到divisor：

```c
static uint32_t get_uart_baudrate_divisor(const struct device *dev,
					  uint32_t baud_rate,
					  uint32_t pclk)
{
	ARG_UNUSED(dev);
	/*
	 * calculate baud rate divisor. a variant of
	 * (uint32_t)(pclk / (16.0 * baud_rate) + 0.5)
	 */
	return ((pclk + (baud_rate << 3)) / baud_rate) >> 4;
}
```

> 这里加0.5是为了四舍五入更精确

这样就可以根据divisor设置两个寄存器了：

```c
ns16550_outbyte(dev_cfg, BRDL(dev), (unsigned char)(divisor & 0xff));
ns16550_outbyte(dev_cfg, BRDH(dev), (unsigned char)((divisor >> 8) & 0xff));
```

通过rk3588 linux的驱动，可知divisor=0xd，因此可以反推得到pclk为24M，修改zephyr设备树uart节点中的current-speed=115200，clock-frequency=24000000，即可配置好正确的波特率，进行输出。

#### root linux的配置

当然，为了使用uart3，要在root linux里先配置好uart3的时钟。这个方法可以修改设备树和源码，让fiq-debugger捎带配一下。也可以等未来hvisor-tool实现了enable时钟的逻辑，通过hvisor-tool配置。

## zephyr运行

* 编译：

```
 west build --pristine -b roc_rk3588 samples/hello_world
```

* 在裸机上运行：

```
tftp 0x50000000 zephyr.bin; dcache flush; icache flush; dcache off; icache off; go 0x50000000
```

注意，这里必须要为uboot增加cache命令的支持，否则运行会报错。因为zephyr入口函数进入前，和linux一样，要求uboot关闭dcache和MMU。

* 作为non root运行

```
insmod hvisor.ko
./hvisor zone start zone1_zephyr.json
```

