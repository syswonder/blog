如何使用ZCU102 FPGA实现运行带H扩展的riscv linux
=====================================

本实现基于如下仓库：
`https://github.com/enjoy-digital/litex`



## 1. 初始化仓库

安装基本依赖项
```bash
pip3 install meson ninja
```

初始化litex
```bash
git clone https://github.com/enjoy-digital/litex.git
cd litex
./litex_setup.py --init --install --user --config=full
```

### 2. 为litex-boards打如下patch

```bash
git patch p1-zcu102.patch
git patch p2-zcu102.patch
```

### 3. 为zcu102构建比特流

```bash
source settings64.sh # Vivado环境变量
litex-boards/litex_boards/targets/xilinx_zcu102.py --build --cpu-type rocket --cpu-variant full --cpu-num-cores 4 --cpu-mem-width 2
```

根据系统配置，大约需要30分钟


### 4. 构建dts设备树

```bash
# Step3生成的build文件将生成csr.json，该文件包含了设备的描述信息，我们利用这个文件来生成设备树。
./litex/litex/tools/litex_json2dts_linux.py csr.json > zcu102.dts
dtc -O dtb zcu102.dts -o zcu102.dtb
```

### 5. 构建Linux

```bash
git clone https://github.com/litex-hub/linux.git

make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- litex_rocket_defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- 
ls arch/riscv/boot/Image

```

### 6. 构建opensbi

```bash
make CROSS_COMPILE=riscv64-unknown-linux-gnu- PLATFORM=generic FW_FDT_PATH=zcu102.dtb FW_JUMP_FDT_ADDR=0x81400000
```
同时构建rootfs.cpio.gz
参考：`https://github.com/litex-hub/linux-on-litex-rocket/blob/master/scripts/build_software.sh`


### 7. 准备启动文件


boot.json
```json
{
  "rootfs.cpio.gz": "",
  "Image": "",
  "fw_jump.bin": ""
}

```

SD卡中放入如下文件：

`fw_jump.bin`, `image`, `rootfs.zpio.gz`, `zcu102.dtb`, `boot.json`



### 8. 启动

启动fpga，打开串口
```bash
litex_term /dev/ttyUSB2
```
