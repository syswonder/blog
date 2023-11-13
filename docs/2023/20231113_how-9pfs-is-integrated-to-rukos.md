# RukOS中的9pfs

时间：2023.11.13

作者：ZhengWu

联系方式：<hello_weekday@163.com>

## 1 Abstract

* 9PFS即9p-filesystem，是一种基于9P协议实现对server端文件系统读写的文件系统。
* 本文主要介绍了RukOS的9pfs模块的使用方式，以及它的具体实现原理。为了使得RukOS原有的文件系统可以实现挂载（mount）更灵活的文件系统，9pfs的实现过程还对原有文件系统的初始化方式进行了改变。
* 在RukOS中，9pfs对应于两个不同的特性（feature），即`virtio-9p`和`net-9p`。其中`virtio-9p`是基于hypervisor中的`virtio-9p`虚拟设备进行通信来控制host文件系统的读写，而`net-9p`则通过网络TCP协议实现连接通信并控制server端文件系统的读写。这两个不同特性可以同时启用并正常工作。

## 2 Guidline

本章将主要介绍如何在RukOS中启用9pfs。virtio-9p和net-9p是分别介绍的，其内容都是相对完整的，作为9pfs的使用者只需要看其中一节即可。
9pfs至少支持x86_64和aarch64两种架构。在不同架构下9pfs的使用方式应当是十分相似的，为了简化篇幅，本章将主要是在当前(2023.11.13)的make编译命令环境下针对aarch64展开。

---

### 2.1 **virtio-9p**

使用`virtio-9p`需要在APP添加该feature。

对于C APP，是在APP目录下的features.txt下加入一行：

```txt
virtio-9p
```

对于rust APP，则需要在APP目录下的Cargo.toml的axstd特性中加入`virtio-9p`特性，具体如下（该例子是`application/fs/shell`中的Cargo.toml）:

```txt
axstd = { path = "../../../ulib/axstd", features = ["alloc", "fs", "virtio-9p"], optional = true }
```

然后，为了使得host（当前是qemu-system-aarch64）启用virtio-9p的后端，需要在`make`命令行中加入参数`V9P=y`。

例如:

```shell
make A=xxxx LOG=error V9P=y V9P_PATH=xxxx ARCH=xxxx
```

`V9P_PATH`是一个**环境变量**，对应于host的文件目录映射路径，如果不设置该参数则为为当前目录`./`。

> 注意：
> 当前版本中将**block设备**作为根目录仍是必选项，**这或许在后续版本中会有所更改**。
> 因此，启用`fs`时将会自动启用blk-fs作为根目录，为此你还需要创建一个`disk.img`（对应命令为`make disk_img`），以及在`make`命令中加入`BLK=y`。
> 另外，通常情况下使用`9pfs`需要同时启用`fs`的特性，除非你仅通过`ax9p`中的接口来直接调用`9pfs`。

完成以上步骤在理论上即可在根目录下创建一个`v9fs`的目录，该目录应当包含host后端所映射的文件目录。

此外，virtio-9p涉及到了一些可设置的环境变量。这些环境变量都存在一个默认值，有时并不需要显示的设置，这些环境变量及其默认值分别为：

```makefile
V9P_PATH ?= ./           # 对应于host的文件目录映射路径，默认为当前目录
ANAME_9P ?= ./           # 对应与tattach时的aname路径参数，在当前版本下的qemu可以设置为任意值，但某些情况（host存在多个映射路径）下可能需要设置为选择的对应host文件目录路径
PROTOCOL_9P ?= 9P2000.L  # 默认选择的9P协议，如`9P2000.L`和`9P2000.u`，区分大小写
```

---

### 2.2 **net-9p**

使用`net-9p`需要在APP添加该feature。

对于C APP，是在APP目录下的features.txt下加入一行：

```txt
net-9p
```

对于rust APP，则需要在APP目录下的Cargo.toml的axstd特性中加入`net-9p`特性，具体如下（该例子是`application/fs/shell`中的Cargo.toml）:

```txt
axstd = { path = "../../../ulib/axstd", features = ["alloc", "fs", "net-9p"], optional = true }
```

然后，`net-9p`需要通过TCP网络协议实现通信，因此需要在`make`命令行中加入参数`NET=y`。

例如:

```shell
make A=xxxx LOG=error NET=y ARCH=xxxx
```

> 注意：
> 当前版本中将**block设备**作为根目录仍是必选项，**这或许在后续版本中会有所更改**。
> 因此，启用`fs`时将会自动启用blk-fs作为根目录，为此你还需要创建一个`disk.img`（对应命令为`make disk_img`），以及在`make`命令中加入`BLK=y`。
> 另外，通常情况下使用`9pfs`需要同时启用`fs`的特性，除非你仅通过`ax9p`中的接口来直接调用`9pfs`。

完成以上步骤在理论上即可在根目录下创建一个`n9fs`的目录，该目录应当包含sever后端所映射的文件目录。

此外，net-9p涉及到了一些可设置的环境变量。这些环境变量都存在一个默认值，有时并不需要显示的设置，这些环境变量及其默认值分别为：

```makefile
NET_9P_ADDR ?= 127.0.0.1:564 # 9P sever端的IP地址和PORT端口号，默认值为 127.0.0.1:564。
ANAME_9P ?= ./           # 对应与tattach时的aname路径参数，能需要设置为具体选择的对应host文件目录路径
PROTOCOL_9P ?= 9P2000.L  # 默认选择的9P协议，如`9P2000.L`和`9P2000.u`，区分大小写
```

---

## 3 Principle of implement

本章将主要介绍9pfs在RukOS中是如何实现的。

### 3.1 Protocol

9P事实上是一种用来进行文件读写的通信协议，它只是在会话层的。这就意味着在本质上9P与物理层、数据链路层等无关。9P通信协议的版本存在`9P2000`、`9P2000.L`和`9P2000.u`三种，其协议内容可以参考以下链接：

* [9P2000的基本协议内容](https://ericvh.github.io/9p-rfc/rfc9p2000.html)

* [9P2000.L拓展协议版本](https://github.com/chaos/diod/blob/master/protocol.md)

* [9P2000.u拓展协议版本](http://ericvh.github.io/9p-rfc/rfc9p2000.u.html)

这些链接主要包括了一些帧格式及不同通信帧所对应的相关操作。

### 3.2 Framework

* `/module/ax9p/src/drv`: 9pfs的驱动实现，主要是协议帧格式的打包
* `/module/ax9p/src/fs`:  9pfs的文件系统实现
* `/module/ax9p/src/lib.rs`: 9pfs初始化函数的实现，这些初始化函数已经被添加在了`axruntime`中。
* `/crate/driver_9p`: 9pfs的驱动的泛型定义
* `/module/ax9p/net`: net-9p的驱动实现
* `/crate/driver_virtio/src/v9p.rs`: virtio-9p的驱动实现

---

## 4 Filesystem initialization

（在`/module/axruntime/src/lib.rs`中）

为了兼容除disk文件系统外的涉及到设备驱动的文件系统，对RukOS的改动是将原`init_rootfs`函数拆分为`init_filesystems`、`prepare_commonfs`和`init_filesystems`三个函数，此外额外新增了一个`mount_points`数组。这样做的目的是方便增加涉及到设备驱动的9pfs及未来可能涉及到的其它文件系统，具体对应代码为：

```rust
        #[cfg(feature = "fs")]
        {
            extern crate alloc;
            use alloc::vec::Vec;
            // By default, mount_points[0] will be rootfs
            let mut mount_points: Vec<axfs::MountPoint> = Vec::new();

            // setup and initialize blkfs as one mountpoint for rootfs
            mount_points.push(axfs::init_filesystems(all_devices.block));
            axfs::prepare_commonfs(&mut mount_points);

            // setup and initialize 9pfs as mountpoint
            #[cfg(feature = "virtio-9p")]
            mount_points.push(ax9p::init_virtio_9pfs(
                all_devices._9p,
                option_env!("AX_ANAME_9P").unwrap_or(""),
                option_env!("AX_PROTOCOL_9P").unwrap_or(""),
            ));
            #[cfg(feature = "net-9p")]
            mount_points.push(ax9p::init_net_9pfs(
                option_env!("AX_9P_ADDR").unwrap_or(""),
                option_env!("AX_ANAME_9P").unwrap_or(""),
                option_env!("AX_PROTOCOL_9P").unwrap_or(""),
            ));

            // setup and initialize rootfs
            axfs::init_filesystems(mount_points);
        }
```

其中`mount_points`数组是`MountPoint`类型的数组。每个元素包含了两个信息，即挂载点和文件系统所对应的实体变量：

```rust
/// mount point information
pub struct MountPoint {
    path: &'static str,
    fs: Arc<dyn VfsOps>,
}
```

挂载文件系统的逻辑是将挂载点和文件系统打包成`MountPoint`类型，并`push`进入`mount_points`这个数组。文件系统初始化最后调用`axfs::init_filesystems(mount_points)`，`init_filesystems`的最后将利用`mount_points`中的信息生成对应的根文件系统，且init_filesystems始终将`mount_points`第一个元素当作根目录挂载。因此理论上未来也可以将9pfs作为根目录生成根文件系统，即：

```rust
        #[cfg(feature = "fs")]
        {
            extern crate alloc;
            use alloc::vec::Vec;
            // By default, mount_points[0] will be rootfs
            let mut mount_points: Vec<axfs::MountPoint> = Vec::new();

            #[cfg(feature = "blkfs")]
            {
            // setup and initialize blkfs as one mountpoint for rootfs
            mount_points.push(axfs::init_filesystems(all_devices.block));
            axfs::prepare_commonfs(&mut mount_points);                
            }

            // setup and initialize 9pfs as mountpoint
            #[cfg(feature = "virtio-9p")]
            mount_points.push(ax9p::init_virtio_9pfs(
                all_devices._9p,
                option_env!("AX_ANAME_9P").unwrap_or(""),
                option_env!("AX_PROTOCOL_9P").unwrap_or(""),
            ));
            #[cfg(feature = "net-9p")]
            mount_points.push(ax9p::init_net_9pfs(
                option_env!("AX_9P_ADDR").unwrap_or(""),
                option_env!("AX_ANAME_9P").unwrap_or(""),
                option_env!("AX_PROTOCOL_9P").unwrap_or(""),
            ));

            // setup and initialize rootfs
            axfs::init_filesystems(mount_points);
        }
```

但当前版本没有这么做，理由是存在基于block文件系统的应用在先前并未添加`blkfs`这一`feature`，这可能导致一些潜在bug。
