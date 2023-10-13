# QEMU模拟ARM64内核

## 一、安装交叉编译器 aarch64-none-linux-gnu- 10.3

网址：[https://developer.arm.com/downloads/-/gnu-a](https://developer.arm.com/downloads/-/gnu-a) 

工具选择：AArch64 GNU/Linux target (aarch64-none-linux-gnu) 

下载链接：[https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC) 

```bash
wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
tar xvf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
ls gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
```

安装完成，记住路径，例如在：/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-，之后都会使用这个路径。

## 二、编译安装QEMU 7.2

1. wget [https://download.qemu.org/qemu-7.2.1.tar.xz](https://download.qemu.org/qemu-7.2.1.tar.xz)
2. tar 解压
3. mkdir build %% cd build
4. ../qemu-7.2.0/configure --enable-kvm --enable-slirp --enable-debug --target-list=aarch64-softmmu,x86_64-softmmu
5. make -j2

> 7.0.0版本的qemu也可以，再低可能就不行了

## 三、编译Kernel 5.4

```bash
git clone https://github.com/torvalds/linux -b v5.4 --depth=1
cd linux
git checkout v5.4
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- defconfig
```

为了能使linux产生一个/dev/ram0，便于后续使用，此时需要编辑本地.config文件，添加一行：

```
CONFIG_BLK_DEV_RAM=y
```

之后进行编译：

```shell
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- Image -j$(nproc)
```

编译完毕，内核文件位于：arch/arm64/boot/Image。记住整个linux文件夹所在的路径，例如：home/korwylee/lgw/hypervisor/linux。

## 四、基于ubuntu 20.04 arm64 base构建文件系统

busybox制作的文件系统过于简单（如没有apt工具），因此，我们需要使用更丰富的ubuntu文件系统来制作linux的根文件系统。

下载：[ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)  

链接：[http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)

```bash
wget http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz

mkdir rootfs
# 创建一个ubuntu.img
dd if=/dev/zero of=ubuntu-20.04-rootfs_ext4.img bs=1M count=8192 oflag=direct
mkfs.ext4 ubuntu-20.04-rootfs_ext4.img
# 将ubuntu.tar.gz放入已经挂载到rootfs上的ubuntu.img中
sudo mount -t ext4 ubuntu-20.04-rootfs_ext4.img rootfs/
sudo tar -xzf ubuntu-base-20.04.5-base-arm64.tar.gz -C rootfs/

# 让rootfs绑定和获取物理机的一些信息和硬件
sudo cp /usr/bin/qemu-system-aarch64 rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts

sudo chroot rootfs

apt-get update
apt-get install git sudo vim bash-completion -y
apt-get install net-tools ethtool ifupdown network-manager iputils-ping -y
apt-get install rsyslog resolvconf udev -y

# 如果上面软件包没有安装，至少要安装下面的包
apt-get install systemd -y

apt-get install build-essential git wget flex bison libssl-dev bc libncurses-dev kmod -y

adduser arm64
adduser arm64 sudo
echo "kernel-5_4" >/etc/hostname
echo "127.0.0.1 localhost" >/etc/hosts
echo "127.0.0.1 kernel-5_4">>/etc/hosts
dpkg-reconfigure resolvconf
dpkg-reconfigure tzdata
exit

sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

最后卸载挂载，完成根文件系统的制作。

## 五、编译jailhouse

```bash
sudo mount ubuntu-20.04-rootfs_ext4.img rootfs/
# 进入rootfs中的某个文件夹
```

编译rootfs:

```bash
# for x86
git clone https://github.com/siemens/jailhouse.git
cd jailhouse 
make

# for aarch64
git clone https://github.com/siemens/jailhouse.git
cd jailhouse 
# KDIR为之前编译的linux文件夹
make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- KDIR=/home/korwylee/lgw/hypervisor/linux

sudo make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- KDIR=/home/korwylee/lgw/hypervisor/linux DESTDIR=/home/korwylee/lgw/hypervisor/linux install 
```

说明：编译jailhouse一定要指定KDIR，说明sysroot目标，才可以编译成功，并且需要提前编译一个linux作为sysroot，否则默认从本机linux源码目录中去找相应的库代码。

## 六、启动QEMU

```bash
qemu-system-aarch64 \
	-machine virt,gic_version=3 \
	-machine virtualization=true \
	-cpu cortex-a57 \
	-machine type=virt \
	-nographic \
	-smp 16 \
	-m 1024 \
	-kernel ./linux/arch/arm64/boot/Image \
	-append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
	-drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
	-device virtio-blk-device,drive=hd0 \
	-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
	-device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01
```

其中kernel为之前编译好的linux镜像。

## 七、运行jailhouse

```shell
cd ~/jailhouse
sudo insmod driver/jailhouse.ko
sudo ./tools/jailhouse enable configs/arm64/qemu-arm64.cell
sudo ./tools/jailhouse disable
```

启动一个gic-demo：

```shell
sudo ./tools/jailhouse cell create configs/arm64/qemu-arm64-gic-demo.cell
sudo ./tools/jailhouse cell load gic-demo inmates/demos/arm64/gic-demo.bin
sudo ./tools/jailhouse cell start gic-demo
sudo ./tools/jailhouse cell destroy gic-demo
```

## 八、启动一个non-root-linux on qemu

目前只实现了在qemu里，启动一个带有内存文件系统的non-root-linux，具体教程如下：

### 8.1编译 busybox 1.36.0 （文件系统）

```bash
wget https://busybox.net/downloads/busybox-1.36.0.tar.bz2
tar -jxvf busybox-1.36.0.tar.bz2

cd busybox-1.36.0
make menuconfig
```

在弹出窗口中，依次进入并设定：

* Settings
  * [*] Build static binary (no shared libs)
  * Cross compiler prefix设置为：`/交叉编译器路径/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-`

效果为：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310131545422.png" alt="Untitled"  />

```bash
# 编译安装
make -j$(nproc) && make install
```

编译完毕后busybox生成在_install目录。

### 8.2 为文件系统创建console

```bash
cd _install
mkdir dev
cd dev
sudo mknod console c 5 1
sudo mknod null c 1 3
sudo mknod tty1 c 4 1
sudo mknod tty2 c 4 2
sudo mknod tty3 c 4 3
sudo mknod tty4 c 4 4
cd ..
mkdir -p etc/init.d/ 
cd etc/init.d; touch rcS
chmod +x rcS
vi rcS

# 压缩成cpio.gz文件系统
find . -print0 | cpio --null -ov --format=newc | gzip -9 > initramfs.cpio.gz
```

之后将initramfs.cpio.gz通过挂载根文件系统，传入到guest linux中。然后启动QEMU，并enable jailhouse。之后执行：

```bash
cd tools/
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/ram0 rdinit=/linuxrc" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb -i ../initramfs.cpio.gz
```

其中linux-Image是之前编译的linux镜像。注意还需要修改tools/jailhouse-cell-linux中class ARM64中这个函数为：

```python
@staticmethod
def get_uncompressed_kernel(kernel):
    return (kernel.read(), False)
```

## 附录

### 1. 配置qemu网络，与外部通信

```bash
+-----------------------------------------------------------------+
|  Host                                                           |
| +---------------------+                                         |
| |                     |                                         |
| | br0:                |                                         |
| |   192.168.0.32/24 +-----+                                   |
| |                     |     |                                   |
| +----+----------------+     |       +-------------------------+ |
|      |                      |       |  Guest                  | |
|      |                      |       | +---------------------+ | |
| +----+----------------+  +--+---+   | |                     | | |
| |                     |  |      |   | | eth0:               | | |
| | eth1:               |  | tap0 |   | |   192.168.0.33/24 | | |
| |   192.168.0.175/24   |  |      +-----+                     | | |
| |                     |  |      |   | +---------------------+ | |
| +---------------------+  +------+   +-------------------------+ |
+-----------------------------------------------------------------+
```

网络连接图

配置网桥 `br0`

```bash
sudo ip link add name br0 type bridge
sudo ip link set dev br0 down
sudo ip addr flush dev br0
sudo ip addr add 192.168.0.32/24 dev br0
sudo ip link set dev br0 up
```

配置 tap 设备 `tap0`

```bash
sudo ip tuntap add name tap0 mode tap
sudo ip link set dev tap0 up
```

将宿主机网络接口 `eth0`和 `tap0`接入网桥 `br0`

```bash
sudo ip link set eth1 master br0
sudo ip link set tap0 master br0
```

然后qemu启动虚拟机，在虚拟机内

```bash
sudo ifconfig eth0 up
sudo ifconfig eth0 192.168.0.33
```

现在可以实现guest ping通host

#### 优化网络，让其可以与外部通信 (暂不可信)

以上步骤完成后虚拟机可与宿主机所在网络的其他设备互连（包括宿主机），也可以通过指定的网关连接互联网，但是此时宿主机无法连接互联网，解决方法如下:

删除 `eth0`接口的默认网关：

```bash
sudo ip route del default dev eth1
```

为 `br0`添加默认网关：

```bash
sudo ip route add default via 192.168.0.1 dev br0
```

### 2. img扩容

当rootfs chroot空间不足时，需要扩容，按照以下步骤进行无损扩容：

```bash
# 首先取消挂载img
umount ./rootfs

## bug: umount: /root/rootfs: target is busy.
## 解决：umount ./rootfs -l 强行卸载（慎用）

dd if=/dev/zero of=add.img bs=1M count=4096 # 新建4G空间
cat add.img >> ubuntu-20.04-rootfs_ext4.img
e2fsck -f ubuntu-20.04-rootfs_ext4.img
resize2fs ubuntu-20.04-rootfs_ext4.img

mount ubuntu-20.04-rootfs_ext4.img rootfs
```
