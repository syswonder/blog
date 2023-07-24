# Jailhouse中的Hypercall整理

时间：2023.7.21

作者：郑元昊

#### cell结构体（动态修改）

```c
struct cell {
	union {
		// 与其他cell共享的内存区域
		struct jailhouse_comm_region comm_region;
		// 填充成完整页大小的步长
		u8 padding[PAGE_SIZE];
	} __attribute__((aligned(PAGE_SIZE))) comm_page;

	// 与架构相关的域定义
	struct arch_cell arch;

	// 用于存储cell相关的状态与配置数据的页数量
	unsigned int data_pages;
	// 指向cell静态信息描述的指针
	struct jailhouse_cell_desc *config;

	// 指向cell的cpu集的指针
	struct cpu_set *cpu_set;
	// 如果cpu集比较小的话直接存储
	struct cpu_set small_cpu_set;

    // 如果cell能被root cell加载的话设置为True
	bool loadable;

    // 指向下一个cell的指针
	struct cell *next;

    // 分配给这个cell的pci设备的链表
	struct pci_device *pci_devices;

    // 保护mmio_locations, mmio_handlers, num_mmio_regions修改的锁
	spinlock_t mmio_region_lock;
	// mmio_locations, mmio_handlers, num_mmio_regions 的生成计数器
	volatile unsigned long mmio_generation;
	// MMIO 区域描述表
	struct mmio_region_location *mmio_locations;
	// MMIO 区域处理表
	struct mmio_region_handler *mmio_handlers;
	// 使用的 MMIO 区域数
	unsigned int num_mmio_regions;
	// MMIO 区域数的最大值
	unsigned int max_mmio_regions;
};

// arch/arm/cell.h
struct arch_cell {
	struct paging_structures mm;

	u32 irq_bitmap[1024/32];

	struct {
		u8 ent_count;
		struct pvu_tlb_entry *entries;
	} iommu_pvu; // pvu是一种iommu，进行二阶段的映射
};
```

#### cell_desc结构体（config文件中的静态配置）

```c
struct jailhouse_cell_desc {
	char signature[5];
	__u8 architecture;
	__u16 revision;

	char name[JAILHOUSE_CELL_NAME_MAXLEN+1];
	__u32 id; // 由driver分配
	__u32 flags;

	__u32 cpu_set_size;
	__u32 num_memory_regions;
	__u32 num_cache_regions;
	__u32 num_irqchips;
	__u32 num_pio_regions;
	__u32 num_pci_devices;
	__u32 num_pci_caps;
	__u32 num_stream_ids;

	__u32 vpci_irq_base;

	__u64 cpu_reset_address;
	__u64 msg_reply_timeout;

	struct jailhouse_console console;
} __attribute__((packed));
```



## HC_CELL_CREATE

driver：

1. 从用户空间复制新cell配置的参数`jailhouse_cell_desc`结构体，并申请一段内存存储配置信息
2. 校验新cell的`signature`、`revision`、`architecture`字段
3. 配置新cell的标志位支持debug输出的hypercall

```c
/* CONSOLE_ACTIVE implies CONSOLE_PERMITTED for non-root cells */
if (CELL_FLAGS_VIRTUAL_CONSOLE_ACTIVE(config->flags))
	config->flags |= JAILHOUSE_CELL_VIRTUAL_CONSOLE_PERMITTED;
```

4. 检查并确保没有其他进程进行`cell_create`操作（获取jailhouse互斥锁）、已完成`jailhouse_enable`操作
5. 配置新cell的`id`和`name`字段，确保`id`唯一
6. 为新cell的`cell`结构体分配内存
7. 初始化`cell`的`entry`链表字段
8. 将新cell配置结构体`cell_desc`的参数配置给新cell的`cell`结构体
9. 新cell对应pci设备和文件系统sysfs的初始化操作
10. 检查并确保新cell的cpu集是`root cell`的cpu集的子集
11. 调用Linux的`cpu_down()`函数和`cpumask_set_cpu()`函数将分配给新cell的cpu脱机，之后linux不会在这些cpu上调度进程
12. 将新cell需要的pci设备从`root cell`中分离
13. 调用hypercall`JAILHOUSE_HC_CELL_CREATE`，传递一个指向结构体`jailhouse_cell_desc`的指针，该指针将新cell描述为它的参数，虚拟机从`root cell`回到hypervisor

hypervisor：

14. 检查所在cell，只支持在`root cell`操作
15. 挂起`root cell`包含的cpu（除当前cpu）
16. 检查并确保每个non root cell的状态都不为`JAILHOUSE_CELL_RUNNING_LOCKED`
17. 计算新cell配置信息结构体`jailhouse_cell_desc`所占内存页数并将cell的页依次映射到hypervisor地址空间
18. 检查并确保新cell的`name`和`id`字段与其他cell都不同
19. 分配储存新cell配置信息的内存
20. 配置新cell的`cpu_set`结构体字段
21. 为新cell做mmio初始化，分配一段给mmio用的空间，配置cell中mmio对应的字段
22. 检查并确保当前运行的cpu不是要分给新cell的cpu
23. 检查并确保新cell的cpu集是`root cell`的cpu集的子集
24. 对每个unit做初始化
25. 将分配给新cell的cpu设为`park`状态，从root cell中移除、分配给新cell、将状态清除
26. 将新cell的内存区域撤销从`root cell`里的映射并重新映射到新cell中
27. `config_commit`刷新`root cell`和新cell的cache
28. 将cell状态设置为`JAILHOUSE_CELL_SHUT_DOWN`
29. 将新cell添加到cell的链表上

driver：

30. `cell_register`添加新cell的信息





## HC_CELL_SET_LOADABLE

driver：

1. 从用户空间复制新cell加载的参数，包括要加载的新cell的镜像
2. 获取jailhouse互斥锁，检查并确保已完成`jailhouse_enable`操作
3. 检查并确保当前`id`对应的cell已经创建
4. 调用hypercall`JAILHOUSE_HC_CELL_SET_LOADABLE`，参数为新cell的id，虚拟机从root cell回到hypervisor

hypervisor：

5. 检查所在cell，只支持在`root cell`操作
6. 挂起`root cell`包含的cpu（除当前cpu）
7. 通过cell的id找到新cell，检查该cell不为空且不是`root cell`，若不合法，则报错
8. 挂起新cell包含的cpu（除当前cpu）

```c
/*
 * Suspend all CPUs assigned to the cell except the one executing
 * the function (if it is in the cell's CPU set) to prevent races.
 */
static void cell_suspend(struct cell *cell) {
	unsigned int cpu;

	for_each_cpu_except(cpu, cell->cpu_set, this_cpu_id())
		suspend_cpu(cpu);
}

static void suspend_cpu(unsigned int cpu_id) {
    // ...
    
	spin_lock(&target_data->control_lock);

	target_data->suspend_cpu = true;
	target_suspended = target_data->cpu_suspended;

	/*
	 * Acts as memory barrier on certain architectures to make suspend_cpu
	 * visible. Otherwise, arch_send_event() will take care of that.
	 */
	spin_unlock(&target_data->control_lock);

    // ...
}
```

9. 将新cell包含的cpu设为park状态，`failed`字段设为false

```c
for_each_cpu(cpu, cell->cpu_set) {
	public_per_cpu(cpu)->failed = false;
	arch_park_cpu(cpu);
}

// arch/arm/...
void arch_park_cpu(unsigned int cpu_id) {
	public_per_cpu(cpu_id)->park = true;

	resume_cpu(cpu_id); // 解除suspend状态
}
```

10. 将新cell状态设置为`JAILHOUSE_CELL_SHUT_DOWN`，将`loadable`字段设置为true
11. `pci_cell_reset`重置新cell使用的pci设备，重置MSI和MSIX等与中断相关的配置
12. 将加载新镜像区域的内存添加stage2地址映射

```c
/* map all loadable memory regions into the root cell */
for_each_mem_region(mem, cell->config, n)
	if (mem->flags & JAILHOUSE_MEM_LOADABLE) {
		err = remap_to_root_cell(mem, ABORT_ON_ERROR);
		if (err)
			goto out_resume;
}
```

13. `config_commit(NULL)`刷新`root cell`的cache
14. `cell_resume(&root_cell)`恢复`root cell`（解除`root cell`包含cpu的挂起状态）

driver：

15. 调用`load_image`函数加载镜像



## HC_CELL_START

driver：

1. 从用户空间复制内存，获取目标cell的`id`字段
2. 获取jailhouse互斥锁，检查并确保已完成`jailhouse_enable`操作
3. 检查并确保当前`id`对应的cell已经创建
4. 调用hypercall`JAILHOUSE_HC_CELL_START`，参数为新cell的id，虚拟机从root cell回到hypervisor

hypervisor：

5. 检查所在cell，只支持在`root cell`操作
6. 挂起`root cell`包含的cpu（除当前cpu）
7. 通过cell的id找到新cell，检查该cell不为空且不是`root cell`，若不合法，则报错
8. 挂起新cell包含的cpu（除当前cpu）
9. 撤销加载新cell镜像区域的stage2地址映射

```c
if (cell->loadable) {
	/* unmap all loadable memory regions from the root cell */
	for_each_mem_region(mem, cell->config, n)
		if (mem->flags & JAILHOUSE_MEM_LOADABLE) {
			err = unmap_from_root_cell(mem);
			if (err)
				goto out_resume;
		}

	config_commit(NULL);

	cell->loadable = false;
}
```

10. `config_commit(NULL)`刷新`root cell`的cache
11. 将`loadable`字段设置为false
12. 将共享内存区域（用做通信）初始化，并设置标志位

```c
comm_region = &cell->comm_page.comm_region;
memset(&cell->comm_page, 0, sizeof(cell->comm_page));

comm_region->revision = COMM_REGION_ABI_REVISION;
memcpy(comm_region->signature, COMM_REGION_MAGIC,
	    sizeof(comm_region->signature));

if (CELL_FLAGS_VIRTUAL_CONSOLE_PERMITTED(cell->config->flags))
	comm_region->flags |= JAILHOUSE_COMM_FLAG_DBG_PUTC_PERMITTED;
if (CELL_FLAGS_VIRTUAL_CONSOLE_ACTIVE(cell->config->flags))
	comm_region->flags |= JAILHOUSE_COMM_FLAG_DBG_PUTC_ACTIVE;
comm_region->console = cell->config->console;
comm_region->pci_mmconfig_base =
	system_config->platform_info.pci_mmconfig_base;
```

13. `pci_cell_reset`重置新cell使用的pci设备，重置MSI和MSIX等与中断相关的配置
14. 重启新cell包含的cpu

```c
for_each_cpu(cpu, cell->cpu_set) {
	public_per_cpu(cpu)->failed = false;
	arch_reset_cpu(cpu);
}

// arch/arm/...
void arch_reset_cpu(unsigned int cpu_id) {
	public_per_cpu(cpu_id)->reset = true;

	resume_cpu(cpu_id);
}
```

15. `cell_resume(&root_cell)`恢复`root cell`（解除`root cell`包含cpu的挂起状态）



## HC_CELL_DESTROY

driver：

1. 从用户空间复制内存，获取目标cell的`id`字段
2. 获取jailhouse互斥锁，检查并确保已完成`jailhouse_enable`操作
3. 检查并确保当前`id`对应的cell已经创建
4. 调用hypercall`JAILHOUSE_HC_CELL_DESTROY`，虚拟机从`root cell`回到hypervisor

hypervisor：

5. 检查所在cell，只支持在`root cell`操作
6. 挂起`root cell`包含的cpu（除当前cpu）
7. 通过cell的id找到新cell，检查该cell不为空且不是`root cell`，若不合法，则报错
8. 挂起新cell包含的cpu（除当前cpu）
9. 将新cell状态设置为`JAILHOUSE_CELL_SHUT_DOWN`
10. 把cell的cpu设置为park状态，归还给`root cell`

```c
for_each_cpu(cpu, cell->cpu_set) {
	arch_park_cpu(cpu);

	set_bit(cpu, root_cell.cpu_set->bitmap);
	public_per_cpu(cpu)->cell = &root_cell;
	public_per_cpu(cpu)->failed = false;
	memset(public_per_cpu(cpu)->stats, 0,
		sizeof(public_per_cpu(cpu)->stats));
}
```

11. 将cell的内存页归还给`root cell` 
12. `config_commit`刷新`root cell`和新cell的cache
13. 将cell从cell list中删除

```c
previous = &root_cell;
while (previous->next != cell)
	previous = previous->next;
previous->next = cell->next;
num_cells--;
```

14. 将cell配置信息用到的内存页释放
15. `cell_resume(&root_cell)`恢复`root cell`（解除`root cell`包含cpu的挂起状态）

driver：

16. 维护`root cell`的cpu集信息
17. 将该cell中的pci设备释放
18. `cell_delete`删除cell的相关信息（与`cell_register`相对）
19. 归还jailhouse互斥锁



## HC_HYPERVISOR_GET_INFO

根据`Hypervisor information type`返回对应字段

```c
static long hypervisor_get_info(struct per_cpu *cpu_data, unsigned long type) {
    switch (type) {
        case JAILHOUSE_INFO_MEM_POOL_SIZE:
            return mem_pool.pages; // hypervisor管理的物理页数量
        case JAILHOUSE_INFO_MEM_POOL_USED:
            return mem_pool.used_pages; // hypervisor正在使用的物理页数量
        case JAILHOUSE_INFO_REMAP_POOL_SIZE:
            return remap_pool.pages; // hypervisor映射的虚拟页数量
        case JAILHOUSE_INFO_REMAP_POOL_USED:
            return remap_pool.used_pages; // hypervisor正在使用的虚拟页数量
        case JAILHOUSE_INFO_NUM_CELLS:
            return num_cells; // cell数量
        default:
            return -EINVAL; // invalid error argument
    }
}
```



## HC_CELL_GET_STATE

只支持在`root cell`操作，返回对应cell的状态字段

```c
for_each_cell(cell)
	if (cell->config->id == id) { // 判断是否是所查询的cell
		u32 state = cell->comm_page.comm_region.cell_state;

		switch (state) {
			case JAILHOUSE_CELL_RUNNING: // 正在运行
			case JAILHOUSE_CELL_RUNNING_LOCKED: // 正在运行但不支持shrink
			case JAILHOUSE_CELL_SHUT_DOWN:
			case JAILHOUSE_CELL_FAILED:
			case JAILHOUSE_CELL_FAILED_COMM_REV:
				return state;
			default:
				return -EINVAL;
		}
	}
return -ENOENT; // Error NO such an ENTry
```



## HC_CPU_GET_INFO

只支持在`root cell`或该cpu属于当前cell时操作，返回对应cpu的字段

```c
if (type == JAILHOUSE_CPU_INFO_STATE) {
	return public_per_cpu(cpu_id)->failed ? JAILHOUSE_CPU_FAILED :
		JAILHOUSE_CPU_RUNNING;
} else if (type >= JAILHOUSE_CPU_INFO_STAT_BASE &&
	type - JAILHOUSE_CPU_INFO_STAT_BASE < JAILHOUSE_NUM_CPU_STATS) {
	type -= JAILHOUSE_CPU_INFO_STAT_BASE;
	return public_per_cpu(cpu_id)->stats[type] & BIT_MASK(30, 0);
} else
	return -EINVAL;
```

## HC_DEBUG_CONSOLE_PUTC

向控制台打印debug信息
