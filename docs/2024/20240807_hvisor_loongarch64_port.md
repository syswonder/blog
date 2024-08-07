# 移植hvisor到LoongArch64架构

wheatfox

## 2024.7.15记录

https://www.kernelconfig.io/config_builtin_dtb

今天在研究如何给hvisor启动的zone0 linux传递dtb时，发现目前**6.9.8的linux**不仅可以内嵌initramfs作为rootfs，还可以通过配置CONFIG_BUILTIN_DTB编译配置选择内嵌的dtb（之前的内核版本里loongarch部分没有这个功能，看changelog应该是最近几个版本新加的，同时在linux仓库中arch/loongarch/boot内多了几个2K1000/2K500的dts文件，正好可以用于内嵌DTB），我将我自己写的3A5000的dts也放了进去。

arch/loongarch/kernel/setup.c：

```clike
static void __init fdt_setup(void)
{
#ifdef CONFIG_OF_EARLY_FLATTREE
	void *fdt_pointer;

	/* ACPI-based systems do not require parsing fdt */
	if (acpi_os_get_root_pointer())
		return;

	/* Prefer to use built-in dtb, checking its legality first. */
	if (IS_ENABLED(CONFIG_BUILTIN_DTB) && !fdt_check_header(__dtb_start))
		fdt_pointer = __dtb_start;
	else
		fdt_pointer = efi_fdt_pointer(); /* Fallback to firmware dtb */

	if (!fdt_pointer || fdt_check_header(fdt_pointer))
		return;

	early_init_dt_scan(fdt_pointer);
	early_init_fdt_reserve_self();

	max_low_pfn = PFN_PHYS(memblock_end_of_DRAM());
#endif
}
```

可以看到，如果配置了CONFIG_BUILTIN_DTB则fdt_pointer则直接指向编译时内嵌在vmlinux里的dtb文件（在内存中的地址），这样就可以不需要外部BIOS或UEFI传递FDT信息（*loongarch新世界的规范是无路如何绕不开UEFI的，即使使用FDT，也依然需要通过UEFI变量表获取FDT地址，非常麻烦*）。

具体的内嵌dtb流程为：

1. 在arch/loongarch/boot/dts中放好需要的dts文件（可以放dtsi并include），如x.dts
2. 修养这个目录下的Makefile，添加x.**dtb**的目标
3. 在menuconfig里打开CONFIG_BUILTIN_DTB（Kernel type and options），填入dts的名字（不带任何后缀！）
4. 之后编译内核时就可以自动编译dtb并内嵌入内核了

**SMP BOOT**

第一个问题是percpu初始化时，唯独只有zone成员初始化为None会导致程序的atomic_sub出现错误参数。原因？

目前能够启动cpu1-3了，但是PRINT_LOCK莫名其妙失效了，而且cpu0在读取BOOT CPU COUNT的时候也永远没有等到，这两个涉及到spin的原子操作均失效了。原因？

推测是CPU1和CPU0看到的内存视图不一致，同一个lock在CPU0修改之后，CPU1仍然拿不到新值，原因？

## 2024.7.16记录

SMP的锁问题已解决，问题在entry.rs中，每个CPU进入时都把bss清理了一遍，自然会让之前别的CPU上的锁失效，注释掉后没有任何问题，输出锁等均正常了（**bss的清理调整为UEFI Packer负责，提前将相关内存memset**）

## 2024.7.17记录

尝试启动zone0 root linux，能够启动，cpucfg模拟啥的没有问题，目前还是存在idle问题，在启动real console后虚拟机运行idle指令等待中断。

看一下KVM怎么实现的：

```clike
case 0x6: /* Cache, Idle and IOCSR GSPR */
		switch (((inst.word >> 22) & 0x3ff)) {
		case 0x18: /* Cache GSPR */
			er = EMULATE_DONE;
			trace_kvm_exit_cache(vcpu, KVM_TRACE_EXIT_CACHE);
			break;
		case 0x19: /* Idle/IOCSR GSPR */
			switch (((inst.word >> 15) & 0x1ffff)) {
			case 0xc90: /* IOCSR GSPR */
				er = kvm_emu_iocsr(inst, run, vcpu);
				break;
			case 0xc91: /* Idle GSPR */
				er = kvm_emu_idle(vcpu);
				break;
			default:
				er = EMULATE_FAIL;
				break;
			}
			break;
		default:
			er = EMULATE_FAIL;
			break;
		}
		break;
```

看一下kvm_emu_idle

```clike
kvm_vcpu_check_block/*
 * Emulate a vCPU halt condition, e.g. HLT on x86, WFI on arm, etc...  If halt
 * polling is enabled, busy wait for a short time before blocking to avoid the
 * expensive block+unblock sequence if a wake event arrives soon after the vCPU
 * is halted.
 */
void kvm_vcpu_halt(struct kvm_vcpu *vcpu)
{
	unsigned int max_halt_poll_ns = kvm_vcpu_max_halt_poll_ns(vcpu);
	bool halt_poll_allowed = !kvm_arch_no_poll(vcpu);
	ktime_t start, cur, poll_end;
	bool waited = false;
	bool do_halt_poll;
	u64 halt_ns;

	if (vcpu->halt_poll_ns > max_halt_poll_ns)
		vcpu->halt_poll_ns = max_halt_poll_ns;

	do_halt_poll = halt_poll_allowed && vcpu->halt_poll_ns;

	start = cur = poll_end = ktime_get();
	if (do_halt_poll) {
		ktime_t stop = ktime_add_ns(start, vcpu->halt_poll_ns);

		do {
			if (kvm_vcpu_check_block(vcpu) < 0)
				goto out;
			cpu_relax();
			poll_end = cur = ktime_get();
		} while (kvm_vcpu_can_poll(cur, stop));
	}

	waited = kvm_vcpu_block(vcpu);

	cur = ktime_get();
	if (waited) {
		vcpu->stat.generic.halt_wait_ns +=
			ktime_to_ns(cur) - ktime_to_ns(poll_end);
		KVM_STATS_LOG_HIST_UPDATE(vcpu->stat.generic.halt_wait_hist,
				ktime_to_ns(cur) - ktime_to_ns(poll_end));
	}
out:
	/* The total time the vCPU was "halted", including polling time. */
	halt_ns = ktime_to_ns(cur) - ktime_to_ns(start);

	/*
	 * Note, halt-polling is considered successful so long as the vCPU was
	 * never actually scheduled out, i.e. even if the wake event arrived
	 * after of the halt-polling loop itself, but before the full wait.
	 */
	if (do_halt_poll)
		update_halt_poll_stats(vcpu, start, poll_end, !waited);

	if (halt_poll_allowed) {
		/* Recompute the max halt poll time in case it changed. */
		max_halt_poll_ns = kvm_vcpu_max_halt_poll_ns(vcpu);

		if (!vcpu_valid_wakeup(vcpu)) {
			shrink_halt_poll_ns(vcpu);
		} else if (max_halt_poll_ns) {
			if (halt_ns <= vcpu->halt_poll_ns)
				;
			/* we had a long block, shrink polling */
			else if (vcpu->halt_poll_ns &&
				 halt_ns > max_halt_poll_ns)
				shrink_halt_poll_ns(vcpu);
			/* we had a short halt and our poll time is too small */
			else if (vcpu->halt_poll_ns < max_halt_poll_ns &&
				 halt_ns < max_halt_poll_ns)
				grow_halt_poll_ns(vcpu);
		} else {
			vcpu->halt_poll_ns = 0;
		}
	}

	trace_kvm_vcpu_wakeup(halt_ns, waited, vcpu_valid_wakeup(vcpu));
}
EXPORT_SYMBOL_GPL(kvm_vcpu_halt);int kvm_emu_idle(struct kvm_vcpu *vcpu)
{
	++vcpu->stat.idle_exits;
	trace_kvm_exit_idle(vcpu, KVM_TRACE_EXIT_IDLE);

	if (!kvm_arch_vcpu_runnable(vcpu))
		kvm_vcpu_halt(vcpu);

	return EMULATE_DONE;
}

// kvm_arch_vcpu_runnable
// 为真条件：irq_pending=1且mp_state为RUNNABLE（multiprocessing-state）
virt/kvm/kvm_main.c

kvm_vcpu_block
/*
 * Block the vCPU until the vCPU is runnable, an event arrives, or a signal is
 * pending.  This is mostly used when halting a vCPU, but may also be used
 * directly for other vCPU non-runnable states, e.g. x86's Wait-For-SIPI.
 */
bool kvm_vcpu_block(struct kvm_vcpu *vcpu)
{
	struct rcuwait *wait = kvm_arch_vcpu_get_wait(vcpu);
	bool waited = false;

	vcpu->stat.generic.blocking = 1;

	preempt_disable();
	kvm_arch_vcpu_blocking(vcpu);
	prepare_to_rcuwait(wait);
	preempt_enable();

	for (;;) {
		set_current_state(TASK_INTERRUPTIBLE);

		if (kvm_vcpu_check_block(vcpu) < 0)
			break;

		waited = true;
		schedule();
	}

	preempt_disable();
	finish_rcuwait(wait);
	kvm_arch_vcpu_unblocking(vcpu);
	preempt_enable();

	vcpu->stat.generic.blocking = 0;

	return waited;
}

kvm_vcpu_check_block
static int kvm_vcpu_check_block(struct kvm_vcpu *vcpu)
{
	int ret = -EINTR;
	int idx = srcu_read_lock(&vcpu->kvm->srcu);

	if (kvm_arch_vcpu_runnable(vcpu))
		goto out;
	if (kvm_cpu_has_pending_timer(vcpu))
		goto out;
	if (signal_pending(current))
		goto out;
	if (kvm_check_request(KVM_REQ_UNBLOCK, vcpu))
		goto out;

	ret = 0;
out:
	srcu_read_unlock(&vcpu->kvm->srcu, idx);
	return ret;
}
```

```clike
/*
 * Emulate a vCPU halt condition, e.g. HLT on x86, WFI on arm, etc...  If halt
 * polling is enabled, busy wait for a short time before blocking to avoid the
 * expensive block+unblock sequence if a wake event arrives soon after the vCPU
 * is halted.
 */
 // 如果开启halt polling则等一段时间而不是直接休眠
void kvm_vcpu_halt(struct kvm_vcpu *vcpu)
```

看一下linux源码里是哪里调用的idle，只有两个地方：arch_cpu_idle_dead和__arch_cpu_idle

根据JTAG查看vm exit的指令地址可以确定第一次idle出现在这个地方：

```clike
90000000002212c0 <>:
90000000002212c0:	28c0204c 	ld.d        	$t0, $tp, 8(0x8)
90000000002212c4:	03400000 	andi        	$zero, $zero, 0x0
90000000002212c8:	0340118c 	andi        	$t0, $t0, 0x4
90000000002212cc:	44001580 	bnez        	$t0, 20(0x14)	# 90000000002212e0 <__arch_cpu_idle+0x20>
90000000002212d0:	03400000 	andi        	$zero, $zero, 0x0
90000000002212d4:	03400000 	andi        	$zero, $zero, 0x0
90000000002212d8:	03400000 	andi        	$zero, $zero, 0x0
90000000002212dc:	06488000 	idle        	0x0
90000000002212e0:	4c000020 	jirl        	$zero, $ra, 0
```

```clike
SYM_FUNC_START(__arch_cpu_idle)
	/* start of rollback region */
	LONG_L	t0, tp, TI_FLAGS
	nop
	andi	t0, t0, _TIF_NEED_RESCHED
	bnez	t0, 1f
	nop
	nop
	nop
	idle	0
	/* end of rollback region */
1:	jr	ra
SYM_FUNC_END(__arch_cpu_idle)
```

想了一下既然已经配置了外部中断直通到虚拟机，那idle模拟也可以直接返回原指令运行，然后让虚拟机等外部中断即可，暂时能够解决问题。

**vmalloc问题**

之前遇到过，当时debug后发现linux给vmalloc的区域是个**空集（low>high）**，原因未知，但是vmlinux.efi启动时没有问题，用我自己的设备树+vmlinux就会有问题

这样看起来是设备树写的有bug？

[https://github.com/enkerewpo/linux-loongarch64/commit/cd6e6814ae7635f3ea2ac99882c09ed38f1c93b3](https://github.com/enkerewpo/linux-loongarch64/commit/cd6e6814ae7635f3ea2ac99882c09ed38f1c93b3)

```clike
[    1.441054] i8042: Probing ports directly.
[    1.445143] CPU 0 Unable to handle kernel paging request at virtual address ffff800000008064, era == 9000000002de2274, ra == 9000000002de225c
[    1.457767] Oops[#1]:
[    1.460021] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 6.9.8 #6
[    1.465817] pc 9000000002de2274 ra 9000000002de225c tp 900000000044c000 sp 900000000044fd20
[    1.474115] a0 00000000000000b4 a1 0000000000000000 a2 900000000044c000 a3 90000000031b11e0
[    1.482414] a4 90000000004b7df0 a5 900000000044fba0 a6 0000000000000001 a7 9000000000490080
[    1.490712] t0 ffff800000008000 t1 0000000000000064 t2 0000000000008000 t3 000000005623054a
[    1.499009] t4 0000000000000000 t5 0000000000000000 t6 0000000000000001 t7 0000000000000001
[    1.507307] t8 9000000000490090 u0 000000000000001e s9 0000000000000008 s0 9000000003e29b78
[    1.515606] s1 9000000003e29b78 s2 00000000000000b4 s3 9000000003ccb278 s4 900000000392e8fc
[    1.523904] s5 9000000003d24000 s6 9000000003220560 s7 90000000032408a0 s8 90000000031c00b0
[    1.532202]    ra: 9000000002de225c i8042_flush+0x3c/0x13c
[    1.537660]   ERA: 9000000002de2274 i8042_flush+0x54/0x13c
[    1.543108]  CRMD: 000000b0 (PLV0 -IE -DA +PG DACF=CC DACM=CC -WE)
[    1.549257]  PRMD: 00000000 (PPLV0 -PIE -PWE)
[    1.553584]  EUEN: 00000000 (-FPE -SXE -ASXE -BTE)
[    1.558345]  ECFG: 00071c1c (LIE=2-4,10-12 VS=7)
[    1.562932] ESTAT: 00010000 [PIL] (IS= ECode=1 EsubCode=0)
[    1.568381]  BADV: ffff800000008064
[INFO  0] (hvisor::arch::loongarch64::trap:858) cpucfg emulation, target cpucfg index is 0x0
[    1.583286]  PRID: 0014c011 (Loongson-64bit, Loongson-3A5000-H)
```

arch/loongarch/include/asm/sparsemem.h

```clike
#ifdef CONFIG_SPARSEMEM_VMEMMAP
#define VMEMMAP_SIZE	(sizeof(struct page) * (1UL << (cpu_pabits + 1 - PAGE_SHIFT)))
#endif
```

arch/loongarch/include/asm/pgtable.h

```clikepp
#define VMALLOC_START	MODULES_END

#ifndef CONFIG_KASAN
#define VMALLOC_END	\
	(vm_map_base +	\
	 min(PTRS_PER_PGD * PTRS_PER_PUD * PTRS_PER_PMD * PTRS_PER_PTE * PAGE_SIZE, (1UL << cpu_vabits)) - PMD_SIZE - VMEMMAP_SIZE - KFENCE_AREA_SIZE)
#else
#define VMALLOC_END	\
	(vm_map_base +	\
	 min(PTRS_PER_PGD * PTRS_PER_PUD * PTRS_PER_PMD * PTRS_PER_PTE * PAGE_SIZE, (1UL << cpu_vabits) / 2) - PMD_SIZE - VMEMMAP_SIZE - KFENCE_AREA_SIZE)
#endif

#define vmemmap		((struct page *)((VMALLOC_END + PMD_SIZE) & PMD_MASK))
#define VMEMMAP_END	((unsigned long)vmemmap + VMEMMAP_SIZE - 1)

#define KFENCE_AREA_START	(VMEMMAP_END + 1)
#define KFENCE_AREA_END		(KFENCE_AREA_START + KFENCE_AREA_SIZE - 1)
```

但是在6.9.8版本里这样修改仍然不能解决问题，而且linux内核代码也不应该修改（除了添加debug输出之外不能修改代码逻辑）

看一下报错的地方

arch/loongarch/mm/fault.c

```clike
static void __kprobes no_context(struct pt_regs *regs,
			unsigned long write, unsigned long address)
{
	const int field = sizeof(unsigned long) * 2;

	/* Are we prepared to handle this kernel fault?	 */
	if (fixup_exception(regs))
		return;

	if (kfence_handle_page_fault(address, write, regs))
		return;

	/*
	 * Oops. The kernel tried to access some bad page. We'll have to
	 * terminate things with extreme prejudice.
	 */
	bust_spinlocks(1);

	pr_alert("CPU %d Unable to handle kernel paging request at "
	       "virtual address %0*lx, era == %0*lx, ra == %0*lx\n",
	       raw_smp_processor_id(), field, address, field, regs->csr_era,
	       field,  regs->regs[1]);
	die("Oops", regs);
}
```

pr_alert打印一下vmalloc_start和vmalloc_end

```clike
[    1.839832] vmalloc start: ffff800012002000, vmalloc end: fffffbffffe00000

[    1.840562] CPU 0 Unable to handle kernel paging request at virtual address ffff800000002064, era == 9000000002cd2a28, ra == 9000000002cd2a10

[    1.859834] Call Trace:
[    1.860073] [<9000000002cd2a28>] _flush+0x54/0x13c
[    1.861479] [<9000000002e85dac>] i8042_init+0x2c4/0x36c
[    1.861822] [<90000000022a0154>] do_one_initcall+0x78/0x1cc
[    1.862138] [<9000000002e50f2c>] kernel_init_freeable+0x234/0x2a4
[    1.862476] [<9000000002e36250>] kernel_init+0x1c/0x10c
[    1.862805] [<90000000022a1228>] ret_from_kernel_thread+0xc/0xa4
```

i8042是键盘驱动，下图是ARM下vmalloc区域的结构，vmalloc_start在内核模块后，vmalloc_end则在固定映射之前。

所以键盘驱动为什么要申请一个不在vmalloc区域的页？这其实就是内核报错的原因，这个地址确实是不对的。

![Untitled](imgs/20240807_hvisor_loongarch64_port/Untitled.png)

去掉键盘驱动后没问题了，vmlinux在qemu里也可以直接运行，进入shell并且能运行指令

但是上板后的root linux还是有老问题，启动init后就没输出了

![Untitled](imgs/20240807_hvisor_loongarch64_port/Untitled%201.png)

也不能说没输出，可以看到这次启动时rcS脚本里的echo就打印出来了（一部分），然后就死掉了……每次启动时似乎卡死的位置还不一样

**依然怀疑是IDLE，这个需要深入研究一下KVM的逻辑**

![Untitled](imgs/20240807_hvisor_loongarch64_port/Untitled%202.png)

时钟问题？

注释调用__arch_cpu_idle的函数逻辑

## 2024.7.23记录

调查一下linux在loongarch架构下调用idle指令的流程：

```clike
/// arch/loongarch/kernel/genex.S
/// 有一些异常处理和复位相关的汇编函数
SYM_FUNC_START()

	/* start of rollback region */
	LONG_L	t0, tp, TI_FLAGS // 加载 mem[$tp+TI_FLAGS] 到 $t0 
	// OFFSET(TI_FLAGS, thread_info, flags);

	nop

	andi	t0, t0, _TIF_NEED_RESCHED // 检查线程是否需要重新调度
	// #define _TIF_NEED_RESCHED	(1<<TIF_NEED_RESCHED) thread_information_flags, arch/loongarch/include/asm/thread_info.h
	bnez	t0, 1f // !=0，即需要调度，直接return

	nop
	nop
	nop

	idle	0 // 不调度则直接idle CPU

	/* end of rollback region */
1:	jr	ra

SYM_FUNC_END(__arch_cpu_idle)

/// arch/loongarch/kernel/idle.c
void __cpuidle arch_cpu_idle(void)
{
	raw_local_irq_enable();
	__arch_cpu_idle(); /* idle instruction needs irq enabled */
	raw_local_irq_disable();
}

/// kernel/sched/idle.c

/**
 * default_idle_call - Default CPU idle routine.
 *
 * To use when the cpuidle framework cannot be used.
 */
void __cpuidle default_idle_call(void)
{
	instrumentation_begin();
	if (!current_clr_polling_and_test()) {
		cond_tick_broadcast_enter();
		trace_cpu_idle(1, smp_processor_id());
		stop_critical_timings();

		ct_cpuidle_enter();
		arch_cpu_idle();
		ct_cpuidle_exit();

		start_critical_timings();
		trace_cpu_idle(PWR_EVENT_EXIT, smp_processor_id());
		cond_tick_broadcast_exit();
	}
	local_irq_enable();
	instrumentation_end();
}

```

main idle function

```clike
/**
 * cpuidle_idle_call - the main idle function
 *
 * NOTE: no locks or semaphores should be used here
 *
 * On architectures that support TIF_POLLING_NRFLAG, is called with polling
 * set, and it returns with polling set.  If it ever stops polling, it
 * must clear the polling bit.
 */
static void cpuidle_idle_call(void)
{
	struct cpuidle_device *dev = cpuidle_get_device();
	struct cpuidle_driver *drv = cpuidle_get_cpu_driver(dev);
	int next_state, entered_state;

	/*
	 * Check if the idle task must be rescheduled. If it is the
	 * case, exit the function after re-enabling the local irq.
	 */
	if (need_resched()) {
		local_irq_enable();
		return;
	}

	/*
	 * The RCU framework needs to be told that we are entering an idle
	 * section, so no more rcu read side critical sections and one more
	 * step to the grace period
	 */

	if (cpuidle_not_available(drv, dev)) {
		tick_nohz_idle_stop_tick();

		default_idle_call();
		goto exit_idle;
	}

	/*
	 * Suspend-to-idle ("s2idle") is a system state in which all user space
	 * has been frozen, all I/O devices have been suspended and the only
	 * activity happens here and in interrupts (if any). In that case bypass
	 * the cpuidle governor and go straight for the deepest idle state
	 * available.  Possibly also suspend the local tick and the entire
	 * timekeeping to prevent timer interrupts from kicking us out of idle
	 * until a proper wakeup interrupt happens.
	 */

	if (idle_should_enter_s2idle() || dev->forced_idle_latency_limit_ns) {
		u64 max_latency_ns;

		if (idle_should_enter_s2idle()) {

			entered_state = call_cpuidle_s2idle(drv, dev);
			if (entered_state > 0)
				goto exit_idle;

			max_latency_ns = U64_MAX;
		} else {
			max_latency_ns = dev->forced_idle_latency_limit_ns;
		}

		tick_nohz_idle_stop_tick();

		next_state = cpuidle_find_deepest_state(drv, dev, max_latency_ns);
		call_cpuidle(drv, dev, next_state);
	} else {
		bool stop_tick = true;

		/*
		 * Ask the cpuidle framework to choose a convenient idle state.
		 */
		next_state = cpuidle_select(drv, dev, &stop_tick);

		if (stop_tick || tick_nohz_tick_stopped())
			tick_nohz_idle_stop_tick();
		else
			tick_nohz_idle_retain_tick();

		entered_state = call_cpuidle(drv, dev, next_state);
		/*
		 * Give the governor an opportunity to reflect on the outcome
		 */
		cpuidle_reflect(dev, entered_state);
	}

exit_idle:
	__current_set_polling();

	/*
	 * It is up to the idle functions to reenable local interrupts
	 */
	if (WARN_ON_ONCE(irqs_disabled()))
		local_irq_enable();
}
```

先试着注释掉idle指令，在qemu中运行没有问题，上板也突然没有问题了，可以用shell了。

**首先发现了几个问题：**

1. idle指令和硬件中断没有太大关系，所以LVZ里的硬件中断虚拟化（GINTC）不一定能解决问题。
2. idle指令的触发器包括外部中断和内部中断（时钟中断、软中断等）。
3. 当CPU处于idle时，PC暂停，那么能够触发中断的只有IPI、Timer、HWI，虚拟机目前只考虑Timer和HWI
4. **怀疑问题可能出在虚拟机自己的【timer中断】的触发上了，这也和之前出现的输出随机卡死有关（时间因素）**
5. 关于host和guest模式下Timer的具体行为目前仅为猜测，没有任何细节资料

![Untitled](imgs/20240807_hvisor_loongarch64_port/Untitled%203.png)

试着移植一下loongarch systemd

[https://github.com/systemd/systemd](https://github.com/systemd/systemd)

https://gitlab.summer-ospp.ac.cn/summer2021/210080299/-/blob/2cf8abe1972f61e1394c167a348282928bb7c4fc/CLFS_For_LoongArch64-20210618.md

大概调查了一下，目前能找到的cross-compile编译器缺很多依赖（如libcap-dev就往往不会提供），目前想编译一个loongarch systemd还是比较困难，缺乏资料（也许可以从一些loongarch发行版比如安同AOSC的文档里找找），目前暂时用busybox。

接下来可以研究一下hvisor-tool在loongarch下的移植

以及hvisor本身在loongarch架构下异常处理和hvisor本身的hypercall系统对接。

**问题：KVM到底如何模拟的idle指令？必须完整搞清楚，不然解决不了这个idle的问题**

## 2024.7.30记录

两个核心问题：Linux在什么情况下调用cpu halt？KVM是怎么模拟idle的？

### cpuidle组件（drivers/cpuidle）

`cpuidle device`和`cpuidle driver`是Linux内核中用于管理CPU空闲状态的两个关键组件。它们是`cpuidle`框架的一部分，负责在处理器空闲时有效地管理电源，以减少能耗并提高电池寿命。以下是它们的详细说明：

cpuidle device

- **定义**：`cpuidle device`表示一个特定的CPU或一组CPU，它们的空闲状态需要被管理。
- **作用**：`cpuidle device`为每个CPU或CPU集群维护一个可用的空闲状态列表。不同的空闲状态表示CPU可以进入的不同级别的低功耗状态，每个状态都有不同的功耗和退出延迟。
- **功能**：在运行时，内核会根据系统的负载、每个空闲状态的特性以及历史行为来选择最合适的空闲状态，以平衡功耗和性能。

cpuidle driver

- **定义**：`cpuidle driver`是一个负责实现特定CPU架构的空闲状态管理的驱动程序。
- **作用**：`cpuidle driver`为特定平台提供了具体的空闲状态实现和管理接口。它定义了每个空闲状态的特征（如功耗、退出延迟）以及如何进入和退出这些状态。
- **功能**：`cpuidle driver`需要注册到`cpuidle`框架，并为`cpuidle device`提供所需的空闲状态信息。不同的CPU架构或平台可能有不同的空闲状态集，因此需要不同的`cpuidle driver`来处理。

当CPU空闲时，`cpuidle`框架会调用相应的`cpuidle driver`，通过对空闲状态进行评估和选择，将CPU置于最合适的低功耗状态。当有任务需要处理时，CPU会从空闲状态中唤醒，以继续处理工作负载。这种机制帮助系统在不需要全部CPU性能时节省电力，从而延长电池寿命并减少能源消耗。

```clikepp
/*
 * Per process flags
 */
#define PF_VCPU			0x00000001	/* I'm a virtual CPU */
#define PF_IDLE			0x00000002	/* I am an IDLE thread */
#define PF_EXITING		0x00000004	/* Getting shut down */
#define PF_POSTCOREDUMP		0x00000008	/* Coredumps should ignore this task */
#define PF_IO_WORKER		0x00000010	/* Task is an IO worker */
#define PF_WQ_WORKER		0x00000020	/* I'm a workqueue worker */
#define PF_FORKNOEXEC		0x00000040	/* Forked but didn't exec */
#define PF_MCE_PROCESS		0x00000080      /* Process policy on mce errors */
#define PF_SUPERPRIV		0x00000100	/* Used super-user privileges */
#define PF_DUMPCORE		0x00000200	/* Dumped core */
#define PF_SIGNALED		0x00000400	/* Killed by a signal */
#define PF_MEMALLOC		0x00000800	/* Allocating memory to free memory. See memalloc_noreclaim_save() */
#define PF_NPROC_EXCEEDED	0x00001000	/* set_user() noticed that RLIMIT_NPROC was exceeded */
#define PF_USED_MATH		0x00002000	/* If unset the fpu must be initialized before use */
#define PF_USER_WORKER		0x00004000	/* Kernel thread cloned from userspace thread */
#define PF_NOFREEZE		0x00008000	/* This thread should not be frozen */
#define PF__HOLE__00010000	0x00010000
#define PF_KSWAPD		0x00020000	/* I am kswapd */
#define PF_MEMALLOC_NOFS	0x00040000	/* All allocations inherit GFP_NOFS. See memalloc_nfs_save() */
#define PF_MEMALLOC_NOIO	0x00080000	/* All allocations inherit GFP_NOIO. See memalloc_noio_save() */
#define PF_LOCAL_THROTTLE	0x00100000	/* Throttle writes only against the bdi I write to,
						 * I am cleaning dirty pages from some other bdi. */
#define PF_KTHREAD		0x00200000	/* I am a kernel thread */
#define PF_RANDOMIZE		0x00400000	/* Randomize virtual address space */
#define PF_MEMALLOC_NORECLAIM	0x00800000	/* All allocation requests will clear __GFP_DIRECT_RECLAIM */
#define PF_MEMALLOC_NOWARN	0x01000000	/* All allocation requests will inherit __GFP_NOWARN */
#define PF__HOLE__02000000	0x02000000
#define PF_NO_SETAFFINITY	0x04000000	/* Userland is not allowed to meddle with cpus_mask */
#define PF_MCE_EARLY		0x08000000      /* Early kill for mce process policy */
#define PF_MEMALLOC_PIN		0x10000000	/* Allocations constrained to zones which allow long term pinning.
						 * See memalloc_pin_save() */
#define PF_BLOCK_TS		0x20000000	/* plug has ts that needs updating */
#define PF__HOLE__40000000	0x40000000
#define PF_SUSPEND_TASK		0x80000000      /* This thread called freeze_processes() and should not be frozen */
```

之前去掉idle指令后能够正常运行的原因：实际上每次kernel调用do_idle时都是一个while循环，里面由do_idle决定是调用idle指令还是polling，所以注释idle指令后不影响调度器的while循环。

### 一种不用修改内核源码的解决方案（不模拟idle）

我也找到了一种不修改linux源码就能够让linux运行在polling模式的halt而不是idle指令halt的方法，利用bootargs和define可以进行配置：

CONFIG_GENERIC_IDLE_POLL_SETUP

```clike
#ifdef CONFIG_GENERIC_IDLE_POLL_SETUP
static int __init cpu_idle_poll_setup(char *__unused)
{
	cpu_idle_force_poll = 1;

	return 1;
}
__setup("nohlt", cpu_idle_poll_setup);

static int __init cpu_idle_nopoll_setup(char *__unused)
{
	cpu_idle_force_poll = 0;

	return 1;
}
__setup("hlt", cpu_idle_nopoll_setup);
#endif
```

bootargs中添加nohlt参数

```clike
**nohlt**		[ARM,ARM64,MICROBLAZE,MIPS,PPC,SH] Forces the kernel to
			busy wait in do_idle() and not use the arch_cpu_idle()
			implementation; requires CONFIG_GENERIC_IDLE_POLL_SETUP
			to be effective. This is useful on platforms where the
			sleep(SH) or wfi(ARM,ARM64) instructions do not work
			correctly or when doing power measurements to evaluate
			the impact of the sleep instructions. This is also
			useful when using JTAG debugger.
			
nohlt   强制内核在do_idle()中进行忙等待，而不使用arch_cpu_idle()的实现；需要CONFIG_GENERIC_IDLE_POLL_SETUP配置项生效。
				这在以下情况下非常有用：平台上的sleep（SH）或wfi（ARM, ARM64）指令无法正常工作，或者在进行功耗测量时评估睡眠指令的影响。
				此外，在使用JTAG调试器时也很有用。
```

目前的bootargs（cmdline）为

```clike
  bootargs = "earlycon console=ttyS0,115200n8 initramfs_async=false rootwait nohlt";
```

这样可以修改内核调度器do_halt时的行为，进行polling而不是idle。我画了一个linux源码中和线程idle相关的调用关系图：

![1.drawio.png](imgs/20240807_hvisor_loongarch64_port/1.drawio.png)

### 不修改内核源码+hvisor模拟idle指令方案

之前尝试过在idle陷入hvisor后直接不处理，然后跳到guest的下一条指令继续执行，然而会出现init输出随机卡死的情况。

梳理一下kvm模拟idle的调用逻辑图：

![Untitled](imgs/20240807_hvisor_loongarch64_port/Untitled%204.png)

```clike
/*
 * 阻塞vCPU直到vCPU可以运行、接收到事件或者有信号待处理。
 * 主要用于暂停vCPU，但也可用于其他vCPU非可运行状态，比如x86的Wait-For-SIPI。
 */
bool (struct kvm_vcpu *vcpu)
{
    // 获取与vCPU关联的等待结构
    struct rcuwait *wait = kvm_arch_vcpu_get_wait(vcpu);
    bool waited = false; // 用于标记vCPU是否进入过等待状态

    // 标记vCPU进入阻塞状态
    vcpu->stat.generic.blocking = 1;

    // 禁用抢占，确保后续代码不会被中断
    preempt_disable();
    // 执行架构特定的vCPU阻塞准备
    kvm_arch_vcpu_blocking(vcpu);
    // 准备进入等待状态
    prepare_to_rcuwait(wait);
    // 重新启用抢占
    preempt_enable();

    for (;;) {
        // 设置当前任务状态为可中断的睡眠状态
        set_current_state(TASK_INTERRUPTIBLE);

        // 检查vCPU是否可以结束阻塞状态
        if (kvm_vcpu_check_block(vcpu) < 0)
            break; // 退出循环，结束等待

        waited = true; // 标记vCPU已经进入等待状态
        schedule(); // 调度其他任务，vCPU进入睡眠状态
    }

    // 禁用抢占，进行解除阻塞的清理工作
    preempt_disable();
    // 完成等待状态，进行相关清理
    finish_rcuwait(wait);
    // 执行架构特定的vCPU解除阻塞操作
    kvm_arch_vcpu_unblocking(vcpu);
    // 重新启用抢占
    preempt_enable();

    // 重置阻塞标志，表明vCPU已不再处于阻塞状态
    vcpu->stat.generic.blocking = 0;

    return waited; // 返回vCPU是否进入过等待状态
}

```

考虑到进入idle后，linux会大量使用时钟中断进行调度，而此时如果hypervisor没有在陷入时正确处理guest timer，就有可能导致问题（不仅是idle指令，hvcl指令可能也会导致guest timer异常从而卡死），所以目前看起来hvisor(loongarch）在陷入时的时钟+中断处理还有问题，**不一定是idle指令本身的模拟问题（毕竟直接跳过这条指令理论上是完全可以解决问题的，但是现在并不能，那就说明问题反而可能出在”陷入hypervisor“这个行为本身上）**。

### Hypervisor如何处理guest timer上下文（loongarch）

arch/loongarch/kvm/timer.c

```clike
kvm_arch_vcpu_put
_kvm_vcpu_put // save hw GCSRs to memory + save timer
kvm_save_timer
```

```clike
/*
 * Save guest timer state and switch to soft guest timer if hard timer was in
 * use.
 */
void kvm_save_timer(struct kvm_vcpu *vcpu)
{
	struct loongarch_csrs *csr = vcpu->arch.csr;

	preempt_disable();

	/* Save hard timer state */
	kvm_save_hw_gcsr(csr, LOONGARCH_CSR_TCFG);
	kvm_save_hw_gcsr(csr, LOONGARCH_CSR_TVAL);
	if (kvm_read_sw_gcsr(csr, LOONGARCH_CSR_TCFG) & CSR_TCFG_EN)
		_kvm_save_timer(vcpu);

	/* Save timer-related state to vCPU context */
	kvm_save_hw_gcsr(csr, LOONGARCH_CSR_ESTAT);
	preempt_enable();
}
```

```clike
/*
 * Save guest timer state and switch to software emulation of guest
 * timer. The hard timer must already be in use, so preemption should be
 * disabled.
 */
static void _kvm_save_timer(struct kvm_vcpu *vcpu)
{
	unsigned long ticks, delta, cfg;
	ktime_t expire;
	struct loongarch_csrs *csr = vcpu->arch.csr;

	cfg = kvm_read_sw_gcsr(csr, LOONGARCH_CSR_TCFG);
	ticks = kvm_read_sw_gcsr(csr, LOONGARCH_CSR_TVAL);

	/*
	 * From LoongArch Reference Manual Volume 1 Chapter 7.6.2
	 * If period timer is fired, CSR TVAL will be reloaded from CSR TCFG
	 * If oneshot timer is fired, CSR TVAL will be -1
	 * Here judge one-shot timer fired by checking whether TVAL is larger
	 * than TCFG
	 */
	if (ticks < cfg)
		delta = tick_to_ns(vcpu, ticks);
	else
		delta = 0;

	expire = ktime_add_ns(ktime_get(), delta);
	vcpu->arch.expire = expire;
	if (kvm_vcpu_is_blocking(vcpu)) {

		/*
		 * HRTIMER_MODE_PINNED is suggested since vcpu may run in
		 * the same physical cpu in next time
		 */
		hrtimer_start(&vcpu->arch.swtimer, expire, HRTIMER_MODE_ABS_PINNED);
	}
}
```

比较重要的是guest timer恢复流程：

```clike
/*
 * Restore soft timer state from saved context.
 */
void kvm_restore_timer(struct kvm_vcpu *vcpu)
{
	unsigned long cfg, estat;
	unsigned long ticks, delta, period;
	ktime_t expire, now;
	struct loongarch_csrs *csr = vcpu->arch.csr;

	/*
	 * Set guest stable timer cfg csr
	 * Disable timer before restore estat CSR register, avoid to
	 * get invalid timer interrupt for old timer cfg
	 */
	cfg = kvm_read_sw_gcsr(csr, LOONGARCH_CSR_TCFG);

	write_gcsr_timercfg(0);
	kvm_restore_hw_gcsr(csr, LOONGARCH_CSR_ESTAT);
	kvm_restore_hw_gcsr(csr, LOONGARCH_CSR_TCFG);
	if (!(cfg & CSR_TCFG_EN)) {
		/* Guest timer is disabled, just restore timer registers */
		kvm_restore_hw_gcsr(csr, LOONGARCH_CSR_TVAL);
		return;
	}

	/*
	 * Freeze the soft-timer and sync the guest stable timer with it.
	 */
	if (kvm_vcpu_is_blocking(vcpu))
		hrtimer_cancel(&vcpu->arch.swtimer);

	/*
	 * From LoongArch Reference Manual Volume 1 Chapter 7.6.2
	 * If oneshot timer is fired, CSR TVAL will be -1, there are two
	 * conditions:
	 *  1) timer is fired during exiting to host
	 *  2) timer is fired and vm is doing timer irq, and then exiting to
	 *     host. Host should not inject timer irq to avoid spurious
	 *     timer interrupt again
	 */
	 
	 /*
	 * 来自 LoongArch 参考手册第 1 卷第 7.6.2 章节
	 * 如果一次性定时器触发，CSR TVAL 将为 -1，有两种情况：
	 *  1) 在退出到宿主机期间定时器被触发
	 *  2) 定时器在虚拟机执行定时器中断期间被触发，然后退出到宿主机。
	 *    宿主机不应该注入定时器中断，以避免再次出现虚假的定时器中断。
	 */
	 
	ticks = kvm_read_sw_gcsr(csr, LOONGARCH_CSR_TVAL);
	estat = kvm_read_sw_gcsr(csr, LOONGARCH_CSR_ESTAT);
	if (!(cfg & CSR_TCFG_PERIOD) && (ticks > cfg)) {
		/*
		 * Writing 0 to LOONGARCH_CSR_TVAL will inject timer irq
		 * and set CSR TVAL with -1
		 */
		write_gcsr_timertick(0);

		/*
		 * Writing CSR_TINTCLR_TI to LOONGARCH_CSR_TINTCLR will clear
		 * timer interrupt, and CSR TVAL keeps unchanged with -1, it
		 * avoids spurious timer interrupt
		 */
		if (!(estat & 0x8))
			gcsr_write(CSR_TINTCLR_TI, LOONGARCH_CSR_TINTCLR);
		return;
	}

	/*
	 * Set remainder tick value if not expired
	 */
	delta = 0;
	now = ktime_get();
	expire = vcpu->arch.expire;
	if (ktime_before(now, expire))
		delta = ktime_to_tick(vcpu, ktime_sub(expire, now));
	else if (cfg & CSR_TCFG_PERIOD) {
		period = cfg & CSR_TCFG_VAL;
		delta = ktime_to_tick(vcpu, ktime_sub(now, expire));
		delta = period - (delta % period);

		/*
		 * Inject timer here though sw timer should inject timer
		 * interrupt async already, since sw timer may be cancelled
		 * during injecting intr async
		 */
		kvm_queue_irq(vcpu, INT_TI);
	}

	write_gcsr_timertick(delta);
}

```

在hvisor中添加对应的timer处理和中断注入逻辑，然后就解决了……

目前可以在不修改任何linux源码并且运行idle指令的情况下正确处理时钟和中断，shell也可以正常用了：

![Untitled](imgs/20240807_hvisor_loongarch64_port/Untitled%205.png)