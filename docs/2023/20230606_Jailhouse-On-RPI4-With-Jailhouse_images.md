# Jailhouse在树莓派硬件平台上的移植

时间：2023.6.6

作者：杨竣轶

## Preparation

🔧工具： Jailhouse-Image

☑️平台：Raspiberry Pi 4B 8GB

## 开始

```bash
git clone 
```

## jailhouse-images配置分析

## 详解dts文件在jailhouse中的使用🌟

```bash
## 原则：在Arm平台上，每一个Jail Linux(包括root/guest)都需要有一个相应的设备树支持其启动。
# root cell使用的设备树，与硬件上直接启动Linux使用的设备树别无二致，存在于Linux源码中，在编译Jailhouse时使用KDIR指定
# guest cell使用的设备树，在config/arm64/dts中可以看见，您可以根据不同的需要定制设备树文件（例如：设备直通、CPU指定），该设备树文件应该是Linux源码中设备树的“子”树。
# 关于cell配置：对于guest中的设备配置，根据guest cell dts的配置，可以轻松的配置其地址位置。
```

## 在mac上的读写sd卡操作

```bash
diskutil list
diskutil umountDisk /dev/diskx
sudo dd if=/linux.dmg of=/dev/rdiskx bs=1m
diskutil eject /dev/diskx
```

## 修改Jailhouse源码重新编译

1. 克隆仓库
2. 修改文件
3. 生成patch: git diff > ../0001-xxx-xx.patch
4. 添加到SRC_URI
5. 重新运行