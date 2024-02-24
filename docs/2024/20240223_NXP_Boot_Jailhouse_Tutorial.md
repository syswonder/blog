# NXP启动Jailhouse

整体思路：

1. 使用SD卡启动第一个Linux，这个Linux最好使用ubuntu的rootfs，并且配通网络【方便安装包】。
2. 启动root Linux，并且编译一遍Linux 内核以及Jailhouse。
3. 重启，修改root dtb，启动root Linux。
4. jailhouse 启动nonroot Linux，该Linux是emmc上的Linux（原厂商的Linux），指定rootfs为emmc。

## 一、制作ubuntu SD卡镜像

```shell
wget https://cdimage.ubuntu.com/ubuntu-base/releases/18.04/release/ubuntu-base-18.04.5-base-arm64.tar.gz
tar xjvf ubuntu-base-18.04.5-base-arm64.tar.gz

cd ubuntu-base-18.04.5-base-arm64
sudo apt-get install qemu
sudo cp /usr/bin/qemu-aarch64-static usr/bin/

sudo mount /sys ./sys -o bind
sudo mount /proc ./proc -o bind
sudo mount /dev ./dev -o bind

sudo mv etc/resolv.conf etc/resolv.conf.saved
sudo cp /etc/resolv.conf etc

sudo LC_ALL=C chroot . /bin/bash
sudo apt-get update 
sudo apt-get install <PKG_NAME> # 安装需要的包，如vim, build-essential.

exit

sudo umount ./sys
sudo umount ./proc
sudo umount ./dev

mv etc/resolv.conf.saved etc/resolv.conf

## 另外，将Linux和jailhouse复制到SD卡中。
# 然后将ubuntu-base-18.04.5-base-arm64目录拷进SD卡中作为rootfs。
```

## 二、启动一个默认的NXP Linux

请参考NXP手册，编译烧写默认的Linux系统。

## 三、修改root设备树

内容：

OK8MP-C-root.dts

```C
// SPDX-License-Identifier: (GPL-2.0+ OR MIT)
/*
 * Copyright 2019 NXP
 */

/dts-v1/;

#include "OK8MP-C.dts"

/ {
        interrupt-parent = <&gic>;

        resmem: reserved-memory {
                #address-cells = <2>;
                #size-cells = <2>;
                ranges;
        };
};

&cpu_pd_wait {
        /delete-property/ compatible;
};

&clk {
        init-on-array = <IMX8MP_CLK_USDHC3_ROOT
                         IMX8MP_CLK_NAND_USDHC_BUS
                         IMX8MP_CLK_HSIO_ROOT
                         IMX8MP_CLK_UART4_ROOT
                         IMX8MP_CLK_OCOTP_ROOT>;
};

&{/busfreq} {
        status = "disabled";
};

&{/reserved-memory} { // 预留的jailhouse内存区域
        jh_reserved: jh@fdc00000 {
                no-map;
                reg = <0 0xfdc00000 0x0 0x400000>;
        };

        loader_reserved: loader@fdb00000 {
                no-map;
                reg = <0 0xfdb00000 0x0 0x00100000>;
        };

        ivshmem_reserved: ivshmem@fda00000 {
                no-map;
                reg = <0 0xfda00000 0x0 0x00100000>;
        };

        ivshmem2_reserved: ivshmem2@fd900000 {
                no-map;
                reg = <0 0xfd900000 0x0 0x00100000>;
        };

        pci_reserved: pci@fd700000 {
                no-map;
                reg = <0 0xfd700000 0x0 0x00200000>;
        };

        inmate_reserved: inmate@60000000 {
                no-map;
                reg = <0 0x60000000 0x0 0x10000000>;
        };
};

&iomuxc {
        pinctrl_uart4: uart4grp {
                fsl,pins = <
                        MX8MP_IOMUXC_UART4_RXD__UART4_DCE_RX    0x49
                        MX8MP_IOMUXC_UART4_TXD__UART4_DCE_TX    0x49
                >;
        };
};

&usdhc3 { // emmc: mmc2，即从emmc启动的Linux，因为这个emmc是nonroot，所以root不要占用，因此要禁用掉
        status = "disabled";
};

&uart4 { // 这个也禁用掉，用于nonroot启动。
        /delete-property/ dmas;
        /delete-property/ dma-names;
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_uart4>;
        status = "disabled";
};

&uart2 { // uart1=ttymxc0 uart4=ttymxc3 default for ttymxc1。
        /* uart4 is used by the 2nd OS, so configure pin and clk */
        pinctrl-0 = <&pinctrl_uart2>, <&pinctrl_uart4>;
        assigned-clocks = <&clk IMX8MP_CLK_UART4>;
        assigned-clock-parents = <&clk IMX8MP_CLK_24M>;
};

&usdhc2 {
        pinctrl-0 = <&pinctrl_usdhc3>, <&pinctrl_usdhc2>, <&pinctrl_usdhc2_gpio>;
        pinctrl-1 = <&pinctrl_usdhc3>, <&pinctrl_usdhc2_100mhz>, <&pinctrl_usdhc2_gpio>;
        pinctrl-2 = <&pinctrl_usdhc3>, <&pinctrl_usdhc2_200mhz>, <&pinctrl_usdhc2_gpio>;
};
```

## 四、启动Linux

uboot指令如下:

`setenv mmcroot /dev/mmcblk1p1 rootwait rw; setenv fdt_file OK8MP-C-root.dtb; run mmcargs; ext4load mmc 1:1 ${loadaddr} home/comet/OK8MP_linux_kernel/arch/arm64/boot/Image; ext4load mmc 1:1 ${fdt_addr} home/comet/OK8MP_linux_kernel/arch/arm64/boot/dts/freescale/OK8MP-C-root.dtb; booti ${loadaddr} - ${fdt_addr}`

解释：

mmcblk1p1: SD卡ext4文件系统分区。

home/comet/OK8MP_linux_kernel/arch/arm64/boot/Image：Image路径

home/comet/OK8MP_linux_kernel/arch/arm64/boot/dts/freescale/OK8MP-C-root.dtb：dtb路径

## 五、编译安装kernel、jailhouse

```shell
make && make install 
cd extra/jailhouse && make && make install 
```

## 六、准备non root Linux文件

imx8mp.cell

```C
/*
 * i.MX8MM Target
 *
 * Copyright 2018 NXP
 *
 * Authors:
 *  Peng Fan <peng.fan@nxp.com>
 *
 * This work is licensed under the terms of the GNU GPL, version 2.  See
 * the COPYING file in the top-level directory.
 *
 * Reservation via device tree: reg = <0x0 0xffaf0000 0x0 0x510000>
 */

#include <jailhouse/types.h>
#include <jailhouse/cell-config.h>

struct {
        struct jailhouse_system header;
        __u64 cpus[1];
        struct jailhouse_memory mem_regions[15];
        struct jailhouse_irqchip irqchips[3];
        struct jailhouse_pci_device pci_devices[2];
} __attribute__((packed)) config = {
        .header = {
                .signature = JAILHOUSE_SYSTEM_SIGNATURE,
                .revision = JAILHOUSE_CONFIG_REVISION,
                .flags = JAILHOUSE_SYS_VIRTUAL_DEBUG_CONSOLE,
                .hypervisor_memory = {
                        .phys_start = 0xfdc00000,
                        .size =       0x00400000,
                },
                .debug_console = {
                        .address = 0x30890000,
                        .size = 0x1000,
                        .flags = JAILHOUSE_CON_TYPE_IMX |
                                 JAILHOUSE_CON_ACCESS_MMIO |
                                 JAILHOUSE_CON_REGDIST_4,
                        .type = JAILHOUSE_CON_TYPE_IMX,
                },
                .platform_info = {
                        .pci_mmconfig_base = 0xfd700000,
                        .pci_mmconfig_end_bus = 0,
                        .pci_is_virtual = 1,
                        .pci_domain = 0,

                        .arm = {
                                .gic_version = 3,
                                .gicd_base = 0x38800000,
                                .gicr_base = 0x38880000,
                                .maintenance_irq = 25,
                        },
                },
                .root_cell = {
                        .name = "imx8mp",

                        .num_pci_devices = ARRAY_SIZE(config.pci_devices),
                        .cpu_set_size = sizeof(config.cpus),
                        .num_memory_regions = ARRAY_SIZE(config.mem_regions),
                        .num_irqchips = ARRAY_SIZE(config.irqchips),
                        /* gpt5/4/3/2 not used by root cell */
                        .vpci_irq_base = 51, /* Not include 32 base */
                },
        },

        .cpus = {
                0xf,
        },

        .mem_regions = {
                /* IVHSMEM shared memory region for 00:00.0 (demo )*/ {
                        .phys_start = 0xfd900000,
                        .virt_start = 0xfd900000,
                        .size = 0x1000,
                        .flags = JAILHOUSE_MEM_READ,
                },
                {
                        .phys_start = 0xfd901000,
                        .virt_start = 0xfd901000,
                        .size = 0x9000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE ,
                },
                {
                        .phys_start = 0xfd90a000,
                        .virt_start = 0xfd90a000,
                        .size = 0x2000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE ,
                },
                {
                        .phys_start = 0xfd90c000,
                        .virt_start = 0xfd90c000,
                        .size = 0x2000,
                        .flags = JAILHOUSE_MEM_READ,
                },
                {
                        .phys_start = 0xfd90e000,
                        .virt_start = 0xfd90e000,
                        .size = 0x2000,
                        .flags = JAILHOUSE_MEM_READ,
                },
                /* IVSHMEM shared memory regions for 00:01.0 (networking) */
                JAILHOUSE_SHMEM_NET_REGIONS(0xfda00000, 0),
                /* IO */ {
                        .phys_start = 0x00000000,
                        .virt_start = 0x00000000,
                        .size =       0x40000000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_IO,
                },
                /* RAM 00*/ {
                        .phys_start = 0x40000000,
                        .virt_start = 0x40000000,
                        .size = 0x80000000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_EXECUTE,
                },
                /* Inmate memory */{
                        .phys_start = 0x60000000,
                        .virt_start = 0x60000000,
                        .size = 0x10000000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_EXECUTE | JAILHOUSE_MEM_DMA,
                },
                /* Loader */{
                        .phys_start = 0xfdb00000,
                        .virt_start = 0xfdb00000,
                        .size = 0x100000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_EXECUTE,
                },
                /* OP-TEE reserved memory?? */{
                        .phys_start = 0xfe000000,
                        .virt_start = 0xfe000000,
                        .size = 0x2000000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
                },
                /* RAM04 */{
                        .phys_start = 0x100000000,
                        .virt_start = 0x100000000,
                        .size = 0xC0000000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
                },
        },

        .irqchips = {
                /* GIC */ {
                        .address = 0x38800000,
                        .pin_base = 32,
                        .pin_bitmap = {
                                0xffffffff, 0xffffffff, 0xffffffff, 0xffffffff,
                        },
                },
                /* GIC */ {
                        .address = 0x38800000,
                        .pin_base = 160,
                        .pin_bitmap = {
                                0xffffffff, 0xffffffff, 0xffffffff, 0xffffffff,
                        },
                },
                /* GIC */ {
                        .address = 0x38800000,
                        .pin_base = 288,
                        .pin_bitmap = {
                                0xffffffff, 0xffffffff, 0xffffffff, 0xffffffff,
                        },
                },
        },

        .pci_devices = {
                { /* IVSHMEM 0000:00:00.0 (demo) */
                        .type = JAILHOUSE_PCI_TYPE_IVSHMEM,
                        .domain = 0,
                        .bdf = 0 << 3,
                        .bar_mask = JAILHOUSE_IVSHMEM_BAR_MASK_INTX,
                        .shmem_regions_start = 0,
                        .shmem_dev_id = 0,
                        .shmem_peers = 3,
                        .shmem_protocol = JAILHOUSE_SHMEM_PROTO_UNDEFINED,
                },
                { /* IVSHMEM 0000:00:01.0 (networking) */
                        .type = JAILHOUSE_PCI_TYPE_IVSHMEM,
                        .domain = 0,
                        .bdf = 1 << 3,
                        .bar_mask = JAILHOUSE_IVSHMEM_BAR_MASK_INTX,
                        .shmem_regions_start = 5,
                        .shmem_dev_id = 0,
                        .shmem_peers = 2,
                        .shmem_protocol = JAILHOUSE_SHMEM_PROTO_VETH,
                },
        },
};
```



imx8mp-linux-demo.c

```C
/*
 * iMX8MM target - linux-demo
 *
 * Copyright 2019 NXP
 *
 * Authors:
 *  Peng Fan <peng.fan@nxp.com>
 *
 * This work is licensed under the terms of the GNU GPL, version 2.  See
 * the COPYING file in the top-level directory.
 */

/*
 * Boot 2nd Linux cmdline:
 * export PATH=$PATH:/usr/share/jailhouse/tools/
 * jailhouse cell linux imx8mp-linux-demo.cell Image -d imx8mp-evk-inmate.dtb -c "clk_ignore_unused console=ttymxc3,115200 earlycon=ec_imx6q,0x30890000,115200  root=/dev/mmcblk2p2 rootwait rw"
 */
#include <jailhouse/types.h>
#include <jailhouse/cell-config.h>

struct {
        struct jailhouse_cell_desc cell;
        __u64 cpus[1];
        struct jailhouse_memory mem_regions[15];
        struct jailhouse_irqchip irqchips[2];
        struct jailhouse_pci_device pci_devices[2];
} __attribute__((packed)) config = {
        .cell = {
                .signature = JAILHOUSE_CELL_DESC_SIGNATURE,
                .revision = JAILHOUSE_CONFIG_REVISION,
                .name = "linux-inmate-demo",
                .flags = JAILHOUSE_CELL_PASSIVE_COMMREG,

                .cpu_set_size = sizeof(config.cpus),
                .num_memory_regions = ARRAY_SIZE(config.mem_regions),
                .num_irqchips = ARRAY_SIZE(config.irqchips),
                .num_pci_devices = ARRAY_SIZE(config.pci_devices),
                .vpci_irq_base = 154, /* Not include 32 base */
        },

        .cpus = {
                0xc,
        },

        .mem_regions = {
                /* IVHSMEM shared memory region for 00:00.0 (demo )*/ {
                        .phys_start = 0xfd900000,
                        .virt_start = 0xfd900000,
                        .size = 0x1000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_ROOTSHARED,
                },
                {
                        .phys_start = 0xfd901000,
                        .virt_start = 0xfd901000,
                        .size = 0x9000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_ROOTSHARED,
                },
                {
                        .phys_start = 0xfd90a000,
                        .virt_start = 0xfd90a000,
                        .size = 0x2000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_ROOTSHARED,
                },
                {
                        .phys_start = 0xfd90c000,
                        .virt_start = 0xfd90c000,
                        .size = 0x2000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_ROOTSHARED,
                },
                {
                        .phys_start = 0xfd90e000,
                        .virt_start = 0xfd90e000,
                        .size = 0x2000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_ROOTSHARED,
                },
                /* IVSHMEM shared memory regions for 00:01.0 (networking) */
                JAILHOUSE_SHMEM_NET_REGIONS(0xfda00000, 1),
                /* UART2 earlycon */ {
                        .phys_start = 0x30890000,
                        .virt_start = 0x30890000,
                        .size = 0x1000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_IO | JAILHOUSE_MEM_ROOTSHARED,
                },
                /* UART4 */ {
                        .phys_start = 0x30a60000,
                        .virt_start = 0x30a60000,
                        .size = 0x1000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_IO,
                },
                /* SHDC3 */ {
                        .phys_start = 0x30b60000,
                        .virt_start = 0x30b60000,
                        .size = 0x10000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_IO,
                },
                /* RAM: Top at 4GB Space */ {
                        .phys_start = 0xfdb00000,
                        .virt_start = 0,
                        .size = 0x10000, /* 64KB */
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_EXECUTE | JAILHOUSE_MEM_LOADABLE,
                },
                /* RAM */ {
                        /*
                         * We could not use 0x80000000 which conflicts with
                         * COMM_REGION_BASE
                         */
                        .phys_start = 0x60000000,
                        .virt_start = 0x60000000,
                        .size = 0x10000000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_EXECUTE | JAILHOUSE_MEM_DMA |
                                JAILHOUSE_MEM_LOADABLE,
                },
                /* communication region */ {
                        .virt_start = 0x80000000,
                        .size = 0x00001000,
                        .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                JAILHOUSE_MEM_COMM_REGION,
                },
        },

        .irqchips = {
                /* uart2/sdhc1 */ {
                        .address = 0x38800000,
                        .pin_base = 32,
                        .pin_bitmap = {
                                (1 << (24 + 32 - 32)) | (1 << (29 + 32 - 32))
                        },
                },
                /* IVSHMEM */ {
                        .address = 0x38800000,
                        .pin_base = 160,
                        .pin_bitmap = {
                                0xf << (154 + 32 - 160) /* SPI 154-157 */
                        },
                },
        },

        .pci_devices = {
                { /* IVSHMEM 00:00.0 (demo) */
                        .type = JAILHOUSE_PCI_TYPE_IVSHMEM,
                        .domain = 0,
                        .bdf = 0 << 3,
                        .bar_mask = JAILHOUSE_IVSHMEM_BAR_MASK_INTX,
                        .shmem_regions_start = 0,
                        .shmem_dev_id = 2,
                        .shmem_peers = 3,
                        .shmem_protocol = JAILHOUSE_SHMEM_PROTO_UNDEFINED,
                },
                { /* IVSHMEM 00:01.0 (networking) */
                        .type = JAILHOUSE_PCI_TYPE_IVSHMEM,
                        .domain = 0,
                        .bdf = 1 << 3,
                        .bar_mask = JAILHOUSE_IVSHMEM_BAR_MASK_INTX,
                        .shmem_regions_start = 5,
                        .shmem_dev_id = 1,
                        .shmem_peers = 2,
                        .shmem_protocol = JAILHOUSE_SHMEM_PROTO_VETH,
                },
        },
};
```



non-root imx8mp-evk-inmate.dts

```C
// SPDX-License-Identifier: (GPL-2.0+ OR MIT)
/*
 * Copyright 2019 NXP
 */

/dts-v1/;

#include <dt-bindings/interrupt-controller/arm-gic.h>

/ {
        model = "Freescale i.MX8MP EVK";
        compatible = "fsl,imx8mp-evk", "fsl,imx8mp";
        interrupt-parent = <&gic>;
        #address-cells = <2>;
        #size-cells = <2>;

        aliases {
                serial3 = &uart4;
                mmc2 = &usdhc3;
        };

        cpus {
                #address-cells = <1>;
                #size-cells = <0>;

                A53_2: cpu@2 {
                        device_type = "cpu";
                        compatible = "arm,cortex-a53";
                        reg = <0x2>;
                        clock-latency = <61036>; /* two CLK32 periods */
                        next-level-cache = <&A53_L2>;
                        enable-method = "psci";
                        #cooling-cells = <2>;
                };

                A53_3: cpu@3 {
                        device_type = "cpu";
                        compatible = "arm,cortex-a53";
                        reg = <0x3>;
                        clock-latency = <61036>; /* two CLK32 periods */
                        next-level-cache = <&A53_L2>;
                        enable-method = "psci";
                        #cooling-cells = <2>;
                };

                A53_L2: l2-cache0 {
                        compatible = "cache";
                };
        };

        psci {
                compatible = "arm,psci-1.0";
                method = "smc";
        };

        gic: interrupt-controller@38800000 {
                compatible = "arm,gic-v3";
                reg = <0x0 0x38800000 0 0x10000>, /* GIC Dist */
                      <0x0 0x38880000 0 0xC0000>; /* GICR (RD_base + SGI_base) */
                #interrupt-cells = <3>;
                interrupt-controller;
                interrupts = <GIC_PPI 9 IRQ_TYPE_LEVEL_HIGH>;
                interrupt-parent = <&gic>;
        };

        timer {
                compatible = "arm,armv8-timer";
                interrupts = <GIC_PPI 13 (GIC_CPU_MASK_SIMPLE(6) | IRQ_TYPE_LEVEL_LOW)>, /* Physical Secure */
                             <GIC_PPI 14 (GIC_CPU_MASK_SIMPLE(6) | IRQ_TYPE_LEVEL_LOW)>, /* Physical Non-Secure */
                             <GIC_PPI 11 (GIC_CPU_MASK_SIMPLE(6) | IRQ_TYPE_LEVEL_LOW)>, /* Virtual */
                             <GIC_PPI 10 (GIC_CPU_MASK_SIMPLE(6) | IRQ_TYPE_LEVEL_LOW)>; /* Hypervisor */
                clock-frequency = <8333333>;
        };

        clk_dummy: clock@7 {
                compatible = "fixed-clock";
                #clock-cells = <0>;
                clock-frequency = <0>;
                clock-output-names = "clk_dummy";
        };

        /* The clocks are configured by 1st OS */
        clk_400m: clock@8 {
                compatible = "fixed-clock";
                #clock-cells = <0>;
                clock-frequency = <200000000>;
                clock-output-names = "200m";
        };

        clk_266m: clock@9 {
                compatible = "fixed-clock";
                #clock-cells = <0>;
                clock-frequency = <266000000>;
                clock-output-names = "266m";
        };

        osc_24m: clock@1 {
                compatible = "fixed-clock";
                #clock-cells = <0>;
                clock-frequency = <24000000>;
                clock-output-names = "osc_24m";
        };

        pci@fd700000 {
                compatible = "pci-host-ecam-generic";
                device_type = "pci";
                bus-range = <0 0>;
                #address-cells = <3>;
                #size-cells = <2>;
                #interrupt-cells = <1>;
                interrupt-map-mask = <0 0 0 7>;
                interrupt-map = <0 0 0 1 &gic GIC_SPI 154 IRQ_TYPE_EDGE_RISING>,
                                <0 0 0 2 &gic GIC_SPI 155 IRQ_TYPE_EDGE_RISING>,
                                <0 0 0 3 &gic GIC_SPI 156 IRQ_TYPE_EDGE_RISING>,
                                <0 0 0 4 &gic GIC_SPI 157 IRQ_TYPE_EDGE_RISING>;
                reg = <0x0 0xfd700000 0x0 0x100000>;
                ranges = <0x02000000 0x00 0x10000000 0x0 0x10000000 0x00 0x10000>;
        };

        soc@0 {
                compatible = "simple-bus";
                #address-cells = <1>;
                #size-cells = <1>;
                ranges = <0x0 0x0 0x0 0x3e000000>;

                aips3: bus@30800000 {
                        compatible = "simple-bus";
                        reg = <0x30800000 0x400000>;
                        #address-cells = <1>;
                        #size-cells = <1>;
                        ranges;

                        uart4: serial@30a60000 {
                                compatible = "fsl,imx8mp-uart", "fsl,imx6q-uart";
                                reg = <0x30a60000 0x10000>;
                                interrupts = <GIC_SPI 29 IRQ_TYPE_LEVEL_HIGH>;
                                status = "disabled";
                        };

                        usdhc3: mmc@30b60000 {
                                compatible = "fsl,imx8mm-usdhc", "fsl,imx7d-usdhc";
                                reg = <0x30b60000 0x10000>;
                                interrupts = <GIC_SPI 24 IRQ_TYPE_LEVEL_HIGH>;
                                fsl,tuning-start-tap = <20>;
                                fsl,tuning-step= <2>;
                                status = "disabled";
                        };
                };
        };
};

&uart4 {
        clocks = <&osc_24m>,
                <&osc_24m>;
        clock-names = "ipg", "per";
        status = "okay";
};

&usdhc3 {
        clocks = <&clk_dummy>,
                <&clk_266m>,
                <&clk_400m>;
        clock-names = "ipg", "ahb", "per";
        bus-width = <8>;
        non-removable;
        status = "okay";
};
```



## 七、启动nonroot

连接硬件：打开两个串口窗口。
`cat non.sh`

```C
#!/bin/bash
insmod /home/comet/OK8MP_linux_kernel/extra/jailhouse/driver/jailhouse.ko
jailhouse disable
jailhouse enable /home/comet/OK8MP_linux_kernel/extra/jailhouse/configs/arm64/imx8mp.cell
export PATH=$PATH:/home/comet/OK8MP_linux_kernel/extra/jailhouse/tools/
jailhouse cell linux /home/comet/OK8MP_linux_kernel/extra/jailhouse/configs/arm64/imx8mp-linux-demo.cell /home/comet/OK8MP_linux_kernel/arch/arm64/boot/Image -d /home/comet/OK8MP_linux_kernel/arch/arm64/boot/dts/freescale/imx8mp-evk-inmate.dtb -c "clk_ignore_unused console=ttymxc3,115200 earlycon=ec_imx6q,0x30890000,115200  root=/dev/mmcblk2p2 rootwait rw"
```

