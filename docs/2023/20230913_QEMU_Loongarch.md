# QEMU Loongarch sysHyper移植

wheatfox 2023.9
enkerewpo@hotmail.com

## QEMU环境配置

本机环境：

![Untitled](QEMU%20Loongarch/Untitled.png)

参考仓库：

[https://github.com/foxsen/qemu-loongarch-runenv](https://github.com/foxsen/qemu-loongarch-runenv)

缺少依赖：`libnettle.so.7`

apt检查libnettle8发现已安装，说明版本过高而缺少7版本的apt安装包（Ubuntu 22的仓库找不到libnettle7）,手动下载deb进行gdebi安装：

```bash
wget http://security.ubuntu.com/ubuntu/pool/main/n/nettle/libnettle7_3.5.1+really3.5.1-2ubuntu0.2_amd64.deb
sudo gdebi libnettle7_3.5.1+really3.5.1-2ubuntu0.2_amd64.deb
```

成功启动Mini Linux：

![Untitled](QEMU%20Loongarch/Untitled%201.png)

```bash
Run a loongarch virtual machine.

Syntax: run_loongarch.sh [-b|c|d|D|g|h|i|k|m|q]
options:
b <bios>    use this file as BIOS
c <NCPU>    simulated cpu number
d use -s to listen on 1234 debug port
D use -s -S to listen on 1234 debug port and stop waiting for attach
g     Use graphic UI.
h     Print this Help.
i <initrd> use this file as initrd
k <kernel> use this file as kernel
m <mem> specify simulated memory size
q <qemu> use this file as qemu
```

使用图形启动：

![Untitled](QEMU%20Loongarch/Untitled%202.png)

## 测试qemu-loongarch-runenv/devel分支

切换到devel分支，运行：

```bash
docker build -t qemu-la .
docker run -it qemu-la /bin/bash
./setup.sh
./run.sh
```

![Untitled](QEMU%20Loongarch/Untitled%203_1.png)

其会依次进行cross compile工具包下载、qemu编译、tianocore(UEFI)编译、linux-v6.1.4内核编译（编译时均指定target arch为`loongarch64-unknown-linux-gnu`）。

在运行`run.sh`发生`-append未找到错误`，在上一行末尾还需要再手动加一个`\`，如图：

![Untitled](QEMU%20Loongarch/Untitled%204_1.png)

之后就可以启动loongarch64-linux-v6.1.4：

![Untitled](QEMU%20Loongarch/Untitled%205_1.png)

打印一下CPU信息：

![Untitled](QEMU%20Loongarch/Untitled%206_1.png)

## QEMU支持情况

模拟设备CPU: Loongson3A5000

qemu目前实现了virtio设备、PCIe控制器、UART口、实时时钟和电源管理端口。未实现：

1. Address Routing Registers
2. IOCSR只能用iocsr指令访问，不能通过MMIO
3. 部分IOCSR未实现：所有Chip Configuration Register和Legacy Interrupts（手册第4章和第11章）
4. 未实现GPIO
5. 未实现温度传感器
6. 未实现DDR4 SDRAM控制器
7. 未实现HyperTransport控制器
8. 未实现UART1, SPI, I2C

## 中断控制

中断路径: `sources -> level3 -> level 2 -> level 1`

```
- level 1: CPU core interrupt controller(HWI0-HWI7)
- level 2: extended Interrupt controller(extioi_pic in qemu, 256 irqs)
- level 3: LS7A1000 interrupt controller(pch_pic in qemu, 64 irqs; pch_msi, 192 irqs, refer to 7A1000 user manual)
- interrupt sources:
    - 4 PCIE interrupt output connect to pch_pic 16,17,18,19
    - LS7A UART/RTC/SCI connect to pch_pic 2/3/4
```

### 有关CPU核中断控制器

有些CPU产生的中断直接连接本控制器，如Stable Timer、Performance Counter、IPI等。

### 有关拓展IO中断控制

1. 每个拓展IO中断irq被映射到CPU核中断控制器HWI0-7
    - 默认所有256个irq被路由到HWI0(CPU int2)
    - 默认情况下所有irq路由到node 0 core 0
    - 不支持轮转分发
2. qemu实现情况：
    - 表11-6. iocsr 0x420 只实现了读，写无效. EXT_INT_en默认打开
    - 表11-7. 基本上总是enabled
    - 表11-8. 可读写，但不会生效
    - 表11-9, 未实现
    - 表11-10, 实现
    - 表11-12, 实现
    - 表11-13/14/15 基本实现，但不支持轮转

### LS7A1000中断控制器

1. 所有pch-pic irq被映射到extioi_pic irq，使用的是HT Message Interrupt向量（offset `0x200-0x238`），默认所有irq映射到extioi input 0
2. qemu实现情况：
    - offset 0/0x20/0x60/0x80/0x380/0x3a0/0x3e0 实现
    - offset 0x40/0xc0/0xe0/0x100-0x13f, 可读写，但不会生效
    - offset 0x200-0x23f, 默认全零
    - offset 0x300/0x320, 未实现

### 如何打开UART中断

io port at `0x1fe001e0`

1. LS7A1000中断控制器：
    - unmask input pin2 (bit 2 of reg at `0x20`)
    - set edge trigger or level trigger mode(`0x60`)
    - set ht message interrupt vector (byte at offset `0x202` equal the target extioi irq number)
2. 设置extioi
    - enable mapped extioi irq
3. 设置cpu core irq
4. 打开global irq

### 当UART irq触发时的操作

1. 响应cpu中断控制器
    - 需要清除中断源，即下一步的动作
2. 响应extioi中断控制器
    - iocsr `0x1800-`写入1
3. 响应LS7A1000中断控制器
    - 写`1<<irq`到intclr寄存器
4. 接收串口字符

# Loongarch手册笔记

[loongsonlab/qemu](https://gitee.com/loongsonlab/qemu)

[龙芯架构文档](https://loongson.github.io/LoongArch-Documentation/README-CN.html)

[在x86平台体验龙芯LoongArch--使用Qemu-7.2安装LoongArch版的ArchLinux_小菜刀_的博客-CSDN博客](https://blog.csdn.net/mxcai2005/article/details/129631722)

[qemu运行虚拟机无反应，只输出一行提示信息:VNC server running on 127.0.0.1:5900_Imagine Miracle的博客-CSDN博客](https://blog.csdn.net/qq_36393978/article/details/118353939?ydreferer=aHR0cHM6Ly93d3cuZ29vZ2xlLmNvbS8=)

编译安装支持Loongarch64的LLVM16+Clang16（Ubuntu22只能apt安装到14）

![Untitled](QEMU%20Loongarch/Untitled%203.png)

[https://github.com/sunhaiyong1978/CLFS-for-LoongArch](https://github.com/sunhaiyong1978/CLFS-for-LoongArch)

[国产loongarch64（龙芯）GCC的初体验](http://www.taodudu.cc/news/show-6268324.html?action=onClick)

之后我下载了官方的cross compile工具包，编译简单的loongarch64程序成功，这里使用的qemu是我自己编译的版本（Github最新release版src），编译时只打开了`qemu-loongarch64`的target。

![Untitled](QEMU%20Loongarch/Untitled%204.png)

![Untitled](QEMU%20Loongarch/Untitled%205.png)

查看生成的loongarch汇编：

![Untitled](QEMU%20Loongarch/Untitled%206.png)

## 要点整理

1. **龙芯架构组成**
    1. 龙芯基础指令集（Loongson Base）
    2. 二进制翻译扩展（LBT）
    3. 向量扩展（LSX）
    4. 高级向量扩展（LASX）
    5. 虚拟化扩展（LVZ）
    
2. 控制状态寄存器（Control and Status Register, **CSR**）
    在实现虚拟化拓展的情况下，处理器中会有两套CSR，一套属于Host，一套属于Guest。

3. **龙芯存储访问类型**
    1. 一致可缓存（Coherent Cached）**CC**
    *最终存储对象+缓存*
    2. 强序非缓存（Strongly-ordered UnCached）**SUC**
    *最终存储对象*
    满足顺序一致性，访存操作依次执行
    3. 弱序非缓存（Weakly-ordered UnCached）**WUC**
    *最终存储对象*
    读访问允许推测执行，写访问则可以内部合并为更大规模，如一个Cache行，之后采用Burst方式写入
    
    其类型在页表项的MAT（Memory Access Type）中标记，龙芯架构要求只有SUC类型的访存指令不能用副作用，即不可推测地执行，通常用于访问IO设备。但是，龙芯架构允许SUC的**取指**操作有副作用。
    
    WUC通常用于加速非缓存内存数据，如显存。
    
4. **原子访存指令**
    AM*, LL, SC
    原子地进行“读+修改+写”操作

5. **同步操作指令**
    龙芯的存储一致性为弱一致性（Weakly Consistency）WC
    使用同步操作保护写共享单元，保证多个处理器核对写共享单元的访问是互斥的
    `dbar`,`ibar`,带有dbar功能的AM原子访存指令,`LL-SC`指令对

6. **龙芯特权架构**
   
    1. 特权等级
    `PLV0-PLV3`，其中`PLV0`权限最高，可以执行特权指令
    `CSR.CRMD.PLV`域指示当前运行在哪个特权等级
    2. CSR访问指令
        1. `csrrd` 将CSR的值写入rd
        2. `csrwr` 将rd的值写入CSR，同时将CSR的旧值写入rd
        3. `csrxchg` 根据rj中的掩码信息，只将rd写入CSR对于掩码为1的位置，其余值不变，同时将CSR的旧值写入rd
    3. 上述指令中需要`csr_num`由14位立即数给出。（0号CSR的csr_num=0，依此类推）
    4. IOCSR访问指令
        1. `iocsrrd` IOCSR采用直接地址映射方式，地址来自寄存器rj
        2. `iocsrwr`
    5. IOCSR寄存器通常可以被多个处理器core同时访问，多个处理器核上的IOCSR访问指令满足顺序一致性（[https://blog.laisky.com/p/sequential-consistent/](https://blog.laisky.com/p/sequential-consistent/)）*sequential consistency*。
    [http://zhangtielei.com/posts/blog-distributed-strong-weak-consistency.html](http://zhangtielei.com/posts/blog-distributed-strong-weak-consistency.html)
    6. CACHE维护指令
    7. TLB维护指令
        1. `tlbsrch` 若未实现LVZ，则使用`CSR.ASID`和`CSR.TLBEHI`的信息去查TLB，如果命中，则将索引写入`CSR.TLBIDX.Index`，将`CSR.TLBIDX`的NE=0，若未命中则NE=1
        2. `tlbrd` 若未实现LVZ，则将`CSR.TLBIDX.Index`作为索引去读TLB
        3. `tlbwr` 若未实现LVZ，将CSR中存放的页表项信息依据`CSR.TLBIDX.Index`存入TLB（CSR.TLBEHI, CSR.TBLELO0, CSR.TLBLO1, CSR.TLBIDX.PS）
        4. `tlbfill`
        5. `tlbclr` 维持TLB与内存页表数据的一致性，依据`CSR.TLBIDX.Index`，当其落入STLB范围时，执行一条tlbclr，将STLB中由Index低位所指示的那一组中的所有路中G=0且ASID=`CSR.ASID.ASID`的页表项无效掉。
        6. `tlbflush` 当Index落在MTLB范围内时，执行tlbflush，将MTLB中所有页表置无效；若落在STLB中则将Index低位所指示的那一组所有路的页表无效掉
        7. `invtlb [op,rj,rk]` ，主要用于无效TLB中的内容，详情见《LoongArch-Vol1-v1.02-CN》P75页操作表格，op=0x0-0x6对应不同的操作。
    8. 软件遍历页表指令
       
       
        | level | CSR |
        | --- | --- |
        | 1 | `CSR.PWCL.PT` |
        | 2 | `CSR.PWCL.Dir1` |
        | 3 | `CSR.PWCL.Dir2` |
        | 4 | `CSR.PWCL.Dir3` |
        1. `lddir [rd,rj,level]` level指定当前访问的是哪一级页表
            1. 若rj[6]=0，此时rj中是level级页表的基址的物理地址，并根据当前的TLB重填地址（？）访问level页表，取得level+1级的页表基址。
            2. 若rj[6]=1，则这时一个大页（Huge Page）的页表项，rj的值直接写入rd。
        2. `ldpte [rj.req]` 若rj[6]=0则rj内容是PTE那一级页表的基址的物理地址，根据当前TLB重填地址访问PTE级页表，取回页表项写入对应CSR中，若rj[6]=1则直接将rj中的值转换为最终的页表项格式写入CSR。
    9. 其他指令
        1. `ertn` 从例外处理返回
        2. `dbcl` 进入调试模式
        3. `idle` 处理器核等待，直至被中断唤醒或被复位
    
7. **虚拟地址空间**
    1. 直接地址翻译模式
    物理地址默认等于虚拟地址`[PALEN-1:0]`位
    2. 映射地址翻译模式
        1. 直接映射地址翻译模式
        配置映射窗口
        `CSR.DMW0`-`CSR.DMW3`寄存器
        虚地址的`[PALEN-1:0]`位与物理地址一致，但高位需要和配置窗口寄存器中的`VSEG`域相等。
        2. 页表映射地址翻译模式
    
8. **页表映射存储管理**
    1. 包含两种TLB：STLB和MTLB，前者页大小一致（`CSR.STLBPS`的PS域配置），后者每一个页表项对应的页可以不一样大。
    2. STLB多路组相联、MTLB全相联。
       
        ![Untitled](QEMU%20Loongarch/Untitled%207.png)
        
    3. 页表项奇偶共用，即不保存奇偶位置，由虚页号最低位判断。
    4. TLB相关例外：
        1. **TLB重填例外**
        访存的虚地址在TLB中没找到，进行TLB重填
        2. **load操作页无效例外**
        load操作找到了但V=0
        3. **store操作页无效例外**
        store操作找到了但V=0
        4. **取指操作无效例外**
        取指找到了但V=0
        5. **页特权等级不合规例外**
        V=1但特权不合规
        6. **页修改例外**
        store且V=1，特权合规，但D=0
        7. **页不可读例外**
        load且V=1，特权合规，但NR=1
        8. **页不可执行例外**
        取指且V=1，特权合规，但NX=1
    5. TLB的初始化 - `invtlb r0.r0`
       
        ![Untitled](QEMU%20Loongarch/Untitled%208.png)
        
        ![Untitled](QEMU%20Loongarch/Untitled%209.png)
        
    
9. **例外与中断**
    1. **中断**
        1. 中断类型
        1个核间中断（IPI），1个定时器中断（TI），1个性能检测计数溢出中断（PMI），8个硬中断（HWI0-HWI7），2个软中断（SWI0-SWI1）。
        2. 中断号越大优先级越高
        3. 中断入口
        在计算入口地址时，中断对应的例外号=自身的中断号+64，中断信号被采样至`CSR.ESTAT.IS`域。
    2. **例外**
        1. TLB重填例外入口来自`CSR.TLBRENTRY`
        2. 机器错误例外入口来自`CSR.MERRENTRY`
        3. 其他例外称为普通例外，入口地址为“入口页号|页内偏移“的计算方式（按位或），入口页号来自`CSR.EENTRY`
        入口偏移=![CodeCogsEqn](QEMU%20Loongarch/CodeCogsEqn.gif)
        4. 例外优先级：中断大于例外、取指阶段产生的优先级最高、译码次之、执行次之。
    
10. **控制状态寄存器一览表**

![Untitled](QEMU%20Loongarch/Untitled%2010.png)

![Untitled](QEMU%20Loongarch/Untitled%2011.png)

![Untitled](QEMU%20Loongarch/Untitled%2012.png)

1. 虚拟化LVZ拓展部分暂未公开文档

## 龙芯架构手册发现的一些问题

1. 在小节2.1.7存储访问类型开头，P10，第四段，”即此类指令不可推测的执行“，”的“应为”地“。（LoongArch-Vol1-v1.02-CN）
2. 在小节4.2.4.5中，“所有路中等于G=0”应去掉”等于“。
3. 在小节4.2.5.2 LDPTE中， 指令格式处的 req 应为 seq。

# 龙芯CPU手册（LS3A5000）

1. **芯片配置寄存器**

   1. 基址为`0x1fe0_0000`，也可以使用IOCSR指令进行访问
   2. 版本寄存器 `0x0000`
   3. 芯片特性寄存器 `0x0008`
   4. 功能设置寄存器 `0x0180`
   5. 引脚驱动设置寄存器 `0x0188`
   6. 功能采样寄存器 `0x0190`
   7. 频率配置寄存器 `0x01B0`
   8. 路由设置寄存器 `0x0400`
   9. 其他功能 `0x0420` 包括JTAG, LA132, Stable Clock, extioi等

2. **软件时钟系统**

   1. Stable Counter
      3A5000中的恒定时钟源，处理器核时钟和结点时钟均来自主时钟，并可以控制分频，而Stable Counter来自输入参考时钟，不随其他时钟频率而变化。

3. **GPIO**

   1. 中断控制寄存器 `0x0510` GPIO中断使能等

4. **处理器核间中断与通信**

5. **IO中断**

   ![Untitled](QEMU%20Loongarch/Untitled%2017_1.png)

   1. **传统IO中断**
   2. **拓展IO中断**
      1. 除了兼容原有的传统 IO 中断方式，3A5000 开始支持扩展 I/O 中断，用于将 HT 总线上的 256 位中断直接分发给各个处理器核，而不再通过 HT 的中断线进行转发，提升 IO 中断使用的灵活性。
      2. **扩展 IO 中断与传统 HT 中断处理的区别**
         传统的 HT 中断处理方式下，HT 中断由 HT 控制器进行内部处理，直接映射到 HT 配置寄存器上的 256 个中断向量，再由 256 个中断向量分组产生 4 个或 8 个中断，再路由至各个不同的处理器核。由于采用的是传统的中断线连接，不能直接产生跨片中断，由此所有的 HT IO 中断都只能直接由单个芯片进行处理。另一方面，芯片内硬件分发的中断只是以最终的 4 个或 8 个中断为单位，不能按位处理，由此导致硬件中断分发不好用的问题。
         扩展 IO 中断方式，HT 中断由 HT 控制器直接发给芯片的中断控制器进行处理，中断控制器能直接得到 256 位中断，而不是之前的 4 个或 8 个中断，这 256 位中断每一位都可以独立路由，独立分发，而且可以实现跨片的分发及轮转。

6. 低速IO控制器

   1. UART控制器

      1. UART0 `0x1fe001e0`
      2. UART1 `0x1fe001e8`

   2. SPI控制器

   3. I2C控制器等

# sysHyper平台移植Loongarch

rust目前已支持loongarch*-unknown-none的target，这也是我们的目标架构。

[Loongarch Rust编译器](http://www.loongnix.cn/zh/toolchain/Rust/)

[loongarch*-unknown-none* - The rustc book](https://doc.rust-lang.org/rustc/platform-support/loongarch-none.html)

首先导出一份自定义的target JSON：

```bash
rustc -Z unstable-options --target=loongarch64-unknown-none-softfloat --print target-spec-json > loongarch64.json
```

下一步则需要将代码中aarch64相关的代码用loongarch体系结构重写，并用rust的cfg进行target控制。

## Jailhouse Hypervisor

![Untitled](QEMU%20Loongarch/Untitled%2018_1.png)

## sysHyper

### config.rs

`HvResult` - 对`Result`的包装，包含`HvError`

`HvError` - sysHyper的错误信息：

```rust
pub struct HvError {
   num: HvErrorNum,
   loc_line: u32,
   loc_col: u32,
   loc_file: &'static str,
   msg: Option<String>,
}
```

可使用`hv_err`辅助宏打印错误信息。

`HvSystemConfig` - 包含jailhouse等相关信息：

```rust
/// General descriptor of the system.
#[derive(Clone, Copy, Debug)]
#[repr(C, packed)]
pub struct HvSystemConfig {
   pub signature: [u8; 6],
   pub revision: u16,
   flags: u32,

   /// Jailhouse's location in memory
   pub hypervisor_memory: HvMemoryRegion,
   pub debug_console: HvConsole,
   pub platform_info: PlatformInfo,
   pub root_cell: HvCellDesc,
   // CellConfigLayout placed here.
}
```

`HvMemoryRegion` - 用于存储内存区域信息

```rust
#[derive(Clone, Copy, Debug)]
#[repr(C, packed)]
pub struct HvMemoryRegion {
   pub phys_start: u64,
   pub virt_start: u64,
   pub size: u64,
   pub flags: MemFlags,
}
```

`HvCellDesc` - Jailhouse Cell信息

```rust
/// The jailhouse cell configuration.
///
/// @note Keep Config._HEADER_FORMAT in jailhouse-cell-linux in sync with this
/// structure.
#[derive(Clone, Copy, Debug)]
#[repr(C, packed)]
pub struct HvCellDesc {
   signature: [u8; 6],
   revision: u16,

   name: [u8; HV_CELL_NAME_MAXLEN + 1],
   id: u32, // set by the driver
   flags: u32,

   pub cpu_set_size: u32,
   pub num_memory_regions: u32,
   pub num_cache_regions: u32,
   pub num_irqchips: u32,
   pub pio_bitmap_size: u32,
   pub num_pci_devices: u32,
   pub num_pci_caps: u32,

   vpci_irq_base: u32,

   cpu_reset_address: u64,
   msg_reply_timeout: u64,

   console: HvConsole,
}
```

`main.rs`的`main`流程：

1. 检查当前进入的CPU是否为0号primary，若是则进入`primary_init_early`
2. 否则`wait_for_counter(INIT_EARLY_OK<1)`，即等待`primary_init_early`完成
  3. `primary_init_early`流程：
     1. `memory::init_heap`
     2. `system_config.check`
     3. `memory::init_frame_allocator`
     4. `memory::init_hv_page_table`
     5. `cell::init`
4. 所有CPU等待是否全部CPU均初始化完成
5. 若上一条成功，则CPU0进入`primary_init_late`
6. 其他同上，等待CPU0完成
   1. `primary_init_late`目前内部无操作
7. 运行`gicv3_cpu_init`
8. 运行`cpu_data.activate_vmm`（每个CPU都进入）
   1. `PerCpu::return_linux`
      1. `vmreturn(PerCpu::guest_reg())`

### tock registers

利用tock registers库可以方便地进行寄存器字段定义。

[tock-registers](https://crates.io/crates/tock-registers)

### hypercall module

`HyperCallCode`，即HyperCall构造函数中的code信息，表示对Hypervisor的操作

```rust
numeric_enum! {
    #[repr(u64)]
    #[derive(Debug, Eq, PartialEq, Copy, Clone)]
    pub enum HyperCallCode {
        HypervisorDisable = 0,
        HypervisorCellCreate = 1,
    }
}
```

`HyperCall`结构，数据成员为cpu_data（`PerCpu`）

```rust
pub struct HyperCall<'a> {
    cpu_data: &'a mut PerCpu,
}
```

`HyperCall`的impl函数成员

```rust
pub fn new(cpu_data: &'a mut PerCpu) -> Self
pub fn hypercall(&mut self, code: u64, arg0: u64, _arg1: u64) -> HvResult // 构造函数
fn hypervisor_disable(&mut self) -> HyperCallResult // Disable
fn hypervisor_cell_create(&mut self) -> HyperCallResult // 创建一个Hypervisor Cell
```

该module下还有一个`arch_send_event`函数，用于向体系结构进行对应的配置，通过`ICC_SGI1R_EL1`寄存器实现。

[ARM GIC（十） GICv3软中断](https://zhuanlan.zhihu.com/p/212546832)

# ARM 体系结构虚拟化

## Overview

ARM® Architecture Reference Manual
ARMv8, for ARMv8-A architecture profile

Chapter D1 The AArch64 System Level Programmers’ Model

1. **异常等级模型(Exception Level Model)**
    1. `EL0` 应用程序
    2. `EL1` OS内核
    3. `EL2` Hypervisor
    4. `EL3` Secure Monitor
2. **Security State**
    
    1. `Secure State` - 此时PE可以访问安全和非安全的物理地址空间
    2. `Non-secure State` - PE只能访问非安全物理地址空间，不能访问Secure System Control相关资源
3. 整体结构
    1. 如下图
       
        ![Untitled](QEMU%20Loongarch/Untitled%2019.png)
    
4. **虚拟化**
    1. 硬件实现需要支持EL2
    2. 虚拟机运行于Non-secure EL1, Non-secure EL0（分别对应OS和Application）
    3. Hypervisor给每个虚拟机分配VMID(Virtual Machine Identifier)
    4. EL2提供：
        1. 一些Identification Registers的虚拟值（给Guest OS）
        2. 在许多场景下Trap，如内存相关、访问寄存器，此时Trap会进入EL2
        3. 中断路由：
            1. 路由给当前的Guest OS
            2. 路由给当前没在运行的Guest OS
            3. 路由给Hypervisor
    5. Stage 2转换
        1. `Stage 1` 将Virtual Address转化为Intermediate Physical Address（VA→IPA）
        2. `Stage 2` 将IPA转化为Physical Address（IPA→PA）
5. **虚拟中断**
    1. SError → Virtual SError
    2. IRQ → Virtual IRQ
    3. FIQ → Virtual FIQ
    4. 启动虚拟中断：`HCR_EL2.{FMO,IMO,AMO}`的routing control bit → 1
    5. 虚拟中断：Non-secure EL0→EL1 / Non-secure EL1→EL1
6. **Saved Program Status Registers(SPSRs)**
    
    1. 用于保存PE状态（当发生异常）
       
        ![Untitled](QEMU%20Loongarch/Untitled%2020.png)
        
    2. 例如target到EL2的exception，则PE state保存到`SPSR_EL2`，EL1和EL3同理
7. **Exception Link Registers(ELRs)** 用于保存异常返回地址
8. **ESR(Exception Syndrome Register)** 负责指示异常的信息
   
    ![Untitled](QEMU%20Loongarch/Untitled%2021.png)
    
9. System registers
10. **System Calls**
    1. **SVC(Supervisor Call)**
        1. 运行在EL0的应用程序通过这个指令通知EL1（发送Supervisor Call exception）
    2. **HVC(Hypervisor Call)**
        1. 当在Non-secure EL1或更高等级运行时，可以通过该指令通知EL2（发送Hypervisor Call exception）
    3. **SMC(Secure Monitor Call)**
        1. 当运行在EL1或更高等级时，该指令可以通知EL3（Secure Monitor Call exception）

## ARM PSCI

Arm Power State Coordination Interface Platform Design Document

![IMG_2712.png](QEMU%20Loongarch/IMG_2712.png)

[ARM系列 -- PSCI - 极术社区 - 连接开发者与智能计算生态](https://aijishu.com/a/1060000000297005)

下面的function功能翻译来自上面的技术博客

- `PSCI_VERSION`：可以调用此API得到当前PSCI的版本信息；
- `CPU_SUSPEND`：OSPM调用此API使处理器核进入低功耗模式；
- `CPU_OFF`：此API用于hotplug，从系统中动态移除某个处理器核。被CPU_OFF移除的处理器核只能通过CPU_ON再次加载。与CPU_SUSPEND不同的是，这个接口函数不需要返回值；
- `CPU_ON`：此API用于动态加载处理器核；
- `AFFINITY_INFO`：此API允许调度方查询亲和实体（affinity instance）的状态；
- `MIGRATE`：用于将受信任的操作系统（trusted OS）迁移到另一个处理器核，从而原处理器核可以调用CPU_OFF关闭电源；
- `MIGRATE_INFO_TYPE`：允许调用方识别受信任操作系统中存在的多核支持级别，通过返回值可以判定受信任操作系统是否必须运行在单一处理器上，是否支持迁移；
- `MIGRATE_INFO_UP_CPU`：指示受信任的操作系统当前的位置；
- `SYSTEM_OFF`：系统关闭；
- `SYSTEM_RESET`：系统冷复位；
- `SYSTEM_RESET2`：此API是对SYSTEM_RESET的扩展；
- `MEM_PROTECT`：此API确保内存在交给操作系统加载程序之前被重写，从而提供防止冷重启攻击的保护；
- `MEM_PROTECT_CHECK_RANGE`：此API用于检查某段内存范围是否受MEM_PROTECT保护；
- `PSCI_FEATURES`：此API允许调用方检测已实现的PSCI函数及其属性；
- `CPU_FREEZE`：此API将处理器核设置于低功耗状态（依赖具体设计实现）。与CPU_SUSPEND和CPU_DEFAULT_SUSPEND不同，中断不能唤醒该处理器；与CPU_OFF也不同，不需要迁移；
- `CPU_DEFAULT_SUSPEND`：此API将处理器核设置于低功耗状态（依赖具体设计实现），与CPU_FREEZE的调用参数不同；
- `NODE_HW_STATE`：此API允许直接从电源控制器或电源控制逻辑确定节点的电源状态。与AFFINITY_INFO不同，此API返回电源状态的物理视图；
- `SYSTEM_SUSPEND`：此API相当于CPU_SUSPEND到最深的低功耗状态，但实际系统中有可能实现比CPU_SUSPEND更深的低功耗状态，比如支持RAM挂起；
- `PSCI_SET_ SUSPEND_MODE`：此API允许设置CPU_SUSPEND用于协调电源状态的模式；
- `PSCI_STAT_RESIDENCY`：此API返回自冷启动后平台处于某个电源状态的时间；
- `PSCI_STAT_COUNT`：此API返回自冷启动后平台使用某个电源状态的次数；

## ARM SCMI

Arm® System Control and Management Interface Platform Design Document

[ARM系列 -- SCMI - 极术社区 - 连接开发者与智能计算生态](https://aijishu.com/a/1060000000300745?eqid=fd4b5abf000a80f50000000664816b13)

![IMG_2713.png](QEMU%20Loongarch/IMG_2713.png)

[ARM SCP入门-AP与SCP通信-电子发烧友网](https://www.elecfans.com/d/2184468.html)

1. 包含协议层和传输层
    1. Protocol
    2. Transport
2. OS通常为agent，SCP为platform，**SCP(System Control Processor)**是一个协处理器，专门负责电源等系统管理。

## GICv3

GICv3 and GICv4 Software Overview

[GIC 中断虚拟化](https://zhuanlan.zhihu.com/p/535997324)

1. **基本概念**
   
    ![Untitled](QEMU%20Loongarch/Untitled%2022.png)
    
    1. **SPI** (Shared Peripheral Interrupt) - 一种全局外围中断，可以路由到指定PE，或到一组PEs
    2. **PPI** (Private Peripheral Interrupt) - 某个PE自身的外围中断，如Generic Timer
    3. **SGI** (Software Generated Interrupt) - 软件写SGI寄存器触发
       
        ![Untitled](QEMU%20Loongarch/Untitled%2023.png)
        
    4. **LPI** (Locality-specific Peripheral Interrupt) - *message-based*中断
       
        ![Untitled](QEMU%20Loongarch/Untitled%2024.png)
        
        ![Untitled](QEMU%20Loongarch/Untitled%2025.png)
        
    5. 一个中断路由示例
    6. **Distributor** (`GICD_*`)
        
        1. SPI的中断优先级配置
        2. SPI开关配置
        3. 生成message-based SPIs
        4. 控制active和pending的SPIs
        5. …
    7. **Redistributors** (`GICR_*`)
        
        1. SGI和PPI的开关配置
        2. SGI和PPI优先级配置
        3. SGI和PPI分组
        4. …
    8. **CPU interfaces** (`ICC_*_ELn`)
        
        1. 打开中断处理的相关配置
        2. 接收中断
        3. 设置PE中断抢占策略
        4. 选择PE最高优先级的pending中断
        5. …
2. **GICv3虚拟化相关**
   
    ![Untitled](QEMU%20Loongarch/Untitled%2026.png)
    
    ![Untitled](QEMU%20Loongarch/Untitled%2027.png)
    
    1. CPU Interface
        1. Physical CPU interface (`ICC_*_ELn`)
        2. Virtualization Control (`ICH_*_EL2`)
        3. Virtual CPU interface (`ICV_*_ELn`)
    
    ![Untitled](QEMU%20Loongarch/Untitled%2028.png)