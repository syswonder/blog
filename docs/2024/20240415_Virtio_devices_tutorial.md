# 在hvisor上使用Virtio-blk和net设备
时间：2024/4/15

作者：李国玮

摘要：介绍如何在hvisor上使用Virtio-blk和Virtio-net设备

### 磁盘镜像的要求

root Linux的磁盘镜像至少需要安装以下几个包：

```
apt-get install git sudo vim bash-completion \
kmod net-tools iputils-ping resolvconf ntpdate
```

### linux Image的要求

在编译root linux的镜像前, 在.config文件中把CONFIG_IPV6和CONFIG_BRIDGE的config都改成y, 以支持在root linux中创建网桥和tap设备。具体操作如下：

```shell
git clone https://github.com/torvalds/linux -b v5.4 --depth=1
cd linux
git checkout v5.4
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- defconfig
# 在.config中增加一行
CONFIG_BLK_DEV_RAM=y
# 修改.config的两个CONFIG参数
CONFIG_IPV6=y
CONFIG_BRIDGE=y
# 编译
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- Image -j$(nproc)
```

## 使用Virtio-blk设备

目前支持在non root Linux中使用Virtio-blk设备作为根文件系统。请仔细查看hvisor-tool的README。

## 使用Virtio-net设备

### 创建网络拓扑

首先需要在root Linux中创建网络拓扑图，以使用Tap设备和网桥设备连通真实网卡。在root Linux中执行以下指令：

```shell
mount -t proc proc /proc
mount -t sysfs sysfs /sys
ip link set eth0 up
dhclient eth0
brctl addbr br0
brctl addif br0 eth0
ifconfig eth0 0
dhclient br0
ip tuntap add dev tap0 mode tap
brctl addif br0 tap0
ip link set dev tap0 up
```

便可创建`tap0设备<-->网桥设备<-->真实网卡`的网络拓扑。

### 启动non-root-linux

请仔细查看hvisor-tool的README，以启动Non root Linux和Virtio守护进程。

在non root linux的命令行执行，以启动网卡：

```shell
mount -t proc proc /proc
mount -t sysfs sysfs /sys
ip link set eth0 up
dhclient eth0
```

可以通过以下指令测试网络的连通：

```
curl www.baidu.com
ping www.baidu.com
```

