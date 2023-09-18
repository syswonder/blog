# 树莓派4B运行 ArceOS 问题总结

时间：2023.6.1

作者：袁世平

联系方式：robert_yuan@pku.edu.cn



## 2023 年 9月 更新

因为目前 ArceOS 中的 feature 发生了一些改变，之前部分的代码都经过了一定的重构。

同时因为李明老师实验了在 qemu 上模拟树莓派4运行 ArceOS，也是对这个工作的一个很好的完善，所以更新一下这个文档，以方便后面的同学更好的了解和上手这部分的代码。

### 在 QEMU 上运行树莓派4

目前的 qemu 最新版也是不支持树莓派4的，这里用了 github 上一个项目 https://github.com/0xMirasio/qemu-patch-raspberry4 来编译得到可以支持树莓派4 的 qemu-system-aarch64 

操作：

```
git clone https://github.com/0xMirasio/qemu-patch-raspberry4
cd qemu-patch-raspberry4
mkdir build
cd build
../configure
make
```

编译时间比较久，之后执行

```
./qemu-system-aarch64 -M help | grep raspi
```

可以看到已经支持了树莓派4

```
raspi0               Raspberry Pi Zero (revision 1.2)
raspi1ap             Raspberry Pi A+ (revision 1.1)
raspi2b              Raspberry Pi 2B (revision 1.1)
raspi3ap             Raspberry Pi 3A+ (revision 1.0)
raspi3b              Raspberry Pi 3B (revision 1.2)
raspi4b1g            Raspberry Pi 4B (revision 1.1)
raspi4b2g            Raspberry Pi 4B (revision 1.2)
```

接下来编译 ArceOS 在 raspi4 上的镜像：

```
# Hello
make A=apps/helloworld ARCH=aarch64 PLATFORM=aarch64-raspi4 SMP=4
# shell using ramdisk
make A=apps/fs/shell ARCH=aarch64 PLATFORM=aarch64-raspi4 SMP=4 BLK=y FEATURES=driver-ramdisk
# shell using sd card
make A=apps/fs/shell ARCH=aarch64 PLATFORM=aarch64-raspi4 SMP=4 BLK=y FEATURES=driver-bcm2835-sdhci
```

使用 qemu-system-aarch64 运行镜像

```
# run hello world
./qemu-system-aarch64 -m 2G -smp 4 -cpu cortex-a72 -machine raspi4b2g -nographic -kernel {yourpath}/helloworld_aarch64-raspi4.bin

# run fs/shell using sd card
./qemu-system-aarch64 -m 2G -smp 4 -cpu cortex-a72 -machine raspi4b2g -nographic -device if=sd,index=0,media=disk -kernel {yourpath}/shell_aarch64-raspi4.bin
```

====== end of September update =====



本文中简称 rust-raspberrypi-OS-tutorials 为 Tutorial

## 摘要

这篇文章主要讲述了如何在树莓派4上运行 ArceOS 

涉及到的内容是树莓派4在各个方面与 qemu-virt 之间的差异所需要对代码进行的修改

主要包括了：启动、串口、中断、多核、存储（Ramdisk）等方面

对于网络显示等设备，因为还没有支持树莓派4的驱动，而树莓派4无法支持 virtio，所以目前还不支持，存储需要使用 Ram Disk 来代替。



## 启动

启动代码主要有一个问题

### 1. 起始地址不一致

在树莓派4中会在启动操作系统代码之前根据 `config.txt` 执行 `start4.elf`  等固件的代码

其中会涉及跳转到操作系统部分代码的指令，并且假定操作系统的第一条指令放在 `0x80000`，所以操作系统代码中也需要保持这个同样的假定。

所以需要修改 `axconfig` 目录下的 `raspi4-aarch64.toml`中的 `kernel-base-paddr` ，同时物理地址，也可以按照 Tutorial 中的，把`phys-memory-base`改成从 `0x0` 开始



## 串口

连接串口主要遇到两个问题：

### 1. 无法连接

因为树莓派4中的串口地址与 QEMU 中不同。在树莓派中这个值为 `0xFE20_1000` 

遇到类似这样的问题，最好是查看 Tutorial 中对应的章节的对应内容。

在代码中需要修改的地方有两个，一个是 `axconfig` 目录下的 `raspi4-aarch64.toml`  中的 `mmio-regions` 把 pl011 UART 的内容改成

`["0xFE20_1000", "0X1000"]` 

另一个是在 `modules/axhal/src/platform/raspi4-aarch64` 下的`pl011.rs` 中，需要修改的是 `line 11` 和 `13`

```rust
const UART_BASE: PhysAddr = PhysAddr::from(0xFE20_1000);    // raspi4 的串口地址为 0xFE20_1000
pub const UART_IRQ_NUM: usize = 153; 						// raspi4 的串口中断号为 153 
```



### 2. 内存访问错误

造成这个错误的原因来自 boot 阶段使用的页表，之前在 qemu 中用的时候，是划分了两个 1G 的内存，一个是从 0x0 ~ 0x4000_0000 标记为 DEVICE 设备段，不可执行。一个是 0x4000_0000 到 0x8000_0000 作为代码段和数据段，标记为 EXECUTE 可以执行。

这是因为之前的启动代码在 0x4008_0000 所以是这样设置的，但是在树莓派4上，从 0x80000 就开始执行，而对应的设备地址（目前只有 UART），则是包括了如 `0xFE20_1000` ，所以需要将 0x0 ~ 0x4000_0000 标记为可执行，把 0xc000_0000 到 0x1_0000_0000 标记为设备段。

这个的代码在 `modules/axhal/src/platform/raspi4-aarch64.rs` 下的`boot.rs` 中

```rust
unsafe fn init_boot_page_table() {
    // 0x0000_0000_0000 ~ 0x0080_0000_0000, table
    BOOT_PT_L0[0] = A64PTE::new_table(PhysAddr::from(BOOT_PT_L1.as_ptr() as usize));
    // 0x0000_0000_0000..0x0000_4000_0000, 1G block, device memory
    BOOT_PT_L1[0] = A64PTE::new_page(
        PhysAddr::from(0),
        MappingFlags::READ | MappingFlags::WRITE | MappingFlags::EXECUTE,
        true,
    );
    // 0x0000_4000_0000..0x0000_8000_0000, 1G block, normal memory
    BOOT_PT_L1[1] = A64PTE::new_page(
        PhysAddr::from(0x4000_0000),
        MappingFlags::READ | MappingFlags::WRITE | MappingFlags::EXECUTE,
        true,
    );
    // 0x0000_8000_0000..0x0000_C000_0000, 1G block, normal memory
    BOOT_PT_L1[2] = A64PTE::new_page(
        PhysAddr::from(0x8000_0000),
        MappingFlags::READ | MappingFlags::WRITE | MappingFlags::EXECUTE,
        true,
    );
    // 0x0000_C000_0000..0x0001_000_0000, 1G block, DEVICE memory
    BOOT_PT_L1[3] = A64PTE::new_page(
        PhysAddr::from(0xc000_0000),
        MappingFlags::READ | MappingFlags::WRITE | MappingFlags::DEVICE,
        true,
    );
}
```



## 中断

在 树莓派4 中使用的中断控制器也是 gic-v2 

这里主要需要改的也就是 GICD 和 GICC 的起始地址

### GIC 地址

在 modules/axhal/src/platform/qemu-virt-aarch64/irq.rs 中

修改

```rust
const GICD_BASE: PhysAddr = PhysAddr::from(0xFF84_1000);    // raspi4
const GICC_BASE: PhysAddr = PhysAddr::from(0xFF84_2000);    // raspi4
```



## 多核启动

在树莓派中启动多核的方法是通过往 SPIN_TABLE 中对应核的地址写入来完成的。

```rust
#[no_mangle]
pub static CPU_SPIN_TABLE: [usize; 4] = [0xd8, 0xe0, 0xe8, 0xf0];

#[cfg(all(feature = "smp", feature = "platform-raspi4-aarch64"))]
pub fn start_secondary_cpus(primary_cpu_id: usize) {
    let entry_address = virt_to_phys(VirtAddr::from(modify_stack_and_start as usize)).as_usize();
    for (i, _) in CPU_SPIN_TABLE.iter().enumerate().take(SMP) {
        if i != primary_cpu_id {
            let release_addr = CPU_SPIN_TABLE[i] as *mut usize;
            unsafe {
                write_volatile(release_addr, entry_address);
            }
            axhal::arch::flush_dcache(release_addr as usize);
        }
    }
    asm::sev();
}
```

核 0 ～ 3 分别为 `[0xd8, 0xe0, 0xe8, 0xf0]`

在这几个地址上放入希望对应核运行的第一个函数的地址，然后刷新 D-Cache 让他们可以看到，然后用 asm::sev() 来发送一个 Event 这样其他核就可以运行了



这里运行的第一个函数不同于 qemu 中运行的第一个函数，直接从 `_start_secondary()` 开始，而是自己写的一个函数，需要先修改每个核的栈，然后才可以执行。

这里的栈是 Core0 在初始化完堆分配器之后分配的一个数组，大小为 config 中设定好的 STACK_SIZE 乘上其他核的数量  (SMP-1)

这里因为栈是向下伸展的，所以每个核获得的栈指针为这个数组的较大的一端，通过 `mpidr_el1`  获得 `cpu_id` （从1开始），然后计算偏移，为 cpuid * 0x40000（2^18）

```rust
#[cfg(all(
    feature = "smp",
    any(
        feature = "platform-raspi4-aarch64",
        feature = "platform-qemu-raspi3-aarch64"
    )
))]
#[no_mangle]
#[link_section = ".text.boot"]
unsafe extern "C" fn modify_stack_and_start() {
    core::arch::asm!("
        mrs     x19, mpidr_el1
        and     x19, x19, #0xffffff             // get current CPU id
        lsl     x20, x19, #18
        ldr     x21, ={secondary_boot_stack}    // core0 store the stack start adress at this address
        mov     x8, {phys_virt_offset} 
        sub     x21, x21, x8
        add     sp, x21, x20
        b       {start_secondary}",
        secondary_boot_stack = sym SECONDARY_BOOT_STACK,
        start_secondary = sym _start_secondary,
        phys_virt_offset = const axconfig::PHYS_VIRT_OFFSET,
    );
}
```

要注意这里因为其他核启动的时候没有配置页表，使用的都是低地址，但是我们的函数是在高地址的虚拟地址，所以在设置栈的时候也需要减去这个偏移。



同时也需要修改之前写错的第一个地方，在 `arch/arc/platform/raspi4_aarch64/boot.rs` 中的函数 `switch_to_el1()` 中设置了 SP_EL1 为 BOOT_STACK，而其他核也会执行这个函数，那么我们设置的 sp 也就没用了，所以需要把 `switch_to_el1()` 中的

```
 SP_EL1.set(BOOT_STACK.as_ptr_range().end as u64);
```

注释掉，并且在 _start 和 _start_secondary 函数中，执行 switch_to_el1 之前，把 SP_EL1 设置好

```
mov     x8, sp                
msr     sp_el1, x8
msr     sp_el0, xzr
```

这样换栈的操作才算真正完成了。



## 修复页表中设置的问题

在页表设置中还有一个问题，是因为 bitflag 的版本升级了，之前用来设置页表的 EXECUTE 的时候，对于 Normal 类型的页表项需要在 0x4 位上设置为1，但是因为页表项的定义中没有在 bitflag 中设置这一项，就会导致在 `/crates/page_table_entry/src/arch/aarch64.rs` 的`from_mem_type()`函数中使用 `from_bits_truncate()`  把这个 0x4 这一位给清除掉，从而设置页表发生错误，进一步导致后面同步语句出现问题，在 axlog 的初始化中卡住。

所以这里需要使用 `from_bits_retain` 来代替 `from_bits_truncate` 才能够正确运行。

```
    fn from_mem_type(mem_type: MemType) -> Self {
        let mut bits = (mem_type as u64) << 2;
        if matches!(mem_type, MemType::Normal) {
            bits |= Self::INNER.bits() | Self::SHAREABLE.bits();
        }
        Self::from_bits_retain(bits)
    }
```



## 跑通 sqlite3 测试（Ramdisk 相关的问题）

【这一条已经在 PR #65 中修复，具体细节见该次提交】

c语言的交叉编译链接，见 Readme

`wget https://musl.cc/aarch64-linux-musl-cross.tgz` 

因为树莓派没有 virtio 所以需要使用 Ramdisk 作为存储设备运行

所以我们需要增加一个使用 ramdisk 的特征 use-ramdisk

在 libax 中加入这个特征

```
use-ramdisk = ["axdriver?/ramdisk", "axfs?/use-ramdisk"]
```

在代码中，ramdisk 的优先级要高于 virtio-blk 所以不需要修改其他地方对 virtio-blk 的设置。

这里 axdriver 是使用 ramdisk 的驱动，axfs 则是告诉 FAT32 需要对 RAM Disk 做 FAT32 的格式化。

之前运行 RamDisk 没有做最开始的格式化，导致直接在上面初始化 FATFS 会导致错误

所以我们需要修改 `axfs/src/fatfs.rs`

在 FatFileSystem 结构体的 new 函数中：

```rust
    pub fn new(mut disk: Disk) -> Self {
        #[cfg(feature = "use-ramdisk")]{
            let opts =  fatfs::FormatVolumeOptions::new();
            fatfs::format_volume(&mut disk, opts).expect("format volume");
        }
        
        let inner = fatfs::FileSystem::new(disk, fatfs::FsOptions::new())
            .expect("failed to initialize FAT filesystem");
        Self {
            inner,
            root_dir: UnsafeCell::new(None),
        }
    }
```

我们增加了在使用 ramdisk 的基础上使用 fatfs 库提供的 format_volume 的方法来格式化 Ramdisk

同时因为格式化需要让 disk 类型变为 mut 类型，所以我们也需要修改传进来的参数为 mut 类型

在修改之后就可以正常运行了
