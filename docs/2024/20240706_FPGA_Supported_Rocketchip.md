# FPGA zcu102

Author: 杨竣轶(Jerry) github.com/comet959

```shell
# Before, Install vivado 2022.2 software
# Ubuntu 20.04 can work fine
sudo apt update

git clone https://github.com/U-interrupt/uintr-rocket-chip.git
cd uintr-rocket-chip
git submodule update --init --recursive
export RISCV=/opt/riscv64
git checkout 98e9e41
vim digilent-vivado-script/config.ini # Env Config

make checkout
make clean
make build

# Use vivado to open the vivado project, then change the top file, run synthesis, run implementation, generate bitstream.
# Connect the zcu102 - Jtag and Uart on your PC.
# Use dd command to flash the image include boot and rootfs part.
# Change the boot button mode to (On Off Off Off)
# Boot the power.

sudo screen /dev/ttyUSB0 115200 # Aarch64 Core Uart
sudo screen /dev/ttyUSB2 115200 # Riscv Core Uart

# On /dev/ttyUSB0
cd uintr-rocket-chip
./load-and-reset.sh # Label: #1

# Focus on ttyUSB2, then you will see the Riscv Linux Boot Msg.

```



## 在RocketChip中开启H扩展

```shell
vim path/to/repo/common/src/main/scala/Configs.scala
```

```scala
// change
class UintrConfig extends Config(
  new WithNBigCores(4) ++
    new WithNExtTopInterrupts(6) ++
    new WithTimebase((BigInt(10000000))) ++ // 10 MHz
    new WithDTS("freechips.rocketchip-unknown", Nil) ++
    new WithUIPI ++
    new WithCustomBootROM(0x10000, "../common/boot/bootrom/bootrom.img") ++
    new WithDefaultMemPort ++
    new WithDefaultMMIOPort ++
    new WithDefaultSlavePort ++
    new WithoutTLMonitors ++
    new WithCoherentBusTopology ++
    new BaseSubsystemConfig
)

// to

class UintrConfig extends Config(
  new WithHypervisor ++
  new WithNBigCores(4) ++
    new WithNExtTopInterrupts(6) ++
    new WithTimebase((BigInt(10000000))) ++ // 10 MHz
    new WithDTS("freechips.rocketchip-unknown", Nil) ++
    new WithUIPI ++
    new WithCustomBootROM(0x10000, "../common/boot/bootrom/bootrom.img") ++
    new WithDefaultMemPort ++
    new WithDefaultMMIOPort ++
    new WithDefaultSlavePort ++
    new WithoutTLMonitors ++
    new WithCoherentBusTopology ++
    new BaseSubsystemConfig
)

```



## This is the suppliment for debug the above error.

### 1. Make a clean initrd

```shell
cd busybox
make defconfig
make menuconfig # (enable Busybox Settings ---> Build Options ---> Build BusyBox as a static binary (no shared libs) ---> yes)
# Init Utilities  --->
#        [*] init
#
# Networking Utilities  --->
#        [ ] inetd
#
# Shells  --->
#        [*] ash
#
make
make install

mkdir $HOME/initrd
cd $HOME/initrd
mkdir -p dev bin sbin etc proc sys usr/bin usr/sbin
cp -a $BUSYBOX_BUILD/_install/* .
rm linuxrc
sudo mknod -m 644 dev/console c 5 1
sudo mknod -m 644 dev/loop0 b 7 0

# init sh
`
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo -e "Hello World\n"
exec /bin/sh
`

find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../rootfs.cpio.gz
```

### 2. Make a riscv kernel image within initrd.

```shell
git clone https://github.com/BOSC-Hvisor/linux.git # this kernel with kvm and tools.
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
# .config
## CONFIG_BLK_DEV_INITRD=y
## CONFIG_INITRAMFS_SOURCE="/home/jerry/work/image/rootfs.cpio.gz" # You should change this path to your own initrn path.
## CONFIG_BLK_DEV_LOOP=y
## CONFIG_BLK_DEV_LOOP_MIN_COUNT=8
## CONFIG_BLK_DEV_RAM=y
## CONFIG_BLK_DEV_RAM_COUNT=16
## CONFIG_BLK_DEV_RAM_SIZE=4096

make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j16
ls -lh arch/riscv/boot/Image
ls -lh arch/riscv/kvm/kvm.ko
```


### 3. Set up your dts.

```dts
/dts-v1/;

/ {
        #address-cells = <2>;
        #size-cells = <2>;
        compatible = "freechips,rocket-chip-dev";
        model = "freechips,rocket-chip-zcu102";

        chosen {
                bootargs = "earlycon console=ttyS0,115200 root=/dev/ram1 rdinit=/init";
                stdout-path = "serial0:115200n8";
        };

        aliases {
                serial0 = "/soc/serial@60000000";
                ethernet1 = &xxv_ethernet_0;
        };

        L19: cpus {
                #address-cells = <1>;
                #size-cells = <0>;
                timebase-frequency = <10000000>;
                L4: cpu@0 {
                        clock-frequency = <100000000>;
                        compatible = "sifive,rocket0", "riscv";
                        d-cache-block-size = <64>;
                        d-cache-sets = <64>;
                        d-cache-size = <16384>;
                        d-tlb-sets = <1>;
                        d-tlb-size = <32>;
                        device_type = "cpu";
                        hardware-exec-breakpoint-count = <1>;
                        i-cache-block-size = <64>;
                        i-cache-sets = <64>;
                        i-cache-size = <16384>;
                        i-tlb-sets = <1>;
                        i-tlb-size = <32>;
                        mmu-type = "riscv,sv39";
                        next-level-cache = <&L14>;
                        reg = <0x0>;
                        riscv,isa = "rv64imafdch";
                        riscv,pmpgranularity = <4>;
                        riscv,pmpregions = <8>;
                        status = "okay";
                        timebase-frequency = <10000000>;
                        tlb-split;
                        L2: interrupt-controller {
                                #interrupt-cells = <1>;
                                compatible = "riscv,cpu-intc";
                                interrupt-controller;
                        };
                };
                L7: cpu@1 {
                        clock-frequency = <100000000>;
                        compatible = "sifive,rocket0", "riscv";
                        d-cache-block-size = <64>;
                        d-cache-sets = <64>;
                        d-cache-size = <16384>;
                        d-tlb-sets = <1>;
                        d-tlb-size = <32>;
                        device_type = "cpu";
                        hardware-exec-breakpoint-count = <1>;
                        i-cache-block-size = <64>;
                        i-cache-sets = <64>;
                        i-cache-size = <16384>;
                        i-tlb-sets = <1>;
                        i-tlb-size = <32>;
                        mmu-type = "riscv,sv39";
                        next-level-cache = <&L14>;
                        reg = <0x1>;
                        riscv,isa = "rv64imafdch";
                        riscv,pmpgranularity = <4>;
                        riscv,pmpregions = <8>;
                        status = "okay";
                        timebase-frequency = <10000000>;
                        tlb-split;
                        L5: interrupt-controller {
                                #interrupt-cells = <1>;
                                compatible = "riscv,cpu-intc";
                                interrupt-controller;
                        };
                };
                L6: cpu@2 {
                        clock-frequency = <100000000>;
                        compatible = "sifive,rocket0", "riscv";
                        d-cache-block-size = <64>;
                        d-cache-sets = <64>;
                        d-cache-size = <16384>;
                        d-tlb-sets = <1>;
                        d-tlb-size = <32>;
                        device_type = "cpu";
                        hardware-exec-breakpoint-count = <1>;
                        i-cache-block-size = <64>;
                        i-cache-sets = <64>;
                        i-cache-size = <16384>;
                        i-tlb-sets = <1>;
                        i-tlb-size = <32>;
                        mmu-type = "riscv,sv39";
                        next-level-cache = <&L14>;
                        reg = <0x2>;
                        riscv,isa = "rv64imafdch";
                        riscv,pmpgranularity = <4>;
                        riscv,pmpregions = <8>;
                        status = "okay";
                        timebase-frequency = <10000000>;
                        tlb-split;
                        L3: interrupt-controller {
                                #interrupt-cells = <1>;
                                compatible = "riscv,cpu-intc";
                                interrupt-controller;
                        };
                };
                L12: cpu@3 {
                        clock-frequency = <100000000>;
                        compatible = "sifive,rocket0", "riscv";
                        d-cache-block-size = <64>;
                        d-cache-sets = <64>;
                        d-cache-size = <16384>;
                        d-tlb-sets = <1>;
                        d-tlb-size = <32>;
                        device_type = "cpu";
                        hardware-exec-breakpoint-count = <1>;
                        i-cache-block-size = <64>;
                        i-cache-sets = <64>;
                        i-cache-size = <16384>;
                        i-tlb-sets = <1>;
                        i-tlb-size = <32>;
                        mmu-type = "riscv,sv39";
                        next-level-cache = <&L14>;
                        reg = <0x3>;
                        riscv,isa = "rv64imafdch";
                        riscv,pmpgranularity = <4>;
                        riscv,pmpregions = <8>;
                        status = "okay";
                        timebase-frequency = <10000000>;
                        tlb-split;
                        L20: interrupt-controller {
                                #interrupt-cells = <1>;
                                compatible = "riscv,cpu-intc";
                                interrupt-controller;
                        };
                };
        };
        L14: memory@80000000 {
                device_type = "memory";
                reg = <0x0 0x80000000 0x0 0x10000000>;
        };
        L18: soc {
                #address-cells = <2>;
                #size-cells = <2>;
                compatible = "freechips.rocketchip-unknown-soc", "simple-bus";
                ranges;
                L9: clint@2000000 {
                        compatible = "riscv,clint0";
                        interrupts-extended = <&L2 3 &L2 7 &L5 3 &L5 7 &L3 3 &L3 7 &L20 3 &L20 7>;
                        reg = <0x0 0x2000000 0x0 0x10000>;
                        reg-names = "control";
                };
                L11: debug-controller@0 {
                        compatible = "sifive,debug-013", "riscv,debug-013";
                        debug-attach = "dmi";
                        interrupts-extended = <&L2 65535 &L5 65535 &L3 65535 &L20 65535>;
                        reg = <0x0 0x0 0x0 0x1000>;
                        reg-names = "control";
                };
                L1: error-device@3000 {
                        compatible = "sifive,error0";
                        reg = <0x0 0x3000 0x0 0x1000>;
                };
                L13: external-interrupts {
                        interrupt-parent = <&L8>;
                        interrupts = <1 2 3 4 5 6>;
                };
                L8: interrupt-controller@c000000 {
                        #interrupt-cells = <1>;
                        compatible = "riscv,plic0";
                        interrupt-controller;
                        interrupts-extended = <&L2 11 &L2 9 &L2 8 &L5 11 &L5 9 &L5 8 &L3 11 &L3 9 &L3 8 &L20 11 &L20 9 &L20 8>;
                        reg = <0x0 0xc000000 0x0 0x4000000>;
                        reg-names = "control";
                        riscv,max-priority = <7>;
                        riscv,ndev = <6>;
                };
                L15: mmio-port-axi4@60000000 {
                        #address-cells = <1>;
                        #size-cells = <1>;
                        compatible = "simple-bus";
                        ranges = <0x60000000 0x0 0x60000000 0x20000000>;
                };
                L16: rom@10000 {
                        compatible = "sifive,rom0";
                        reg = <0x0 0x10000 0x0 0x10000>;
                        reg-names = "mem";
                };
                L0: subsystem_pbus_clock {
                        #clock-cells = <0>;
                        clock-frequency = <100000000>;
                        clock-output-names = "subsystem_pbus_clock";
                        compatible = "fixed-clock";
                };
                L10: uintc@3000000 {
                        #interrupt-cells = <1>;
                        compatible = "riscv,uintc0";
                        interrupt-controller;
                        interrupts-extended = <&L2 0 &L5 0 &L3 0 &L20 0>;
                        reg = <0x0 0x3000000 0x0 0x1000>;
                        reg-names = "control";
                };
                L17: serial@60000000 {
                        compatible = "ns16550a";
                        current-speed = <115200>;
                        interrupt-parent = <&L8>;
                        interrupts = <1>;
                        reg = <0x00 0x60000000 0x00 0x2000>;
                        reg-shift = <2>;
                        reg-offset = <0x1000>;
                        clock-frequency = <100000000>;
                };
        };

        amba_pl: amba_pl@0 {
                #address-cells = <2>;
                #size-cells = <2>;
                compatible = "simple-bus";
                ranges ;
                axi_dma_0: dma@60100000 {
                        #dma-cells = <1>;
                        clock-names = "s_axi_lite_aclk", "m_axi_sg_aclk", "m_axi_mm2s_aclk", "m_axi_s2mm_aclk";
                        clocks = <&pl_clk_100m>, <&pl_clk_100m>, <&misc_clk_0>, <&misc_clk_0>;
                        compatible = "xlnx,eth-dma";
                        interrupt-names = "mm2s_introut", "s2mm_introut";
                        interrupt-parent = <&L8>;
                        interrupts = <2 3>;
                        reg = <0x0 0x60100000 0x0 0x10000>;
                        xlnx,addrwidth = /bits/ 8 <0x20>;
                        xlnx,include-dre ;
                        xlnx,num-queues = /bits/ 16 <0x1>;
                };
                misc_clk_0: misc_clk_0 {
                        #clock-cells = <0>;
                        clock-frequency = <156250000>;
                        compatible = "fixed-clock";
                };
                pl_clk_100m: clk_100m {
                        #clock-cells = <0>;
                        clock-frequency = <100000000>;
                        compatible = "fixed-clock";
                };
                xxv_ethernet_0: ethernet@60110000 {
                        axistream-connected = <&axi_dma_0>;
                        axistream-control-connected = <&axi_dma_0>;
                        clock-frequency = <100000000>;
                        clock-names = "rx_core_clk", "dclk", "s_axi_aclk", "s_axi_lite_aclk", "m_axi_sg_aclk", "m_axi_mm2s_aclk", "m_axi_s2mm_aclk";
                        clocks = <&misc_clk_0>,  <&pl_clk_100m>,  <&pl_clk_100m>, <&pl_clk_100m>, <&pl_clk_100m>, <&misc_clk_0>, <&misc_clk_0>;
                        compatible = "xlnx,xxv-ethernet-4.1", "xlnx,xxv-ethernet-1.0";
                        device_type = "network";
                        interrupt-names = "mm2s_introut", "s2mm_introut";
                        interrupt-parent = <&L8>;
                        interrupts = <2 3>;
                        local-mac-address = [00 0a 35 00 00 01];
                        phy-mode = "base-r";
                        reg = <0x0 0x60110000 0x0 0x10000>;
                        xlnx = <0x0>;
                        xlnx,add-gt-cntrl-sts-ports = <0x0>;
                        xlnx,anlt-clk-in-mhz = <0x64>;
                        xlnx,axis-tdata-width = <0x40>;
                        xlnx,axis-tkeep-width = <0x7>;
                        xlnx,base-r-kr = "BASE-R";
                        xlnx,channel-ids = "1";
                        xlnx,clocking = "Asynchronous";
                        xlnx,cmac-core-select = "CMACE4_X0Y0";
                        xlnx,core = "Ethernet MAC+PCS/PMA 64-bit";
                        xlnx,data-path-interface = "AXI Stream";
                        xlnx,enable-datapath-parity = <0x0>;
                        xlnx,enable-gt-board-interface = <0x0>;
                        xlnx,enable-pipeline-reg = <0x0>;
                        xlnx,enable-preemption = <0x0>;
                        xlnx,enable-preemption-fifo = <0x0>;
                        xlnx,enable-rx-flow-control-logic = <0x0>;
                        xlnx,enable-time-stamping = <0x0>;
                        xlnx,enable-tx-flow-control-logic = <0x0>;
                        xlnx,enable-vlane-adjust-mode = <0x0>;
                        xlnx,family-chk = "zynquplus";
                        xlnx,fast-sim-mode = <0x0>;
                        xlnx,gt-diffctrl-width = <0x4>;
                        xlnx,gt-drp-clk = "100.00";
                        xlnx,gt-group-select = "Quad X0Y0";
                        xlnx,gt-location = <0x1>;
                        xlnx,gt-ref-clk-freq = "156.25";
                        xlnx,gt-type = "GTH";
                        xlnx,gtm-group-select = "NA";
                        xlnx,include-auto-neg-lt-logic = "None";
                        xlnx,include-axi4-interface = <0x1>;
                        xlnx,include-dre ;
                        xlnx,include-fec-logic = <0x0>;
                        xlnx,include-hybrid-cmac-rsfec-logic = <0x0>;
                        xlnx,include-rsfec-logic = <0x0>;
                        xlnx,include-shared-logic = <0x1>;
                        xlnx,include-statistics-counters = <0x1>;
                        xlnx,include-user-fifo = <0x1>;
                        xlnx,ins-loss-nyq = <0x1e>;
                        xlnx,lane1-gt-loc = "X0Y4";
                        xlnx,lane2-gt-loc = "NA";
                        xlnx,lane3-gt-loc = "NA";
                        xlnx,lane4-gt-loc = "NA";
                        xlnx,line-rate = <0xa>;
                        xlnx,mii-ctrl-width = <0x4>;
                        xlnx,mii-data-width = <0x20>;
                        xlnx,num-of-cores = <0x1>;
                        xlnx,num-queues = /bits/ 16 <0x1>;
                        xlnx,ptp-clocking-mode = <0x0>;
                        xlnx,ptp-operation-mode = <0x2>;
                        xlnx,runtime-switch = <0x0>;
                        xlnx,rx-eq-mode = "AUTO";
                        xlnx,rxmem = <0x40000>;
                        xlnx,statistics-regs-type = <0x0>;
                        xlnx,switch-1-10-25g = <0x0>;
                        xlnx,sys-clk = <0xfa0>;
                        xlnx,tx-latency-adjust = <0x0>;
                        xlnx,tx-total-bytes-width = <0x4>;
                        xlnx,xgmii-interface = <0x1>;
                        zclock-names = "NULL";
                        zclocks = "NULL";
                        xxv_ethernet_0_mdio: mdio {
                                #address-cells = <1>;
                                #size-cells = <0>;
                        };
                };
        };
};

```

```shell
dtc -I dts -O dtb -o ./rocketchip.dtb ./rocketchip.dts
```


### 4. Build KVMTOOL

```shell
git clone https://git.kernel.org/pub/scm/linux/kernel/git/will/kvmtool.git
export ARCH=riscv
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
cd kvmtool
make lkvm-static
${CROSS_COMPILE}strip lkvm-static
cd ..
```

The above commands will create `kvmtool/lkvm-static` which will be our user-space tool for KVM RISC-V.


### 5. Make a firmware for riscv soft core.

```shell
make -C /path/to/opensbi clean &&  make FW_PAYLOAD=y PLATFORM=zcu102 CROSS_COMPILE=riscv64-unknown-linux-gnu- FW_PAYLOAD_PATH=/path/to/Image FW_FDT_PATH=/path/to/rocketchip.dtb -j8
```

You can copy this **fw_payload.bin** to replace the firmware and jump to the **Label: #1**, then you can rerun the test.

