# Jailhouse PCI 与 ivshmem 机制

时间：2023.12.28

作者：Linkun Chen

## 1 Abstract

本文档为jailhouse中vpci和ivshmem（用于虚拟机间通信的共享内存）相关代码的阅读记录。

## 2 PCI

### 2.1 **PCI 注册**

PCI在`pci.c`文件的末尾注册了unit：

```c
DEFINE_UNIT(pci, "PCI");
```

这是一个宏定义，展开之后就是：
```c
static const struct unit unit_pci __attribute__((section(".units"), aligned(8), used)) = {
    .name = "PCI",
    .init = pci_init,
    .shutdown = pci_shutdown,
    .mmio_count_regions = pci_mmio_count_regions,
    .cell_init = pci_cell_init,
    .cell_exit = pci_cell_exit,
};
```

### 2.2 **PCI 初始化**

`pci_init`是PCI的初始化函数，其是在hypervisor enable的时候完成的，调用过程为：

> `setup.c`: `init_late()` -> `unit->init()`

`pci_init`被调用之前，jailhouse会打印提示语句：

```txt
Initializing unit: PCI
```

`pci_init`完成的工作：

1. 获取`PCI MMCONFIG`的MMIO起始位置和区域大小。

    * 以qemu为例，`PCI MMCONFIG`的MMIO起始位置`mmcfg_start`为0x7000000，`mmcfg_size`为0x100000。

2. 调用`pci_cell_init`对rootcell完成pci的初始化

    * 在`cell->config`里找到关于pci配置信息，包括：`num_pci_devices`以及`jailhouse_pci_device`。
    
    * 给上文1.提到的`PCI MMCONFIG`区域注册mmio处理函数`pci_mmconfig_access_handler`。
    
    * 从mempool中分配一定量的内存，用于在cell结构体里保存pci设备的信息（`cell->pci_devices`）。

    * 遍历`cell->config`里pci的配置信息，对每个设备而言：

        * 如果这个设备是共享内存的话（`JAILHOUSE_PCI_TYPE_IVSHMEM`），那么调用`ivshmem_init`对共享内存设备做初始化

        * 否则，进行物理设备的相关操作

3. 初始化共享内存`ivshmem_init`（ivshemem, 这里的iv是Inter-VM，即虚拟机间的意思）

	* 找到在cell config里面定义的共享内存mem区域（`dev_info->shmem_region`）
	
	* 获取当前要注册的ivshmem设备的bdf（在cell config中定义），检查当前的共享内存表（ivshmem_list），判断里面是否已经有和当前bdf相同的ivshmem。
	
	* 若无，则说明hypervisor要注册一个全新的ivshmem，这是情况1
	
	* 否则，需要将这个要注册的设备和已有的ivshmem关联起来，这是情况2（回头再看）
	
	* 每个ivshmem有两个“端点”，编号为0和1，两边分别对应两个虚拟机。此时我们选择本次注册设备在ivshmem中的端点序号（情况1下是0，情况2下是1），然后给对应端点填上相应信息。
	
	* 调用`pci_reset_device`重置这个ivshmem device的状态

### 2.3 **Jailhouse driver(EL1) 对 PCI 的后处理**

**create_vpci_of_overlay**

* 找到gic节点、gic地址cell大小，以及gic_phandle

```c
	gic = of_find_matching_node(NULL, gic_of_match);
	
	if (of_property_read_u32(gic, "#address-cells", &gic_address_cells) < 0)
		gic_address_cells = 0;
	gic_phandle = gic->phandle;
	
	of_node_put(gic); // 移除gic节点的引用
```

* 将`vpci_template.dtb`中的设备树片段应用于当前设备树。

```c
	if (of_overlay_fdt_apply(__dtb_vpci_template_begin,
			__dtb_vpci_template_end - __dtb_vpci_template_begin,
			&overlay_id) < 0)
		return false;
```

```arm
__dtb_vpci_template_begin:
.incbin "/home/clk/workspace/jailhouse/driver/vpci_template.dtb" 
__dtb_vpci_template_end:
.global __dtb_vpci_template_end
```

	如下是`vpci_template.dts`设备树片段：

```devicetree
/dts-v1/;
/ {
	fragment {
		target-path = "/";
		__overlay__ {
			#address-cells = <2>;
			#size-cells = <2>;

			pci@0 {
				compatible = "pci-host-ecam-generic";
				device_type = "pci";
				bus-range = <0 0>;
				#address-cells = <3>;
				#size-cells = <2>;
				#interrupt-cells = <1>;
				interrupt-map-mask = <0 0 0 7>;
				ranges = <0 0 0 0 0 0 0>;
				status = "disabled";
			};
		};
	};
};
```

这里我觉得就是为给dts插入了一个pci节点，然后对这个节点的配置进行一些修改（比如地址改成pci_mmcfg的起始地址，大小也改），然后应用配置，cpu会对新增的pci区域做一个扫描。

### 2.4 **Handle PCI Dabt**

#### 2.4.1 **查询bdf值**

前面提到我们给`MMCFG`区域注册了MMIO处理函数`pci_mmconfig_access_handler`，现在当我们完成hypervisor enable返回EL1时，会触发对pci mmcfg区域的一系列访问，并触发这个处理函数（由`create_vpci_of_overlay` 11 ~ 12触发，具体原因未知，尚需探索）

`pci_mmconfig_access_handler`完成的工作：

* 计算bdf和reg值：

	```
		u32 reg_addr = mmio->address & 0xfff;
		u16 bdf = mmio->address >> 12;
	```
	
	每个pci设备占据一页的大小，且由bdf（bus+device+function）唯一标识。

> 在PCI总线结构中，BDF表示了设备在总线上的位置。具体来说，BDF分别代表Bus Number（总线编号）、Device Number（设备编号）和Function Number（功能编号）。它们的位数如下：
> * 总线编号（Bus Number）通常占据8位，因此它的取值范围是0-255。
> * 设备编号（Device Number）通常占据5位，因此它的取值范围是0-31。
> * 功能编号（Function Number）通常占据3位，因此它的取值范围是0-7。
> 
> 综合起来，BDF编码的格式是B:D:F，其中B占8位，D占5位，F占3位，标识为xx:xx:x

	具体而言，我们可以按如下方式拆分地址0x7000000：

>	地址： 0x7000000 -> 0x0000000 （相对起始地址偏移量）  
>   拆分： 0x00(b)    00(d+f)    000(reg)

#### 2.4.2 **PCI配置空间**

[PCI Configuration Space - Wikipedia](https://en.wikipedia.org/wiki/PCI_configuration_space)

PCI配置空间如图所示：

![pci mmcfg space](img/20231209_jailhouse-pci-mmcfg-space.png)

> 设备ID（Device ID）和供应商ID（Vendor ID）寄存器用于识别设备（如集成电路），通常被称为PCI ID。16位的供应商ID由PCI-SIG分配。然后，供应商为其分配16位的设备ID。[Device ID & Vendor ID查询网址](https://web.archive.org/web/20080209182956/http://www.pcidatabase.com/index.php)

>
> 状态寄存器用于报告支持的功能以及是否发生了某些类型的错误。
>
> 命令寄存器包含一个位掩码，用于单独启用和禁用各种功能。
>
> Header Type寄存器的值根据设备的功能确定了头部剩余48字节（64-16）的不同布局。即，用于根复杂型、交换机和桥接器的类型1头部。然后用于端点的类型0头部。在设备被告知可以使用内存写入和无效事务之前，必须编程缓存行大小寄存器。这通常应与CPU的缓存行大小匹配，但正确的设置取决于系统。该寄存器不适用于PCI Express。
>
> 子系统ID（Subsystem ID）和子系统供应商ID（Subsystem Vendor ID）用于区分特定型号（如插卡）。虽然供应商ID是芯片组制造商的ID，但子系统供应商ID是卡片制造商的ID。子系统ID由子系统供应商从与设备ID相同的号码空间分配。例如，在无线网络卡的情况下，芯片制造商可能是Broadcom或Atheros，而卡片制造商可能是Netgear或Hewlett-Packard。通常，供应商ID-设备ID组合指定主机应加载哪个驱动程序来处理设备，因为具有相同的VID:DID组合的所有卡可以由同一个驱动程序处理。子系统供应商ID-子系统ID组合标识了卡片，这是驱动程序可以使用的信息，以在其操作中应用次要的特定于卡片的更改。

#### 2.4.3 **PCI配置空间读写**

`pci_mmconfig_access_handler`会根据mmio为读还是为写进入如下两个分支(`pci_cfg_write_moderate` or `pci_cfg_read_moderate`)：

```c
	if (mmio->is_write) {
		result = pci_cfg_write_moderate(device, reg_addr, mmio->size,
						mmio->value);
		if (result == PCI_ACCESS_REJECT)
			goto invalid_access;
	} else {
		result = pci_cfg_read_moderate(device, reg_addr, mmio->size,
					       &val);
		if (result != PCI_ACCESS_PERFORM)
			mmio->value = val;
	}
	if (result == PCI_ACCESS_PERFORM)
		mmio_perform_access(pci_space, mmio);

	return MMIO_HANDLED;
```

#### 2.4.4 **针对ivshmem的PCI配置空间读写**

在`pci_cfg_read_moderate`中定义了针对ivshmem的pci设备的读取处理函数`ivshmem_pci_cfg_read`。

```c
	if (device->info->type == JAILHOUSE_PCI_TYPE_IVSHMEM) {
		return ivshmem_pci_cfg_read(device, address, value);
	}
```

`ivshmem_pci_cfg_read`将读请求重定向到ivshmem device的endpoint里的cspace上：

```c
enum pci_access ivshmem_pci_cfg_read(struct pci_device *device, u16 address,
				     u32 *value)
{
	struct ivshmem_endpoint *ive = device->ivshmem_endpoint;

	if (address < sizeof(default_cspace))
		*value = ive->cspace[address / 4] >> ((address % 4) * 8);
	else
		*value = -1;
	return PCI_ACCESS_DONE;
}
```

写请求同理。

### 2.5 **IVSHMEM Design**

* jailhouse做了一个vpci处理机制（类似vgic），每个cell都抽象出了一些vpci设备
* jailhouse会读取cell->config获取cell拥有哪些pci设备，并特判设备是否是ivshmem，分情况做不同处理。
	目前config里面全都是ivshmem设备，还没有真实的物理设备。

* jailhouse将ivshmem视为一个虚拟的pci设备
* hypervisor enable之后，jailhouse将修改root cell的设备树使内核能够识别这个新的ivshmem设备
* cell->config的pci field给出了基本的ivshmem信息

* ivshmem可以分为四个区域：
	* pci-mmcfg区域（需hypervisor dabt介入处理）
		由cell->config决定区域位置（pci配置区域起始地址 + bdf * 0x1000），在qemu-virt-arm64中起始于0x7000000
		保存着ivshmem的一些基本信息（如是否支持msix，和以下几个区域的起始地址及大小）
		EL1可以通过pci驱动读写这个ivshmem设备的基本值
	
		Header Registers

		| Offset | Register               | Content                                              |
		|-------:|:-----------------------|:-----------------------------------------------------|
		|    00h | Vendor ID              | 110Ah                                                |
		|    02h | Device ID              | 4106h                                                |
		|    04h | Command Register       | 0000h on reset, writable bits are:                   |
		|        |                        | Bit 0: I/O Space (if Register Region uses I/O)       |
		|        |                        | Bit 1: Memory Space (if Register Region uses Memory) |
		|        |                        | Bit 3: Bus Master                                    |
		|        |                        | Bit 10: INTx interrupt disable                       |
		|        |                        | Writes to other bits are ignored                     |
		|    06h | Status Register        | 0010h, static value                                  |
		|        |                        | In deviation to the PCI specification, the Interrupt |
		|        |                        | Status (bit 3) is never set                          |
		|    08h | Revision ID            | 00h                                                  |
		|    09h | Class Code, Interface  | Protocol Type bits 0-7                               |
		|    0Ah | Class Code, Sub-Class  | Protocol Type bits 8-15                              |
		|    0Bh | Class Code, Base Class | FFh                                                  |
		|    0Eh | Header Type            | 00h                                                  |
		|    10h | BAR 0                  | MMIO or I/O register region                          |
		|    14h | BAR 1                  | MSI-X region                                         |
		|    18h | BAR 2 (with BAR 3)     | optional: 64-bit shared memory region                |
		|    2Ch | Subsystem Vendor ID    | same as Vendor ID, or provider-specific value        |
		|    2Eh | Subsystem ID           | same as Device ID, or provider-specific value        |
		|    34h | Capability Pointer     | First capability                                     |
		|    3Eh | Interrupt Pin          | 01h-04h, must be 00h if MSI-X is available           |

	* shared-memory区域（直通）
		在两个cell之间共享，用于传输信息

	* ivshmem-registers区域（需hypervisor dabt介入处理）
		| Offset | Register          |
		|-------:|:------------------|
		|    00h | ID                |
		|    04h | Maximum Peers     |
		|    08h | Interrupt Control |
		|    0Ch | Doorbell          |
		|    10h | State             |

		Doorbell可以用于给peer cell发送中断（先通过dabt进入hypervisor，hypervisor再向peer-cell注入中断）

	* msix-table区域
		qemu-arm64貌似不支持msix，暂时略过

* ivshmem是异步通信，写完shared-memory之后直接发送中断（写doorbell）就可以直接返回，不必等待peer-cell回应
* jailhouse v0.10中只有x86的ivshmem-demo，但是最新版本的jailhouse已经可以运行arm64版本的ivshmem-demo了，虽然目前运行时存在问题：ivshmem-demo会向peer 2发送信息，但是peer 2并不存在
* 最新版本jailhouse的ivshmem十分复杂，能够支持多个peer-cell之间的通信

