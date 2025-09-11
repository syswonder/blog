# 在rk3588上通过hvisor启动64/32位zephyr
时间：2025/9/2

作者：李国玮

本文档主要介绍zephyr，以及如何将zephyr移植到rk3588上，并通过hvisor将zephyr作为non root zone启动。目前支持启动64位zephyr（包括多核），以及AArch32 Zephyr（目前只支持单核）。请关注本文档更新。

## RTOS的核心概念

RTOS强调任务的优先级，以及时间的准确性。因此要求：

1. 高优先级任务可以随时抢占低优先级任务的执行

当高优先级任务处于就绪状态时，比如某个设备发出中断告知资源就绪、高优先级任务等待的信号量被释放等等，此时系统会立刻调度高优任务，抢占低优任务的执行。注意，这里和时钟中断无关。

2. 与时钟相关的任务，时钟要求精确

一些任务比如周期性采样、定时睡眠等，这些依赖时钟中断的，则要求时钟必须高精度，以保证精确的时间，便于任务更好地在给定时间内完成实时任务。

## zephyr介绍

zephyr是一个RTOS，支持A核、M核。可参考：

Introduction：https://docs.zephyrproject.org/latest/introduction/index.html

zephyr支持的所有板子：https://docs.zephyrproject.org/latest/boards/index.html#

Beyond the start guide：https://docs.zephyrproject.org/latest/develop/beyond-GSG.html#beyond-gsg

### west

[west](https://github.com/zephyrproject-rtos/west)是一个用于管理多个仓库的超级版Git工具。它通过一个west.yml文件来进行仓库的初始化和更新。默认情况下该文件使用[west.yml](https://github.com/zephyrproject-rtos/zephyr/blob/main/west.yml)。

west官方文档：https://docs.zephyrproject.org/latest/develop/west/index.html

初始化的时候，需要两个命令：

* west init：clone upstream repository named in default west.yml. 也可以通过west init -l xxx直接指定zephyr路径
* west update：通过zephyr目录下的west.uml中指定的各个模块，来进行clone

### 板级支持

如何支持一个新板子：https://docs.zephyrproject.org/latest/hardware/porting/board_porting.html#board-porting-guide

RK3568：https://docs.zephyrproject.org/latest/boards/firefly/roc_rk3568_pc/doc/index.html

### kernel

#### 设备树解析

参考这里：https://www.cnblogs.com/jayant97/articles/17209392.html

Linux的DTS会被编译为DTB，然后在启动时由Bootloader传递给kernel。但Zephyr运行在性能较差的嵌入式平台上，故不可能专门运行一个解析器去读DTB。

 因此，DTS实际上实在编译时被Zephyr的构建系统（一套python脚本）变成了头文件，这个头文件的位置是：

```
${project_folder}/build/zephyr/include/generated/devicetree_generated.h
```

 了解即可，实际开发不需要查看这个头文件。

这个头文件生成的流程：在dts目录下，除了有设备树描述设备信息以外，还有一个bindings子目录，是各个设备节点的元数据，描述设备树节点的内容。在编译时，设备树中compatible和bindings中compatible相同节点的进行匹配，然后利用bindings将其转换为devicetree_generated.h。具体可以参考：[Devicetree HOWTOs — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/build/dts/howtos.html#dt-create-devices)

* C代码如何使用设备树

DeviceTree最终会用来生成`devicetree_generated.h`头文件，包含了DeviceTree中的所有信息。自然而然的，我们会想到要在C/C++代码中访问这些信息。

>  注意，由于DeviceTree中节点名称、属性名称允许使用的字符集是比C语言变量命名所允许的字符集更广泛的，因此，Zephyr规定，在C语言中访问DeviceTree的内容时，名称内的字母全部都变成**小写字母**、且特殊符号都变成**下划线**。
>
>  例如`zephyr,user`变为`zephyr_user`；`my-gpio`变为`my_gpio`。

 我们无需关心`devicetree_generated.h`文件本身的内容，因为它不是给人看的，需要使用一套宏函数来将其读出。在需要操作DeviceTree的文件中包含以下头文件：

```c
#include <zephyr/devicetree.h>
```

这里给出一个示例：

1. 在overlay文件中新增一个属性，表示自己需要一个GPIO进行测试，属性名称为`test-gpios`。这是一个gpio specifier。

```markdown
/{
    zephyr,user {
        test-gpios = <&gpio0 17 0>;
    };
};
```

1. 在`main.c`中，获取这个specifier，并操作GPIO

```c
#include <zephyr/drivers/gpio.h>

// 自己想要操作的节点的id，这里想要操作的节点是zephyr,user
#define NODE_ID DT_PATH(zephyr_user)

// 获取到zephyr,user节点的test-gpios属性，并把它作为gpio specifier，读入GPIO驱动。
static const struct gpio_dt_spec test_io = GPIO_DT_SPEC_GET(NODE_ID, test_gpios);

// 实际代码
int main()
{
    // 判断设备（这里是gpio控制器）是否已初始化完毕
    // 一般情况下，在application运行前，zephyr驱动就已经把控制器初始化好了
    if (!device_is_ready(test_io.port)) {
        return;
    }
    
    // 重新配置IO
    // 如果DeviceTree里写好了，这里也可以不配
    gpio_pin_configure_dt(&test_io, GPIO_OUTPUT_INACTIVE);
    
    
    // 操作IO
    gpio_pin_set_dt(&test_io,1);
    gpio_pin_set_dt(&test_io,0);
    
    return 0;
}
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

* 数据结构

`struct z_kernel _kernel`代表整个zephyr内核，用`struct k_thread`来表示线程控制块。`struct _ready_q`为调度队列。

内核主线程为`struct k_thread z_main_thread`，其入口地址为bg_thread_main函数。

##### main thread

内核的主线程，用于初始化，并跳转到应用程序的main函数。优先级默认最高。

#### interrupt

注册中断的接口函数：

[`IRQ_CONNECT`](https://docs.zephyrproject.org/latest/doxygen/html/group__isr__apis.html#ga131739d1faf501a15590053817aba984)：注册中断处理函数，如果需要在运行时确定则使用：[`irq_connect_dynamic()`](https://docs.zephyrproject.org/latest/doxygen/html/group__isr__apis.html#ga4e9915b92b09df49b99bc449f0cc31a1)

zephyr支持中断嵌套。

#### memory management

如果开启CONFIG_MMU，则自动开启虚拟内存机制。虚拟内存的起始地址KERNEL_VM_BASE，和设备树中指定的zephyr,sram的起始地址一致。Image的text和data，和物理内存一般是恒等映射。

所以sram可以视为和dram一样的东西。linker.ld指定了text段的起始地址，等于物理内存的起始地址。

zephyr中一些可以设置的config：

[`CONFIG_KERNEL_VM_BASE`](https://docs.zephyrproject.org/latest/kconfig.html#CONFIG_KERNEL_VM_BASE)：等于设备树中指定的内存起始地址

[`CONFIG_KERNEL_VM_SIZE`](https://docs.zephyrproject.org/latest/kconfig.html#CONFIG_KERNEL_VM_SIZE)：虚拟地址空间的大小默认为0x800000，我们没有设定，目前是默认的。

[`CONFIG_KERNEL_VM_OFFSET`](https://docs.zephyrproject.org/latest/kconfig.html#CONFIG_KERNEL_VM_OFFSET)：image离VM的偏移，默认为0，我们也是0.

[`CONFIG_KERNEL_DIRECT_MAP`](https://docs.zephyrproject.org/latest/kconfig.html#CONFIG_KERNEL_DIRECT_MAP)：我们没有设置。驱动MMIO映射页表时会映射到上面的虚拟地址空间。

虚拟地址空间的排布：

```
+--------------+ <- K_MEM_VIRT_RAM_START
| Undefined VM | <- architecture specific reserved area
+--------------+ <- K_MEM_KERNEL_VIRT_START
| Mapping for  |
| main kernel  |
| image        |
|              |
|              |
+--------------+ <- K_MEM_VM_FREE_START
|              |
| Unused,      |
| Available VM |
|              |
|..............| <- grows downward as more mappings are made
| Mapping      |
+--------------+
| Mapping      |
+--------------+
| ...          |
+--------------+
| Mapping      |
+--------------+ <- memory mappings start here
| Reserved     | <- special purpose virtual page(s) of size K_MEM_VM_RESERVED
+--------------+ <- K_MEM_VIRT_RAM_END
```

虚拟地址空间排布图中各个宏定义如下（来自zephyr文档）：

- `K_MEM_VIRT_RAM_START` is the beginning of the virtual memory address space. This needs to be page aligned. Currently, it is the same as [`CONFIG_KERNEL_VM_BASE`](https://docs.zephyrproject.org/latest/kconfig.html#CONFIG_KERNEL_VM_BASE).
- `K_MEM_VIRT_RAM_SIZE` is the size of the virtual memory address space. This needs to be page aligned. Currently, it is the same as [`CONFIG_KERNEL_VM_SIZE`](https://docs.zephyrproject.org/latest/kconfig.html#CONFIG_KERNEL_VM_SIZE).
- `K_MEM_VIRT_RAM_END` is simply (`K_MEM_VIRT_RAM_START` + `K_MEM_VIRT_RAM_SIZE`).
- `K_MEM_KERNEL_VIRT_START` is the same as `z_mapped_start` specified in the linker script. This is the virtual address of the beginning of the kernel image at boot time. 和K_MEM_VIRT_RAM_START一样。
- `K_MEM_KERNEL_VIRT_END` is the same as `z_mapped_end` specified in the linker script. This is the virtual address of the end of the kernel image at boot time.  image运行时占用的大小加上image的起始地址
- `K_MEM_VM_FREE_START` is the same `K_MEM_KERNEL_VIRT_END` which is the end of the kernel image. It is the beginning of the virtual address space where addresses can be allocated for memory mapping. 
- `K_MEM_VM_RESERVED` is an area reserved to support kernel functions. For example, some addresses are reserved to support demand paging。在mmu.c文件中，如果没有开启demand paging，则大小为0。我们没有开启，因此大小为0

在代码中，这些宏的来源如下：

```c

/** Start address of physical memory. */
#define K_MEM_PHYS_RAM_START	((uintptr_t)CONFIG_SRAM_BASE_ADDRESS)

/** Size of physical memory. */
#define K_MEM_PHYS_RAM_SIZE	(KB(CONFIG_SRAM_SIZE))

/** End address (exclusive) of physical memory. */
#define K_MEM_PHYS_RAM_END	(K_MEM_PHYS_RAM_START + K_MEM_PHYS_RAM_SIZE)

/** Start address of virtual memory. */
#define K_MEM_VIRT_RAM_START	((uint8_t *)CONFIG_KERNEL_VM_BASE)

/** Size of virtual memory. */
#define K_MEM_VIRT_RAM_SIZE	((size_t)CONFIG_KERNEL_VM_SIZE)

/** End address (exclusive) of virtual memory. */
#define K_MEM_VIRT_RAM_END	(K_MEM_VIRT_RAM_START + K_MEM_VIRT_RAM_SIZE)

/** Boot-time virtual start address of the kernel image. */
#define K_MEM_KERNEL_VIRT_START	((uint8_t *)&z_mapped_start[0])

/** Boot-time virtual end address of the kernel image. */
#define K_MEM_KERNEL_VIRT_END	((uint8_t *)&z_mapped_end[0])

/** Boot-time virtual address space size of the kernel image. */
#define K_MEM_KERNEL_VIRT_SIZE	(K_MEM_KERNEL_VIRT_END - K_MEM_KERNEL_VIRT_START)

/**
 * @brief Offset for translating between static physical and virtual addresses.
 *
 * @note Do not use directly unless you know exactly what you are going.
 */
#define K_MEM_VM_OFFSET	\
	((CONFIG_KERNEL_VM_BASE + CONFIG_KERNEL_VM_OFFSET) - \
	 (CONFIG_SRAM_BASE_ADDRESS + CONFIG_SRAM_OFFSET))

```



#### device model

https://docs.zephyrproject.org/latest/kernel/drivers/index.html

* uart

在.defconfig中，定义了CONFIG_UART_NS16550。在cmakelists.txt中，由于定义了CONFIG_UART_NS16550，会将uart_ns16550.c加入到源代码。RK3588 SoC 使用的 UART 控制器与 NS16550 兼容。

## aarch64 zephyr移植到rk3588

移植板子的整体思路：

1. 考虑hvisor是否需要修改，以启动zephyr虚拟机
2. 为了调试方便，可以在zephyr中用汇编向root linux的串口写数据，来查看zephyr运行到哪里
3. 一步步解决卡住的问题即可。

由于zephyr支持rk3568，而3568和3588比较相似，因此参考3568的板级支持以及Soc支持，即可移植到3588上。这里给出移植过程中遇到的一些难点以及解决方案。

### 对hvisor的检查

移植中遇到了一个诡异的bug，有时候能启动，有时候启动不了。最初怀疑是hvisor有关icache、dcache的问题，但是我查看linux启动汇编代码（位于head.S），head.S中一段注释描述了Linux内核关于入口函数的约定：

```
/*
 * Kernel startup entry point.
 * ---------------------------
 *
 * The requirements are:
 *   MMU = off, D-cache = off, I-cache = on or off,
 *   x0 = physical address to the FDT blob.
 *
 * This code is mostly position independent so you call this at
 * __pa(PAGE_OFFSET).
 *
 * Note that the callee-saved registers are used for storing variables
 * that are useful before the MMU is enabled. The allocations are described
 * in the entry routines.
 */
```

1. 引导程序必须在启动前通过SCTLR_EL1关闭MMU。因为还没有建立Stage 1页表，所以hvisor不需要刷新TLB。
2. 启动前必须关闭D-cache。
3. 启动前可以开启或关闭I-cache。因为引导程序的代码段和linux代码段必不重合。不过如果Linux作为虚拟机运行，那么hvisor还是有必要清理一下I-cache，因为一个CPU上可能会启动多个虚拟机。

而hvisor在进入虚拟机前的确满足了这些要求，包括关闭icache、dcache、MMU。

最终发现这个问题在于，hvisor-tool通过/dev/mem访问内存时，不会访问cache；而通过/dev/hvisor访问内存时，会访问cache。把加载镜像的函数从使用/dev/hvisor改为/dev/mem后，原zephyr就可以一直正常运行。

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

## aarch32 zephyr移植到rk3588

接下来介绍，如何将zephyr以aarch32位模式，在hvisor启动起来。首先介绍一些预备知识：

### 预备知识

#### 32位意味着什么

在计算机领域，32位和64位（通常指处理器架构和操作系统）的本质区别主要体现在以下几个方面：

1.  **内存寻址能力 (Memory Addressing):** 这是最核心的区别。
    - **32位系统：** 使用32位的内存地址，理论上可以寻址的最大内存空间是 232 字节，即 4 GB。这意味着即使计算机安装了超过 4 GB 的物理内存，32位操作系统和应用程序也通常只能访问其中的约 4 GB（实际可用内存可能因硬件和其他因素而略低于 4 GB）。
    - **64位系统：** 使用64位的内存地址，理论上可以寻址的最大内存空间是 264 字节，这是一个极其庞大的数字（大约 18EB，即 1800万 TB）。虽然实际可用的内存受到硬件和操作系统的限制，但64位系统可以轻松支持远超 4 GB 的内存，这对于运行内存密集型应用程序（如视频编辑、大型数据库、虚拟机和大型游戏）至关重要。
2.  **处理器寄存器大小 (Processor Register Size):**
    - **32位处理器：** 主要使用32位的寄存器来存储数据和内存地址。
    - **64位处理器：** 使用64位的寄存器。更大的寄存器意味着处理器一次可以处理更多的数据，从而在某些计算任务中提供更高的效率和性能。
3.  **数据处理能力 (Data Handling):**
    - **32位系统：** 处理器一次处理32位的数据。
    - **64位系统：** 处理器一次处理64位的数据。这种更宽的数据处理能力可以加快处理大量数据的速度，尤其是在科学计算、图形渲染和数据压缩等领域。

#### armv8与armv7的区别

区别有很多，但对于我们要移植的话，最主要的区别是：

**指令集架构 (Instruction Set Architecture - ISA):**

- **ARMv7:** 主要支持 32 位的 ARM 指令集 (A32) 和 Thumb 指令集 (T32)。它是一个纯 32 位架构。
- **ARMv8:** 引入了 AArch64 指令集，这是一个全新的 64 位指令集。同时，**ARMv8 保留了对 32 位指令集的兼容，称为 AArch32，它包含了 ARMv7 的 A32 和 T32 指令集**，并加入了一些新的指令。这意味着 ARMv8 处理器可以在 64 位模式 (AArch64) 或 32 位模式 (AArch32) 下运行。

也就是说，让zephyr以32位（aarch32）的模式跑起来，其实本质上就是跑armv7架构下的zephyr。zephyr仓库里有对armv7架构板子的大量支持，我们参考其中一些包括CYCLONEV、STM32MP13X即可。而且zephyr本身支持A7和A9核，这些都是armv7架构下的CPU。因此一种移植的办法，就是按照它们这个，使用A7或A9的CPU核进行编译，尝试运行。至于编译器的选择，是west自动做的。我们只需要写好west编译命令就好了！

#### aarch32和aarch64寄存器的对应关系

其实我很好奇，为什么一个armv8的CPU，可以跑32位的程序。其实背后的原理很简单（不是），把64位寄存器砍掉高32位不就成了32位了吗！

> [ARM AArch32和AArch64通用寄存器、状态寄存器-CSDN博客](https://blog.csdn.net/JaCenz/article/details/127614765)：首先需要看看armv7下的CPU mode，armv7下EL1会有额外的mode，比armv8略微复杂些。先做个了解吧！

对于armv7来说，各CPU mode下寄存器banking的结果如下。所谓banking，是指一个名字相同的寄存器，例如R8，在**物理上**存在多个寄存器，例如R8、R8_fiq，它们分别用于不同mode：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/2024/image-20250703092210337.png" alt="image-20250703092210337" style="zoom:50%;" />

而aarch64和aarch32寄存器的映射关系则如下（https://armv8-doc.readthedocs.io/en/latest/04.html）：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/2024/image-20250703092330958.png" alt="image-20250703092330958" style="zoom:50%;" />

这里的映射关系，就是我刚才说的把64位寄存器的高32位砍掉，只用低32位。暂且可以这么理解吧！比如W0，就是armv8 cpu X0寄存器的低32位，aarch32模式下也是使用这个寄存器，不过是当成了R0.

### 开始移植

如果想直接看移植的代码，可以看这几个PR：

1. 64位zephyr移植到裸机RK3588：[board: arm64: Add AArch32 virtual machine support for ROC-RK3588-PC by KouweiLee · Pull Request #95311 · zephyrproject-rtos/zephyr](https://github.com/zephyrproject-rtos/zephyr/pull/95311)
2. 32位zephyr的移植：[board: arm64: Add AArch32 virtual machine support for ROC-RK3588-PC by KouweiLee · Pull Request #95311 · zephyrproject-rtos/zephyr](https://github.com/zephyrproject-rtos/zephyr/pull/95311)

而且我移植过程中主要参考的是zephyr/boards/arm/fvp_baser_aemv8r，包含了aarch64和aarch32两种配置。zephyr/soc/arm/fvp_aemv8r中则包含了相关的SOC配置。

#### hvisor的相关改动

hvisor要启动aarch32虚拟机，需要修改两个寄存器：HCR_EL2和SPSR_EL2，以允许EL1运行AArch32模式。此外，zephyr运行过程中会访问GIC，抑或发生硬件中断，这时会陷入hvisor，hvisor的异常向量表和异常处理函数做了新增和改动，具体修改的部分可见PR：[Add aarch32-guest mode support by KouweiLee · Pull Request #187 · syswonder/hvisor](https://github.com/syswonder/hvisor/pull/187)。

一些我思考过的问题：

1. 为什么zephyr陷入EL2时，不需要在EL2保存EL1的上下文。

可以看aarch64和aarch32那张图，aarch32的通用寄存器，都直接映射在aarch64的W寄存器了。

#### MMU enable后直接死掉

我遇到的第一个问题，是enable MMU后直接程序死了，不往下运行了，我最初是怀疑没清理cache。但后来发现，加上清理cache的函数也不行。

> cache的三种操作方式：
>
> clean:
>      检查对应内存cache line 的dirty bit。如果dirty bit为1，将cache line的内容写回下一级存储，并将dirty bit置为0.
>
> invalid:
>      检查对应内存cache line 的valid bit.如果valid bit 为1，置为0.
>
> flush:
>      每条cache line 先clean，再invalid.

后来发现，为了调试方便，我让zephyr通过Root linux uart2输出字符。但我之前忘记把uart2的地址映射到页表。zephyr最初构建页表，会首先映射代码段、数据段，然后根据mmu_region.c文件中指定的内存区域进行映射。因此将uart2加入到mmu_region.c在页表构建时映射，就可以了。

#### GIC初始化被卡住

遇到的第二个比较严重的问题是，当zephyr初始化GIC-v3时，有个地方就被卡住了，最终定位到是在arm_gic_iterate_rdists这个函数：

```c
uint64_t aff = arm_gic_mpidr_to_affinity(GET_MPIDR()); // 读MPIDR寄存器，获取CPU编号：(aff2 << 16 | aff1 << 8 | aff0)，注意，aarch32下的MPIDR寄存器不包含aff3字段，

val = arm_gic_get_typer(rdist_addr + GICR_TYPER); // 读typer寄存器，获取redistributor对应的CPU编号
uint64_t gicr_aff = GICR_TYPER_AFFINITY_VALUE_GET(val);

if (arm_gic_aff_matching(gicr_aff, aff)) {
    return rdist_addr;
}
```

zephyr读typer时，aarch32会分两次读，而原来的hvisor逻辑只支持对addr读， addr+4地址读会被忽略，因此修改hvisor逻辑即可：

```c
static inline uint64_t arm_gic_get_typer(mem_addr_t addr)
{
	uint64_t val;

#if defined(CONFIG_ARM)
	val = sys_read32(addr);
	val |= (uint64_t)sys_read32(addr + 4) << 32;
#else
	val = sys_read64(addr);
#endif

	return val;
}
```

此外由于aarch32下MPIDR没有aff3字段，还需要修改arm_gic_aff_matching，只匹配aff2到aff0。

#### zephyr内核发生异常无法进入zephyr异常向量表

这还是地址映射的问题。armv7在以前不支持虚拟化拓展的版本中，是不支持vbar寄存器（就是armv8用来设置异常向量表地址的寄存器）的，只支持通过SCTLR 来设置中断向量表的地址：

> 在 armv7 中，中断向量表可以设置在两个地址：0x00000000 和 0xffff0000，由协处理器 cp15 的 SCTLR 的 bit13 来控制，默认情况下，中断向量表的位置在 0x00000000，实际上，对于操作系统而言，比如 linux，会更倾向于将中断向量表放在 0xffff0000 处，因为 0x0 处在用户空间下，需要额外做一些限制，而 0xffff0000 处在内核空间，另一方面，0 地址通常是 NULL 指针访问的地址，这需要 MMU 做相应的访问限制规则，总之，将向量表放在高地址处会更方便。[https://zhuanlan.zhihu.com/p/362717490]

但是对于armv8的aarch32却不同，在armv8寄存器手册中，有如下内容，表明可以设置vbar的值：

<img src="https://zmhqcp62hu.feishu.cn/space/api/box/stream/download/asynccode/?code=MzYyNjMzNjNkMTcwMDJhZTI3N2Q4Y2NjNTkxODM5MWVfY1pwRnhQdmQ3WThqMHpYV2dDQnNXM0s5bGFPc2F6eDdfVG9rZW46V0JIemJDM0M4b2tjSEd4V2tIQmNZZ1NUbnVoXzE3NTc1NTA5Nzk6MTc1NzU1NDU3OV9WNA" alt="img" style="zoom: 50%;" />

但要设置的前提是，需要将SCTLR寄存器的HIVECS清0，表示异常向量表地址使用vbar来注册。不过zephyr没有默认提供这种方式，zephyr直接在relocate_vector_table函数中将异常向量表注册为0地址。不过该函数采用了`__weak`符号，表示如果有其他同名函数被定义，则会采用其他函数来代替这个默认函数。因此我在rk3588的soc.c文件下加入了如下内容：

```c
#define VECTOR_ADDRESS ((uintptr_t)_vector_start)

void relocate_vector_table(void)
{
	write_sctlr(read_sctlr() & ~HIVECS);
	write_vbar(VECTOR_ADDRESS & VBAR_MASK);
	barrier_isync_fence_full();
}
```

#### uart3无法输出

都配置好后，我发现任何向uart3 MMIO地址读写的操作，都会触发data abort。uart3会被zephyr映射到虚拟地址空间，我之前想不到为什么会出现data abort异常。后来我猜想可能是将uart3映射到虚拟地址空间时，修改页表的操作只修改了dcache中的页表，而MMU读页表时是直接从内存里读。然后我在页表映射的函数中加入了清理cache的函数：

```C++
void arch_mem_map(void *virt, uintptr_t phys, size_t size, uint32_t flags)
{
    int ret = __arch_mem_map(virt, phys, size, flags);

    if (ret) {
        LOG_ERR("__arch_mem_map() returned %d", ret);
        k_panic();
    } else {
        sys_cache_data_flush_all(); // 新加的
        invalidate_tlb_all();
    }
}
```

然后竟然可以工作了。但是后来查阅ARM MMU手册时发现，页表在设置TTBR0时，已经通过标志位告诉MMU，页表的缓存策略是什么。因此MMU会直接从dcache中读取最新的页表。所以很奇怪，不知道是怎么使它工作起来的，不过我猜想，可能是rk3588的硬件bug。

#### timer只能发一次中断

aarch32/64 zephyr在裸机上运行没问题，但是hvisor上，timer只能发一次中断。主要是在测试synchronization时遇到的问题。我查看zephyr代码，对timer的下一时刻触发时钟中断，是没有错误的。

最终发现，Arm gicv3 中，hvisor设置ICH_VMCR_EL2的VEOIM为1：

<img src="https://zmhqcp62hu.feishu.cn/space/api/box/stream/download/asynccode/?code=MDYwYWQwMWYzODkyZWY0OWY2ODk4OGNjMGI1NTJiZDlfdTYzRTZMbFBnejhRZ29hNERxOXZWaGJ4UU51cjN1WklfVG9rZW46SkJONWJIZ05Sb2wwcHJ4ZVhPdGNwRlVlbmplXzE3NTc1NTMyMDc6MTc1NzU1NjgwN19WNA" alt="img" style="zoom:50%;" />

这就要求虚拟机要deactivate一个中断，必须还要写ICV_DIR_EL1寄存器。但zephyr以裸机的思维运行，只写EOIR0寄存器。我不知道之前为什么我们把VEOIM设为1。当我修改完后，zephyr可以正常运行了。

* 那么直接的一个问题出现了，为什么linux在VEOIM为1为0时都可以运行？解释如下：

直接一句话回答，linux会自己根据自己的启动模式，来设置自己的EOImode。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/2024/asynccode" alt="img" style="zoom:50%;" />

linux在初始化gic时，会检查：如果CPU是从非EL2模式下启动的，比如hvisor启动linux直接在EL1上启动，那么会设置supports_deactivate_key为false。如果是在EL2模式，表示该linux裸机从uboot上启动，后续可能会使用kvm虚拟化，会设置supports_deactivate_key为true。

supports_deactivate_key的作用则是，在gic_irq_domain_map函数中，会根据supports_deactivate_key的不同，会设置不同的gic chip，不同gic chip在清中断时做法不同，分别对应EOImode为0和为1的情况。

而在函数gic_cpu_sys_reg_init，会根据supports_deactivate_key来设置cpu 寄存器ICC_CTLR_EL1.EOImode：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/2024/image-20250911091944101.png" alt="image-20250911091944101" style="zoom:50%;" />

在虚拟化场景下，ICC_CTLR_EL1会被映射为ICV_CTLR_EL1，通过该寄存器可以修改虚拟机对中断的处理是传统模式还是分离模式。arm手册中提到ICV_CTLR_EL1.EOImode和ICH_VMCR_EL2.EOImode是一致的，因此，无论hvisor对虚拟机设置的ICH_VMCR_EL2.EOImode是什么，之后都会被linux修改，对于linux虚拟机，linux则会采用传统模式，只写EOIR寄存器即可完成priority_drop + deactivate irq。

## 给zephyr官方仓库提交pr的tips

官方仓库专门有[说明](https://docs.zephyrproject.org/latest/contribute/guidelines.html#contribute-guidelines)，为提交pr提供了一些指导和强制性建议。以下是我提交pr的用到的一些tips：

* 板权声明

推荐用这个：

```
# Copyright (c) 2025 Syswonder
# SPDX-License-Identifier: Apache-2.0
```

* commit msg要求最后加上Signed-off-by：xxx的方法

直接用git commit -s来提交，通过来设置name和email：

```bash
git config user.name
git config user.email
```

* 本地检查CI是否通过

```
./scripts/ci/check_compliance.py -c upstream/main..
```

* 写文档

写文档的教程：[Documentation Guidelines — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/contribute/documentation/guidelines.html#doc-guidelines)

本地编译和查看文档，在docs下执行：

```
make html HW_FEATURES_VENDOR_FILTER=firefly SPHINXOPTS="-j auto --keep-going -T"
python3 -m http.server -d _build/html --bind 127.0.0.1
```



## zephyr的调试

### 向root linux串口写数据

由于jtag使用过于艰难，这里有一种很好的调试zephyr运行到哪里的办法：直接向root linux串口写数据

```
static void send_char(char c) {
    register uint32_t addr = 0xfeb50000;
    __asm__ volatile (
        "strb %1, [%0]\n\t"
        :
        : "r"(addr), "r"(c)
        : "memory"
    );
}
```

arm汇编代码则为：

```asm
# for arm
push    {r0, r1}
ldr     r0, =0xfeb50000 
mov     r1, #'S'
strb    r1, [r0]
pop     {r0, r1}
```

### zephyr提供的调试方法

1. 查看当前编译下，启用的config有哪些：

   ```shell
   cat build/zephyr/.config | grep CONFIG_ARM_AARCH32_MMU
   ```

2. `zephyr.map` 是 Zephyr 构建系统在编译链接阶段生成的 **链接映射文件**（Linker Map File），它详细记录了最终可执行文件（`zephyr.elf`）中所有符号、段（section）、地址的排布情况，是**调试启动流程、分析段布局、查看符号地址的关键工具**。

## zephyr运行

拉取这个分支：[KouweiLee/zephyr at add_roc_rk3588_aarch32](https://github.com/KouweiLee/zephyr/tree/add_roc_rk3588_aarch32)。待PR均合并后，可以拉取官方仓库。目前需拉取这个。

### 利用clangd来支持函数跳转

用cursor/vscode看zephyr代码不用clangd没法看。所以在编译前：

首先执行：

```
west config build.cmake-args -- -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

west build 在执行真正的 `cmake` 之前，会把 `build.cmake-args` 里保存的字符串按 shell 语法拆成若干 token，再追加到最终 `cmake` 命令行后面。以后每次 `west build` 实际跑的命令会变成：`cmake … -DCMAKE_EXPORT_COMPILE_COMMANDS=ON …`。该arg会使cmake编译时自动生成compile_commands.json。

再执行编译命令，然后在.vscode/.settings.json中加入：

```json
    "clangd.arguments": [
        "--background-index",
        "--compile-commands-dir=${workspaceFolder}/zephyr/build",
        "--completion-style=detailed",
        "--header-insertion=iwyu",
        "--pch-storage=memory"
    ]
```

就可以通过clangd来索引文件了。

### 编译+运行aarch64 zephyr

例如，编译4核smp下的zephyr，应用为samples/synchronization。

```
west build --pristine -b roc_rk3588_pc/rk3588_aarch64/smp samples/synchronization
```

如果只编译单核，则把/smp去掉即可

* 在裸机上运行：

```
tftp 0x50000000 zephyr.bin; dcache flush; icache flush; dcache off; icache off; go 0x50000000
```

注意，这里必须要为uboot增加cache命令的支持，否则运行会报错，具体而言，要在uboot源码中的rk3588_common.h中，加入`#define CONFIG_CMD_CACHE`。因为zephyr入口函数进入前，和linux一样，要求uboot关闭dcache和MMU。

* 作为non root运行

```
insmod hvisor.ko
./hvisor zone start zone1_zephyr.json
```

zone1_zephyr.json可以查看这个示例：

```json
{
    "arch": "arm64",
    "name": "zephyr",
    "zone_id": 1,
    "cpus": [2],
    "memory_regions": [
        {
            "type": "ram",
            "physical_start": "0x50000000",
            "virtual_start":  "0x50000000",
            "size": "0x25000000"
        },
        {
            "type": "io",
            "physical_start": "0xfeb50000",
            "virtual_start":  "0xfeb50000",
            "size": "0x20000"
        }
    ],
    "interrupts": [334],
    "ivc_configs": [],
    "kernel_filepath": "./zephyr.bin",
    "dtb_filepath": "null",
    "kernel_load_paddr": "0x50000000",
    "dtb_load_paddr":   "0x60000000",
    "entry_point":      "0x50000000",
    "kernel_args": "",
    "arch_config": {
        "gic_version": "v3",
        "gicd_base": "0xfe600000",
        "gicd_size": "0x10000",
        "gicr_base": "0xfe680000",
        "gicr_size": "0x100000",
        "gits_base": "0x0",
        "gits_size": "0x0",
        "is_aarch32": false
    }
}
```

### 编译+运行aarch32 zephyr

例如，编译单核zephyr，应用为samples/synchronization。

```
west build --pristine -b roc_rk3588_pc/rk3588_aarch32 samples/synchronization
```

* 作为non root运行

```
insmod hvisor.ko
./hvisor zone start zone1_zephyr.json
```

zone1_zephyr.json可以查看这个示例：

```json
{
    "arch": "arm",
    "name": "zephyr",
    "zone_id": 1,
    "cpus": [2],
    "memory_regions": [
        {
            "type": "ram",
            "physical_start": "0x50000000",
            "virtual_start":  "0x50000000",
            "size": "0x25000000"
        },
        {
            "type": "io",
            "physical_start": "0xfeb50000",
            "virtual_start":  "0xfeb50000",
            "size": "0x20000"
        }
    ],
    "interrupts": [334],
    "ivc_configs": [],
    "kernel_filepath": "./zephyr.bin",
    "dtb_filepath": "null",
    "kernel_load_paddr": "0x50000000",
    "dtb_load_paddr":   "0x60000000",
    "entry_point":      "0x50000000",
    "kernel_args": "",
    "arch_config": {
        "gic_version": "v3",
        "gicd_base": "0xfe600000",
        "gicd_size": "0x10000",
        "gicr_base": "0xfe680000",
        "gicr_size": "0x100000",
        "gits_base": "0x0",
        "gits_size": "0x0",
        "is_aarch32": true
    }
}
```
