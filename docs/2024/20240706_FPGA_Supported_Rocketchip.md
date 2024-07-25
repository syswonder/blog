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
./load-and-reset.sh

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


