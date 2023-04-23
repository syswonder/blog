# 使用gdb调试jailhouse步骤
## 启动qemu
在qemu启动参数中加入-s -S，qemu会在1234端口打开一个gdbserver，同时在启动时等待gdb进行连接，为了调试方便，最好只启用一个cpu。
## 启动gdb
由于是aarch64架构，所以用gdb-multiarch，启动gdb，输入```target remote:1234```连接到qemu的
gdbserver，输入```set arch aarch64```设置架构。输入c让qemu继续启动过程。
## 用gdb跟踪jailhouse.ko
进入linux后，首先需要跟踪到driver，然后才能跳到hypervisor，先安装driver，```insmod path/to/jailhouse.ko```
然后然后需要获取driver的段地址。
```sh 
cd /sys/module/jailhouse/sections
cat .text
cat .data
cat .bss
```
然后将driver的信息传给gdb，此时就可以正常调试driver了，在enter hypervisor处打断点。
```gdb
add-symbol-file path/to/jailhouse.ko [.text地址] -s .data [.data地址] -s .bss [.bss地址]
b enter_hypervisor
```
## 从driver跳到arch_entry
我们的最终目标是使用```add-symbol-file path/to/hypervisor.o [address]```将hypervisor的代码地址传给gdb，这样gdb就可以正常调试hypervisor的代码了。

执行enable jailhouse的命令，之后会运行到断点enter_hypervisor处停止，在gdb内可以查看当前的信息，找到跳转指令，之后跟着的地址就是arch_entry的入口地址，可以对照jailhouse\hypervisor\arch\arm64\entry.S里的代码进行查看。注意当前的地址是el1下linux的虚拟地址，每次运行jailhouse时都可能会变化，所以不能直接用当前的地址作为hypervisor的地址。
## 得到启用el2下，启用mmu之前hypervisor的代码地址
进入arch_entry后，注意到有两个连续的```hvc #0```，第一个```hvc #0```的作用是替换当前的异常向量表，控制```hvc #0```的入口地址到el2_entry，第二个```hvc #0```的作用是进入el2_entry，使hypervisor进入el2。进入el2_entry后，可以发现当前的地址发生了较大的变化，这是因为当前的地址空间被切换为了el2的地址空间，由于启动mmu，其实也就等于物理地址。这里记下当前的el2_entry地址，可以利用el2_entry与hypervisor的固定偏移计算出hypervisor实际被加载的地址，在当前的环境下应该是**0x7fc00800**，不过这个地址也不是最终地址，因为之后马上就会启用mmu。
## 得到启动mmu后在el2下的地址
在el2_entry代码中，会跳转到enable_mmu_el2，从enable_mmu_el2返回后，地址又会发生较大变化，不过按照jailhouse的设置，实际上只是在原有地址上加了一段偏移，观察变化前后某一个地址的偏移，计算出来偏移是0xffff40600000，那根据之前的计算结果，最终hypervisor的地址应该是**0xffffc0200800**，将hypervisor的信息传给gdb，此时就可以正常调试hypervisor的代码了，尝试在init_early处打断点，可以正常定位到该函数，在
```c
printk("\nInitializing Jailhouse hypervisor %s on CPU %d\n",
		   JAILHOUSE_VERSION, cpu_id);
```
处打断点，执行后qemu打印对应信息，确定当前的hypervisor代码地址是正确的，因为就算传给gdb的hypervisor地址是错的，gdb也不会发现，它只会根据符号对应的内存地址设断点。



