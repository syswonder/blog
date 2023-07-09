# Bao Docs

时间：2023.7.9

作者：陈林锟

## 概述

针对嵌入式平台的静态分区Hypervisor有Jailhouse和Bao。与Jailhouse不同，Bao是Type-1 Hypervisor，意味着它不需要依赖其他的操作系统来启动。目前Bao支持ARM-v8和RISC-V架构。

下面对Bao的分析均针对ARMv8-a。

## 如何运行Bao？

获取项目源码：

```bash
git https://github.com/bao-project/bao-demos
```

接下来按照 `README.md` 进行操作即可。注意过程中会下载Linux内核、QEMU、U-Boot等项目的源码，请务必留出足够的磁盘空间！

## Bao的代码结构

按照上述流程操作之后，Bao-demos项目中将出现项目wrkdir/srcs/bao，这就是Bao的核心代码。

分析Bao的启动流程（以ARMv8-a 64位，QEMU上运行为例）：

### 配置文件

- 虚拟机配置文件在bao-demos中: `$DEMO/configs/$PLATFORM.c`中。

- 硬件平台配置文件在`platform/$PLATFORM/xxxx_desc.c`中。

### Boot阶段：CPU线性化、master设置、禁用cache等

根据src/linker.ld中的规则，入口点在`_image_start`符号处，或者说程序是从`.boot`段开始的。

```asm
ENTRY(_image_start)
```

从u-boot出来之后，运行的第一行代码位于src/arch/armv8/aarch64/boot.S中：

```asm
 .section ".boot", "ax"
.globl _reset_handler
.globl _el2_entry
_el2_entry:
_reset_handler:

	mrs  x0, MPIDR_EL1
	adrp x1, _image_start
```

以上代码是在EL2态下运行的。将x0设为MPIDR_EL1（处理器ID），x1设为hypervisor镜像的起始地址。在qemu-aarch64-virt中镜像被装载到0x50000000处，所以x1是0x50000000。

```asm
	adr	x3, _hyp_vector_table
	msr	VBAR_EL2, x3
```

这段用于装载EL2的异常向量表。

```asm
	mov x3, x0, lsr #8 
	and x3, x3, 0xff
	adr x4, platform
	ldr x4, [x4, PLAT_ARCH_OFF+PLAT_ARCH_CLUSTERS_OFF+PLAT_CLUSTERS_CORES_NUM_OFF]
	ldr x5, =BAO_VAS_BASE
	sub x4, x4, x5
	add x4, x4, x1
	mov x5, xzr
	mov x7, xzr
1:
	cmp x5, x3
	b.ge	2f
	ldrb w6, [x4]
	add x4, x4, #1
	add x5, x5, #1
	add x7, x7, x6
	b 	1b
2:
	and x0, x0, #0xff
	add x0, x0, x7
```
这段用于CPU的线性化，原先x0保存的是MPIDR_EL1（由aff2, aff1, aff0构成），现在根据platform结构体提供的信息将MPIDR_EL1映射为线性的cpu id覆盖原有的x0，便于后续处理。

```asm
.pushsection .data
_master_set:
	.8byte 	0
.popsection
/**
 * Setting CPU_MASTER by lottery, the first cpu to atomically set _master_set,
 * becomes the cpu master. As a result x9 should is set to !is_cpu_master.
 */
    mov x5, #1
    adr	x3, _master_set
_set_master_cpu:
	ldaxr w9, [x3]
	cbnz w9, 1f
	stlxr w9, w5, [x3]
	cbnz w9, _set_master_cpu
	adr x3, CPU_MASTER
	str x0, [x3]
1: 
```
这段代码用于竞争master，_master_set这个地址的变量=1表示该cpu为master。

之后：需要将系统带到一个良好的状态。包括禁用MMU（已完成），所有缓存（指的是除了指令缓存外还有其他缓存），以及分支预测（BP）等等，并对它们进行无效化处理。

`ic iallu`：将指令缓存（Instruction Cache）中的所有条目无效化。

接下来跳转到`boot_arch_profile_init`执行适用于armv8-a的boot代码。

```
mrs x3, SCTLR_EL2 
bic x3, x3, #0x7	
msr SCTLR_EL2, x3
```

以上代码禁用SCTLR（System Control Register）的cache和MMU。

### Boot阶段：Hypervisor全局页表的建立

boot_arch_profile_init完成了Hypervisor全局页表的建立。

Hypervisor的全局页表不使用传统的恒等映射。bao的每个CPU拥有独立的根页表，采用4KB大小的页面和8B大小的页表项，即每个页面含有 512个页表项。通过利用根页表的512个页表项对每个CPU的虚拟地址空间进行划分。

1. PTE Index=0~506，对应的地址空间为[0x0, 0xfd8000000000)：这部分根页表未进行映射，访问这些地址会产生缺页错误。
2. PTE Index=507，对应的地址空间为[0xfd8000000000, 0xfe0000000000)：表示全局虚拟地址空间，用于存储所有CPU共享的数据，映射的物理地址空间包括hypervisor镜像和虚拟机镜像。这个空间是共享的，即所有CPU的index=507的页表项都指向同一个次级页表。
3. PTE Index=508，对应的地址空间为[0xfe0000000000, 0xfe8000000000)：每个虚拟机的虚拟地址空间。这个空间不是全局共享的，而是同属于一个VM的CPU所共享的。其中包含了虚拟机所需的数据，例如 VMCB、VCPU结构体、Stage-2页表等。
4. PTE index=509，对应的地址空间为[0xfe8000000000, 0xff0000000000)：每个CPU的私有地址空间，可以通过这个空间访问一些 CPU 私有数据，例如为每个CPU单独设置的配置结构体`struct cpu`，如下所示。

```
struct cpu {
    cpuid_t id;

    bool handling_msgs;
    
    struct addr_space as;

    struct vcpu* vcpu;

    struct cpu_arch arch;

    struct cpuif* interface;

    uint8_t stack[STACK_SIZE] __attribute__((aligned(PAGE_SIZE)));
    
} __attribute__((aligned(PAGE_SIZE)));
```

5. 最后，index=510 和 index=511的页表项分别指向 Hypervisor 全局根页表本身以及 VM 的 Stage-2 根页表的递归页表项，用于页表层级结构中对某个页表项的快速寻址（用虚拟地址寻找页表项）。

### 初始化阶段：cpu_init

1. 初始化struct cpu

每个cpu都有一个0xfe0000000000地址，这个地址保存cpu结构体，初始化之后可以直接用cpu()访问。

2. master唤醒其余cpu
PSCI（Power State Coordination Interface）是由ARM官方定义的接口，用于协调系统中不同处理器核心的电源管理状态。通过PSCI接口，系统软件可以在运行时控制处理器核心的启动、暂停、重启等操作，以降低功耗、提高性能，并确保系统稳定性。

PSCI接口包含一系列函数，如psci_version()、psci_cpu_off()、psci_cpu_on()、psci_system_off()、psci_system_reset()等，这些函数可由处理器固件和操作系统内核在必要时调用，实现CPU核心的启动、关闭和电源管理等功能。其中，需要调用psci_cpu_on()函数来唤醒目标CPU核心。

具体而言，psci_cpu_on()函数的作用是将目标CPU的MPDIR（多处理器目录表）、Hypervisor的入口地址、上下文ID等参数传递给寄存器x0、x1、x2和x3，并通过调用SMC（Secure Monitor Call）指令以EL3（Exception Level 3）模式进入ATF，请求唤醒目标CPU核心。随后，所有CPU核心将能够进入Hypervisor的初始化阶段，为后续流程做好准备。

等待其余核心均完成cpu_init之后才能继续。

### 初始化阶段：mem_init

此阶段用于初始化内存。

1. mem_prot_init：初始化每个cpu的as（addrspace）结构体，并在根页表中映射递归页表项。

2. 只有master cpu执行：mem_setup_root_pool，初始化内存池（可理解为Physical Frame Allocator），用于管理空闲内存，使用bitmap记录空闲内存的可用情况。

3. 定义了重要函数mem_alloc_map，用于在从内存池中分配一段连续物理内存，并自动寻找空闲的虚拟地址映射到这段内存上。

### 初始化阶段：console_init

初始化串口输出，首先用mem_alloc_map_dev将串口的物理地址映射到hypervisor的全局虚拟地址空间，然后调用uart_init进行初始化，调用uart_enable使能串口。

### 初始化阶段：interrupts_init

这部分主要是初始化gic（物理中断）<详见gic_init()>。以gicv3为例：

- 启用了ICC_*的cpu接口，也就是说可以直接访问ICC_*寄存器，而不是必须用mmio访问。

```c
sysreg_icc_sre_el2_write(ICC_SRE_SRE_BIT | ICC_SRE_ENB_BIT);
```

- 初始化gicd：gic分发器（GIC Distributor）是用于管理SPI（共享外设中断）的配置的，对应中断编号为32及以上。这里配置[GIC_CPU_PRIV, int_num) 编号范围内的中断全部为禁用状态。然后打开gicd亲和性路由以及gicd总使能开关。

```c
    /* Bring distributor to known state */
    for (size_t i = GIC_NUM_PRIVINT_REGS; i < GIC_NUM_INT_REGS(int_num); i++) {
        /**
         * Make sure all interrupts are not enabled, non pending,
         * non active.
         */
        gicd->IGROUPR[i] = -1;
        gicd->ICENABLER[i] = -1;
        gicd->ICPENDR[i] = -1;
        gicd->ICACTIVER[i] = -1;
    }
    
    for (size_t i = GIC_CPU_PRIV; i < GIC_MAX_INTERUPTS; i++) {
        gicd->IROUTER[i] = GICD_IROUTER_INV;
    }

    /* Enable distributor and affinity routing */
    gicd->CTLR |= GICD_CTLR_ARE_NS_BIT | GICD_CTLR_ENA_BIT;
```

- 初始化gicr：处于gic_cpu_init()中的gicr_init()。gicr用于管理PPI（Private Peripheral Interrupt，私有外设中断）和 SGI（Software Generated Interrupt，软件生成中断）的配置设置，对应的编号范围为[0, 31]。

- 初始化gicc：处于gic_cpu_init()中的gicc_init()。

### 初始化阶段：vmm_init

1. vmm_arch_init()： 初始化vtcr_el2、hcr_el2、cptr_el2（全直通）

2. vmm_io_init()： 初始化smmu（这部分qemu并不需要）

3. ipc_init()： 初始化虚拟机间的通信区域，分配物理内存空间

4. vmm_assign_vcpu(&master, &vm_id)： 所有cpu执行这一函数，尝试争夺虚拟机，从而完成vm和vcpu的分配。该函数会设置master和vm_id两个变量。

5. vmm_alloc_install_vm(vm_id, master)： 给vm和vcpu结构体分配物理内存空间，然后将vm结构体（包括vcpu）映射到目前cpu的地址空间中。

6. vm_init():

- vm_master_init(vm, config, vm_id)：其中的as_init(&vm->as, AS_VM, vm->id, NULL, config->colors)用于分配vm的stage-2地址翻译页表。

- vm_vcpu_init(vm, config): 初始化每个cpu的vcpu结构体（包括vcpu_id和上下文），设置虚拟机入口点entry。

- vm_arch_init(vm, config)：初始化vgic。

- vm_init_mem_regions(vm, config)： 将虚拟机镜像映射到stage-2页表中，保证gpa->hpa的转换不会出错。

7. vcpu_run(): 调用vcpu_arch_entry()进入el1执行，此后需通过触发异常才能再次进入el2。

```c
.global vcpu_arch_entry
vcpu_arch_entry:
    mrs x0, tpidr_el2
    ldr x0, [x0, #CPU_VCPU_OFF]
    add x0, x0, #VCPU_REGS_OFF
    mov sp, x0

    ldp x0, x1, [sp, #(8*31)]
    msr ELR_EL2, x0
    msr SPSR_EL2, x1

    ldp x0, x1,   [sp, #(8*0)]
    ldp x2, x3,   [sp, #(8*2)]
    ldp x4, x5,   [sp, #(8*4)]
    ldp x6, x7,   [sp, #(8*6)]
    ldp x8, x9,   [sp, #(8*8)]
    ldp x10, x11, [sp, #(8*10)]
    ldp x12, x13, [sp, #(8*12)]
    ldp x14, x15, [sp, #(8*14)]
    ldp x16, x17, [sp, #(8*16)]
    ldp x18, x19, [sp, #(8*18)]
    ldp x20, x21, [sp, #(8*20)]
    ldp x22, x23, [sp, #(8*22)]
    ldp x24, x25, [sp, #(8*24)]
    ldp x26, x27, [sp, #(8*26)]
    ldp x28, x29, [sp, #(8*28)]
    ldr x30,      [sp, #(8*30)]
    
    eret
    b   .
```

### 异常处理

```c
.balign 0x800
.global _hyp_vector_table
_hyp_vector_table:

/* 
 * EL2 with SP0
 */  
.balign ENTRY_SIZE
curr_el_sp0_sync:        
    b	.
.balign ENTRY_SIZE
curr_el_sp0_irq:  
    b   .
.balign ENTRY_SIZE
curr_el_sp0_fiq:         
    b	.
.balign ENTRY_SIZE
curr_el_sp0_serror:      
    b	.
          

/* 
 * EL2 with SPx
 */  
.balign ENTRY_SIZE  
curr_el_spx_sync:
    SAVE_HYP_GPRS
    mov x0, sp
    bl	internal_abort_handler        
    b	.
.balign ENTRY_SIZE
curr_el_spx_irq:         
    b	.
.balign ENTRY_SIZE
curr_el_spx_fiq:         
    b	.
.balign ENTRY_SIZE
curr_el_spx_serror:
    SAVE_HYP_GPRS
    mov x0, sp
    bl	internal_abort_handler       
    b	.         

/* 
 * Lower EL using AArch64
 */  

.balign ENTRY_SIZE
lower_el_aarch64_sync:
    VM_EXIT
    bl	aborts_sync_handler
    b   vcpu_arch_entry
.balign ENTRY_SIZE
lower_el_aarch64_irq:    
    VM_EXIT
    bl  gic_handle
    b   vcpu_arch_entry
.balign ENTRY_SIZE
lower_el_aarch64_fiq:    
    b	.
.balign ENTRY_SIZE
lower_el_aarch64_serror: 
    b	.          

/* 
 * Lower EL using AArch32
 */  
.balign ENTRY_SIZE   
lower_el_aarch32_sync:   
    b	.
.balign ENTRY_SIZE
lower_el_aarch32_irq:    
    b	.
.balign ENTRY_SIZE
lower_el_aarch32_fiq:    
    b	.
.balign ENTRY_SIZE
lower_el_aarch32_serror: 
    b	.

.balign ENTRY_SIZE
```

以上是el2的异常向量表定义，从中可以得出若干重要信息：

1. el2态中发生异常，会跳转到函数internal_abort_handler()：此时会panic提示系统崩溃。

2. el1/el0态发生同步异常：会跳转到函数aborts_sync_handler()处理异常。

3. el1/el0态发生irq异常：表示接收到了gic中断（物理中断）会跳转到函数gic_handle()处理异常。

### 同步异常处理：aborts_sync_handler

1. 根据相关寄存器计算异常地址ipa_fault_addr；从esr_el2中获取ec，根据ec跳转到相应异常处理函数（在abort_handlers中定义）。

2. bao做了全套的中断虚拟化：vgic异常在aborts_data_lower()函数中处理（这个时候hypervisor没有在虚拟机的stage-2页表中进行映射，所以会缺页异常）。

```c

emul_handler_t handler = vm_emul_get_mem(cpu()->vcpu->vm, addr);

```

这里根据异常地址addr确定到底是访问的到底是哪个gic寄存器区域（gicd or gicr），并找到相关联的处理函数。

```c

if (handler(&emul)) {
    unsigned long pc_step = 2 + (2 * il);
    vcpu_writepc(cpu()->vcpu, vcpu_readpc(cpu()->vcpu) + pc_step);
} else {
    ERROR("data abort emulation failed (0x%x)", far);
}

```

调用处理函数handler进行vgic的处理，对在config里面注册的中断号，将修改同步到物理gic上。

### 处理gic中断：gic_handle()

```c
void gic_handle()
{
    uint32_t ack = gicc_iar();
    irqid_t id = bit32_extract(ack, GICC_IAR_ID_OFF, GICC_IAR_ID_LEN);
    if (id < GIC_FIRST_SPECIAL_INTID) {
        enum irq_res res = interrupts_handle(id);
        gicc_eoir(ack);
        if (res == HANDLED_BY_HYP) gicc_dir(ack);
    }
}
```

interrupts_handle()用来处理gic中断：

- 如果这个中断id已经被虚拟机注册，那么调用`vcpu_inject_hw_irq()`向当前vcpu接口注入中断（写lr寄存器）。
- 如果这个中断id被hypervisor保留了，那么由hypervisor处理。

