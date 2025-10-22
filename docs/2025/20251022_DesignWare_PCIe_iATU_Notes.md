# DesignWare PCIe iATU介绍
时间：2025/10/22

作者：徐仲锴 (xuzhongkai25e@ict.ac.cn)

摘要：以NXP i.MX8MP的DWC驱动为例，介绍DesignWare PCIe iATU地址转换单元的功能。

# 回顾ECAM

在QEMU中PCIe使用的是标准的ECAM驱动，在设备树的PCIe节点中， `reg` 属性中给出了配置空间的内存地址段， `ranges` 属性中给出了BAR空间的CPU侧地址和PCI侧的地址映射关系。

```
	pcie@10000000 {
		// ...
		ranges = <0x1000000 0x00 0x00 0x00 0x3eff0000 0x00 0x10000 
        0x2000000 0x00 0x10000000 0x00 0x10000000 0x00 0x2eff0000 
        0x3000000 0x80 0x00 0x80 0x00 0x80 0x00>;

		reg = <0x40 0x10000000 0x00 0x10000000>;

		device_type = "pci";
		compatible = "pci-host-ecam-generic";
	};
```

这几段映射关系都是硬件已经完成了的，软件直接访问CPU侧地址就等同于访问PCI资源。例如上面给出的设备树中给出的配置空间起始地址是0x4010000000，此时想要访问 `00:01.0` 的配置空间，只需要**根据BDF生成对应的PCI地址，加上起始地址得到对应的配置空间起始地址即可**。

# DesignWare PCIe iATU

## DW

DesignWare PCIe和ECAM PCIe的不同之处在于，ECAM设备树中给出的CPU侧和PCIe侧地址映射关系是硬件已经完成了的，而且配置空间是无需映射的，无需另外处理；在DW设备树中同样会给出映射关系，**但这段映射关系必须通过配置iATU才能生效，配置iATU之后的地址访问才是有效的**。总之，在DW PCIe中，软件需要将设备树中给出的地址映射关系写到iATU中。

**另外，配置空间只给出了CPU侧的地址，那它到底应该映射到哪个PCI地址？后面我们会详细介绍，映射到哪个PCI地址取决于想要访问哪个BDF的资源，这里我们不细究，因为这个部分主要讲述如何使用iATU配置映射关系。**

```
	pcie@33800000 {
		compatible = "fsl,imx8mp-pcie\0snps,dw-pcie";
		reg = <0x00 0x33800000 0x00 0x400000 
		0x00 0x1ff00000 0x00 0x80000>;
		reg-names = "dbi\0config";
		device_type = "pci";
		ranges = <0x81000000 0x00 0x00 0x00 0x1ff80000 0x00 0x10000 
		0x82000000 0x00 0x18000000 0x00 0x18000000 0x00 0x7f00000>;
		// ...
	};
```

DW设备树中的DBI指的是RC的内部寄存器，可以用于**配置iATU**、MSI以及访问 `00:00.0`设备的资源等等，DBI的寄存器是可以直接通过内存访问到的。你可以通过 `DBI_BASE` 找到 `iATU_BASE`：

```
	// drivers/pci/controller/dwc/pcie-designware.h

/*
 * The default address offset between dbi_base and atu_base. Root controller
 * drivers are not required to initialize atu_base if the offset matches this
 * default; the driver core automatically derives atu_base from dbi_base using
 * this offset, if atu_base not set.
 */
#define DEFAULT_DBI_ATU_OFFSET (0x3 << 20)
```

明确两个概念：
- OutBound：主机端访问PCI资源，例如通过CPU侧的地址访问配置空间、BAR。
- InBound：PCI侧访问内存，例如设备DMA。

本节主要讨论OutBound部分，也就是只关心软件怎么访问PCI资源。

## iATU介绍
iATU (Internal Address Translation Unit)是RootComplex内部的地址转换单元，负责CPU侧地址和PCI侧之间的地址转换。简单来说，在iATU内部存在多个OutBound Region，每个OutBound Region代表了一段地址映射关系，这段关系的描述包括；

- CPU侧的起始地址 (源地址)
- PCI侧的起始地址 (目标地址)
- 大小
- 类型
  - CFG0：root bus资源的访问，或上游总线为root bus的资源的访问
  - CFG1：除CFG0之外的配置空间的访问
  - MEM：MEM BAR的访问
  - IO：IO BAR的访问

你可以在Linux中看到对应的register map：

```
// drivers/pci/controller/dwc/pcie-designware.h

/*
 * iATU Unroll-specific register definitions
 * From 4.80 core version the address translation will be made by unroll
 */
#define PCIE_ATU_UNR_REGION_CTRL1	0x00 // type of the region
#define PCIE_ATU_UNR_REGION_CTRL2	0x04 // write 0x80000000 to enable the config
#define PCIE_ATU_UNR_LOWER_BASE		0x08 // cpu lower addr (32bit)
#define PCIE_ATU_UNR_UPPER_BASE		0x0C // cpu upper addr (32bit)
#define PCIE_ATU_UNR_LIMIT		0x10 // region size
#define PCIE_ATU_UNR_LOWER_TARGET	0x14 // pci lower addr
#define PCIE_ATU_UNR_UPPER_TARGET	0x18 // pci upper addr
```

对于每个Region，以及不同的转换方向都有一组寄存器来设置上面这些信息，这是比较符合直觉的，在iATU中称为Unroll配置模式，配置过程是：
- 找到ATU中Region相关的地址空间的起始地址： `region_start`
- 每一组Region的配置寄存器占用的空间大小（例如0x200）： `region_bytes`
- 将要配置的Region： `region_id`
- 对应的Region的配置寄存器组的起始地址： `region_start + region_id * region_bytes`
- 进行寄存器配置
- 写这个Region的 `PCIE_ATU_UNR_REGION_CTRL2` 寄存器来表示配置完成，循环读取寄存器的值，确认生效

你可以在 `drivers/pci/controller/dwc/pcie-designware.c` 中的 `dw_pcie_prog_outbound_atu_unroll` 函数看到对应过程，在 `drivers/pci/controller/dwc/pcie-designware.h` 中有对应的寄存器定义。

但还有另一种配置模式称为ViewPort，在这个模式下，整个地址空间中只存在一组配置寄存器，同时用于多个Region以及不同转换方向的配置，但同样存在多个region，我们需要先写一个 `PCIE_ATU_VIEWPORT` 寄存器来表示接下来的配置要对哪个Region生效（注意同时必须设置OutBound / InBound），则配置过程是：
- `PCIE_ATU_REGION_OUTBOUND | region_id` -> `PCIE_ATU_VIEWPORT`
- 进行寄存器配置
- 写 `PCIE_ATU_CR2` 寄存器来表示配置完成，循环读取寄存器的值，确认生效

你可以在 `drivers/pci/controller/dwc/pcie-designware.c` 中的 `dw_pcie_prog_outbound_atu` 函数看到对应过程，这个函数是一个通用函数，它会先判断硬件是否支持Unroll，如果支持就调用上面说的 `dw_pcie_prog_outbound_atu_unroll` ，如果不支持则通过ViewPort方式配置。

对了，它是这样判断硬件支持模式的，用到上面提过的 `PCIE_ATU_VIEWPORT` 寄存器：

```
static u8 dw_pcie_iatu_unroll_enabled(struct dw_pcie *pci)
{
	u32 val;

	val = dw_pcie_readl_dbi(pci, PCIE_ATU_VIEWPORT);
	if (val == 0xffffffff)
		return 1;

	return 0;
}
```

现在我们知道iATU的作用以及如何配置了，但还有一些事情需要考虑：
- Region数量不够映射所有映射关系
- 通过内存访问PCI资源的规则

接下来我们会一一说明的。

## NR_REGION < NR_NEED_MAP

在我所使用的i.MX8MP开发板中，Linux定义了3个Region，但实际却只使用了2个Region。

而我们在设备树中看到了3段映射关系，配置空间、MEM BAR、IO BAR。这该咋办？显然是轮流使用。

讨论具体的方案之前，我们需要想想哪一段关系是常驻的。系统启动后我们会枚举所有的设备，再对设备的配置空间进行一些配置，然后驱动设备，后续就可以使用设备了，从这个过程我们可以看出来，BAR的使用率是更高的，配置空间只在最初的枚举阶段用得频繁。

所以Linux的方案是：
- Region 0：MEM BAR的映射，常驻。
- Region 1：IO BAR和配置空间的映射交替使用，IO BAR常驻，当需要使用配置空间时，换为配置空间的映射，一次访问一次设置，访问结束换回IO BAR。

所以你可以看到 `drivers/pci/controller/dwc/pcie-designware-host.c` 中 `dw_pcie_access_other_conf`是这样写的，一次配置空间的访问，需要用到3条指令：

```
	dw_pcie_prog_outbound_atu(pci, PCIE_ATU_REGION_INDEX1, type, cpu_addr,
				  busdev, cfg_size);
	if (write)
		ret = dw_pcie_write(va_cfg_base + where, size, *val);
	else
		ret = dw_pcie_read(va_cfg_base + where, size, val);

	if (pci->num_viewport <= 2)
		dw_pcie_prog_outbound_atu(pci, PCIE_ATU_REGION_INDEX1,
					  PCIE_ATU_TYPE_IO, pp->io_base,
					  pp->io_bus_addr, pp->io_size);
```

从hypervisor中可以跟踪这个过程：

```
iATU[1] OUT CFG0: CPU[0x1ff00000-0x1ff3ffff] → PCIe[0x1000000] sz=0x40000

# 配置了config和pci侧的映射关系以后开始访问数据
# 这里通过config访问到了NVMe的第一个信息
# 访问的地址是 0x1ff00000 + 0
[INFO  0] (hvisor::pci::pci:524) Reg Cfg Read: 0x0, Size: 0x4, Value: 0x2263126f

# 一次read/write指令的调用结束，都要把region变回IO BAR的配置
iATU[1] OUT IO: CPU[0x1ff80000-0x1ff8ffff] → PCIe[0x0] sz=0x10000
```

## 配置空间的访问

在上面的hypervisor的跟踪中，你可能会发现，NVMe是一个Endpoint设备，而访问的地址却是配置空间的起始地址，这个位置本应该是 `00:00.0` 的配置空间，这是因为前面我们说过 `00:00.0` 的资源是通过DBI空间访问的。

那么是代表配置空间是从 `00:01.0` 设备开始计算吗，不对， `lspci` 会看到NVMe的BDF是 `01:00.0`，那到底咋回事呢？

我们再回顾一下访问配置空间前，是如何配置iATU的：

```
iATU[1] OUT CFG0: CPU[0x1ff00000-0x1ff3ffff] → PCIe[0x1000000] sz=0x40000
```

这里我们会发现有点奇怪，BAR空间的映射关系是写到设备树中的，但配置空间只给出了CPU侧的地址，到底要映射到哪个PCI地址？

我们再观察这个PCIe地址 `0x100_0000`：

```
0x  1    0    0    0    0    0    0
0b 0001 0000 0000 0000 0000 0000 0000
   bbbb dddd dfff rrrr rrrr rrrr rrrr
```

如果将它看作某个BDF的偏移，那它就是 `01:00.0` 。

这里为什么寄存器偏移占用16位而不是12位，是笔者通过代码推测的，即下文的 `PCIE_ATU_FUNC` 宏，具体的原因笔者也很困惑，欢迎交流。

我们再来看一下具体的Linux代码：

```
	// drivers/pci/controller/dwc/pcie-designware-host.c
	// dw_pcie_access_other_conf(...)

	busdev = PCIE_ATU_BUS(bus->number) | PCIE_ATU_DEV(PCI_SLOT(devfn)) |
		 PCIE_ATU_FUNC(PCI_FUNC(devfn));

	// ...

	dw_pcie_prog_outbound_atu(pci, PCIE_ATU_REGION_INDEX1, type, cpu_addr,
			busdev, cfg_size);
```

可以看到，首先确定要访问的BDF的PCI偏移地址，接着将 `reg` 中的配置空间地址映射到对应的PCI偏移地址。所以可以得出结论，当想要访问某个BDF的配置空间时：

- 计算BDF对应的PCI偏移地址： `pci_addr`
- iATU映射： `reg.config` -> `pci_addr`
- 访问该BDF配置空间的寄存器： `reg.config + register_offset`

而在ECAM中应该是这样的： `reg.config + pci_addr + register_offset`

所以，可以总结，ECAM中给出的配置空间的内存地址段代表了所有BDF的配置空间，而在DW中给除的配置空间的内存地址段则像一个多路选择器一样，根据iATU映射到的BDF的不同，代表了不同的BDF的配置空间。

至于为什么这样设计，我猜是因为要映射所有BDF的空间就需要另外一大段内存用于映射PCI地址空间，这样的设计能够兼容不同规格的主板，这样的思想在其他方面也有体现，例如DW自带MSI的模拟机制，能使得PCIe设备在中断控制器不支持MSI的情况下也能够使用MSI，比如这篇文档使用的开发板并不支持ITS子系统，但你在 `/proc/interrupts` 依然能看到设备使用了 `PCI-MSI` 中断。

# 总结

这篇博客中我们基于具体的Linux驱动代码介绍了DW PCIe中如何通过配置iATU进行OutBound访问，其过程为：

- 确认硬件工作模式：Unroll / ViewPort
- 配置iATU
  - Region 0：MEM BAR
  - Region 1：IO BAR
- 通过DBI访问 `00:00.0` 相关资源
- 访问其他BDF资源：
  - 生成PCI地址
  - Region 1：CPU[reg.config] -> PCI[pci_addr]
  - 访问 CPU[reg.config + register_offset]
  - 恢复Region 1：IO BAR