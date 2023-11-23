# Redis on ArceOS

时间：2023.7.30

作者：晏巨广

联系方式：coolyanjg@163.com

## 摘要

本文主要讲述了为 ArceOS 兼容 Redis 过程中遇到的问题，以及如何在 ArceOS 上运行 Redis，包括在 Qemu 上和在 x86 真机上两种运行环境，并展示了在 Qemu 上的部分阶段性运行结果。在 x86 上主要完成的是正确性调试工作。

同时，在运行说明中表述了修改 ArceOS 或 Redis 源码的要求，以及运行命令。

## 前期探索记录

### 本地使用 musl-gcc 编译 Redis，熟悉 Redis 编译过程

- 发现：Redis 会编译出来 redis-client, redis-server 等可执行文件，需要的是 redis-server
- 然后看 Makefile 里面**redis-server 是怎么编译出来的，需要用到哪些源文件**
- 发现（MAKEFILE 中）：redis-server 需要的 obj：REDIS_SERVER_OBJ，以及三个 .a：hiredis, lua, hdrhistogram
- 然后理清 redis-server 的所有依赖的 obj 是怎么编译出来的：
  - REDIS_SERVER_OBJ：来自 redis/src 下的所有 .c -> .o
  - 三个 .a：来自 redis/deps 

### 修改 include 路径及相关编译参数，合到 Rukos 框架

- **修改 CFLAGS**（这里面包含了 nostd 之类的参数，建议从 arceos/rukos 的 MAKEFILE 看看已经提供了些什么，以及重要的是指向include 头文件的位置，即 -Ixxxx），修改为 arceos 现有的 CFLAGS（先添加 -nostdinc -I（指向include路径），再一步步添加）
- 这里都是手动敲命令编译的
- 在这里，由于之前的 include 路径下面只有很少的头文件声明，因此在这里会出现很多的 undefined 的东西，然后**一个个加到 ulib/axlibc/include 里面**，并**添加对应的 .c 文件的函数实现**（先全部写成 unimplemented()）这里花了很多时间
- 直到能够成功编译出 redis-server.o，在这中间可能会遇到**很多很多**与具体应用相关的报错，需要针对应用添加额外的 CFLAGS
  - 比如：Redis 中需要指定 USE_JEMALLOC=no，这点是在编译的时候，发现他使用自己实现的 jemalloc，但我们并不需要他这样做
  - 以及编译中会遇到很多需要额外的编译选项，比如 -mcmodel 什么的，通过自己 google 以及问学长解决

### 匹配现有的 build 框架，静态链接出 ELF 文件，能够通过 `make A=xxxx` 来生成 ELF

- 这里需要熟悉 makefile 的结构，**`ELF = app.o + libaxlibc.a + libc.a`**
- 理清楚 rukos 的编译、链接到运行的过程
- 添加 axbuild.mk，需要熟悉一下 axbuild.mk 需要提供什么变量、方法
- arceos/rukos 编译 axlibc/ 中的 rust 文件，会生成一个 libaxlibc.a 在 target/.... 下（参见 scripts/make/build_c.mk）
- 编译的 C 文件会生成在 ulib/axlibc/build_xxx 下，这里面有一个 libc.a
- 链接libaxlibc.a，libc.a 具体的 app.o，注意 LDFLAGS 怎么添加，参见 build_c.mk（这部分在添加了 axbuild.mk 之后应该就不用手动添加 LDFLAGS，但需要检查编译、链接打印出来的信息，看看是否真的传进去了）
- 这里又可能会出现链接的错误，具体错误具体处理
  - **出现像 .ebss 之类的未定义符号？** 在链接的时候需要指定 `-T$(LD_SCRIPT)`，LD_SCRIPT 在 `modules/axhal/linker_$(PLATFORM_NAME).lds` 下。
  - **应用程序有很多的 .o 文件，添加到 axbuild.mk 会很长？** 修改应用的MAKEFILE，使用增量链接（-r）将所有的应用程序相关的.o 打包成一个 .o，例如在 Redis 里面是将所有 .o 打包为了 redis-server.o，参考 `apps/c/redis/redis.patch` 的54-57行。

### 运行！调试！

- 由于前面对所有的函数都是只有声明和空实现，在这里运行起来的时候就能知道哪些函数是空实现，再**一步步补全**
  - 这一步现在绝大部分函数都补全了实现了
- 接下来可能会看到（用 printf 查看具体运行到哪儿了会更方便，GDB 由于函数调用太多，容易跟丢）：
  - 为什么这里是个空指针却不报错？-----> 发现现在 arm 下对空指针会翻译成 0
  - 为什么这个循环了几次之后突然变成了空指针？ -----> 发现有函数写错了（calloc 多清空了几个字节）
  - 为什么这个地方会爆出 page fault？ -----> 发现原有的 realloc 没有判断传来的指针是不是空指针
  - 这个函数在哪儿调用的？ -----> 跟踪源码（很深的函数调用）找到函数，查看参数是不是合法
  - 为什么启动完突然自己退出了？ ----> 有函数写错了/有的函数明明没实现却没有记录 unimplemented/系统检查失败的
  - 其余具体问题具体处理

## 遗留问题

- 传参问题：目前是修改 Redis 源码，将参数写死（argc, argv, port, IP等），并在 server.c/main() 函数一开始进行传入。
  - 该问题已经在添加了解析 dtb/multiboot 后解决，但需要参考现有的 doc 文件添加对应的参数。
- 系统检查：在 arm 下运行时，Redis 会通过 `fork`，`mmap` 等函数检查环境兼容性，这部分通过 `mmap` 返回返回 `MAP_FAILED` 来避免。（已解决）
- 空指针问题：在 aarch64 下，仍然存在页表映射，映射为0，不会出现地址错误；但在 x86 下会出现空指针错误。
- `fwrite bug`：关于出现 `fwrite` 没写完的问题，后续通过对 `fwrite` 进行修改，检查是否需要继续发送写命令来解决。
  - 原因在于底层的外部库：`rust-fatfs` 对于写操作，会在达到 cluster 边界时被截断。
  - 例如：cluster size 为 4096B 的时候，一个 5000B 的写操作只能写入前 4096B，需要再发一次写命令。
- `riscv` 对于 `long double` 数据类型存在问题：已通过编译选项解决。
- 目前 `redis-patch` 已合并到[ArceOS主线](https://github.com/rcore-os/arceos)，以及[Rukos主线](https://github.com/syswonder/rukos)。

## Qemu 环境

### 运行方式

- 运行：`make A=apps/c/redis/ LOG=info BLK=y NET=y ARCH=aarch64 SMP=4 run` (for aarch64)，x86 改 `ARCH=x86_64` 即可。在 make 的过程中会通过 `wget` 拉取 Redis-7.0.12 源码。

### 运行说明

- 如果存在对编译选项、源码的修改，建议执行 `make clean` 后再重新运行。

### 测试

- 使用 redis-cli 和 redis-benchmark 来测试，需要提前安装好相关的工具
  - sudo apt install redis-tools
- 运行：`redis-cli -p 5555` 连接到 redis server
  - 默认端口号 5555
- 运行：`redis-benchmark -p 5555` 进行测试
  - 示例：`redis-benchmark -p 5555 -n 5 -q -c 10`
    - -n：每个测试发出多少个请求
    - -q：显示实时运行速率（每秒处理多少个请求）
    - -c：并发数

### 其他说明

- 请使用 `SMP=4` ，因为 Redis 创建的其中一个后台线程会调用 `pthread_mutex_lock` 之后，调用 `pthread_cond_wait`，如果不开启 SMP 的话，现在的锁实现会阻止切换，从而导致这个后台线程一直占有 CPU，Redis 无法提供服务
- 建议使用 `ramdisk` feature。
- 需要扩大内存（目前不存在） 
  - 修改了 qemu.mk 和 qemu-virt-xxx.toml，将内存提升到 2G
- 扩大了磁盘大小（具体至少需要多大的磁盘没有测）（该问题目前已不存在）
  - 这个问题主要是导致了之前的 `fwrite` 出错，后来修改了 `fwrite` 的实现后不需要特别大的磁盘
    - disk_size = 64MB -> file_size <= 512B
    - disk_size = 8GB -> file_size <= 4096B

- 需要**扩大 fd table 的 fd 数量限制，并且注释掉 flatten_object 中的数量检查**
- 在之前网络不是异步发的时候，会出现一次连接，多次测试，性能下降；合并主线后已经**不出现该情况**了
- 在服务器端的 log 来看，由于 Redis 会在一定时间执行了一定数量的操作之后，会创建快照，然后 fork 一个子进程出来将这个快照持久化，但现在 fork 会返回 -1，导致 Redis 认为这个 fork 一直在失败，从而不断地重试，影响测试性能，久了会导致卡死。
  - 目前已通过传入参数禁止后台 `fork` 子进程去做快照的保存。
- 在 wsl 等环境下需要关闭 kvm，或在 `make` 的时候添加 `ACCEL=n`。
- 目前偶尔出现的一个尚未解决 `x86` 环境下的 bug：
  - 在 x86 环境下开启 `kvm`，会出现像网络包没对齐的现象：一条指令的结果在下一条指令执行完才被返回到 redis-cli 端。且如果 redis-cli 解析到返回的网络包不对，会报错。
  - 但能正常执行 `redis-bench`，并且通过 `redis-cli` 连接后，**等待一定时间**，便不会出现该现象，猜测是建立连接过程通过异步发包造成的（该现象在 net 模块改成异步的之后出现的）。
- 在测试 x86 的时候，由于做了优化，现在已经能够达到上百万的性能（在 benchmark 的时候，请添加 -P 16）

### 测试结果

#### aarch64

##### set and get

- 运行：
  - redis-benchmark -p 5555 -n 100000 -q -t set -c 30
  - redis-benchmark -p 5555 -n 100000 -q -t get -c 30
- 0630 

| Operation | Op number | Concurrency | Round | Result(request per seconds) |
| --------- | --------- | ----------- | ----- | --------------------------- |
| SET       | 100K      | 30          | 1     | 11921.79                    |
|           |           |             | 2     | 11873.66                    |
|           |           |             | 3     | 11499.54                    |
|           |           |             | 4     | 12001.92                    |
|           |           |             | 5     | 11419.44                    |
| GET       | 100K      | 30          | 1     | 13002.21                    |
|           |           |             | 2     | 12642.23                    |
|           |           |             | 3     | 13674.28                    |
|           |           |             | 4     | 12987.01                    |
|           |           |             | 5     | 12853.47                    |

- 0710（处理了 `fwrite` 的问题）

| Operation | Op number | Concurrency | Round | Result(request per seconds) |
| --------- | --------- | ----------- | ----- | --------------------------- |
| SET       | 100K      | 30          | 1     | 12740.48                    |
|           |           |             | 2     | 13150.97                    |
|           |           |             | 3     | 13147.52                    |
|           |           |             | 4     | 12898.23                    |
|           |           |             | 5     | 12918.23                    |
| GET       | 100K      | 30          | 1     | 13253.81                    |
|           |           |             | 2     | 14332.81                    |
|           |           |             | 3     | 14600.67                    |
|           |           |             | 4     | 13974.29                    |
|           |           |             | 5     | 14005.60                    |

- 0714(Update net implementation, maximum: 2.9w(set)，由于性能较高之后，波动峰值比较大)

| Operation | Op number | Concurrency | Round | Result(request per seconds) |
| --------- | --------- | ----------- | ----- | --------------------------- |
| SET       | 100K      | 30          | 1     | 25713.55                    |
|           |           |             | 2     | 25246.15                    |
|           |           |             | 3     | 24968.79                    |
|           |           |             | 4     | 25018.76                    |
|           |           |             | 5     | 25348.54                    |
| GET       | 100K      | 30          | 1     | 27917.37                    |
|           |           |             | 2     | 28360.75                    |
|           |           |             | 3     | 27525.46                    |
|           |           |             | 4     | 27901.79                    |
|           |           |             | 5     | 27495.19                    |

##### 其他测试

- 0704

| Operation   | Op number | Concurrency | Round | Result(request per seconds) |
| ----------- | --------- | ----------- | ----- | --------------------------- |
| PING_INLINE | 100K      | 30          | 1     | 12147.72                    |
| INCR        | 100K      | 30          | 1     | 13097.58                    |
| LPUSH       | 100K      | 30          | 1     | 12955.05                    |
| RPUSH       | 100K      | 30          | 1     | 11339.15                    |
| LPOP        | 100K      | 30          | 1     | 12611.93                    |
| RPOP        | 100K      | 30          | 1     | 13106.16                    |
| SADD        | 100K      | 30          | 1     | 12773.02                    |
| HSET        | 100K      | 30          | 1     | 11531.37                    |
| SPOP        | 100K      | 30          | 1     | 12918.23                    |
| ZADD        | 100K      | 30          | 1     | 10462.44                    |
| ZPOPMIN     | 100K      | 30          | 1     | 12817.23                    |
| LRANGE_100  | 100K      | 30          | 1     | 6462.45                     |
| LRANGE_300  | 100K      | 30          | 1     | 3318.84                     |
| LRANGE_500  | 100K      | 30          | 1     | 2522.13                     |
| LRANGE_600  | 100K      | 30          | 1     | 1877.30                     |
| MSET        | 100K      | 30          | 1     | 8929.37                     |

- 0714

```
PING_INLINE: 28768.70 requests per second
PING_BULK: 31347.96 requests per second
SET: 23185.72 requests per second
GET: 25700.33 requests per second
INCR: 25746.65 requests per second
LPUSH: 20416.50 requests per second
RPUSH: 20868.12 requests per second
LPOP: 20370.75 requests per second
RPOP: 19956.10 requests per second
SADD: 25361.40 requests per second
HSET: 21431.63 requests per second
SPOP: 25438.82 requests per second
ZADD: 23820.87 requests per second
ZPOPMIN: 26954.18 requests per second
LPUSH (needed to benchmark LRANGE): 26385.22 requests per second
LRANGE_100 (first 100 elements): 23912.00 requests per second
LRANGE_300 (first 300 elements): 22665.46 requests per second
LRANGE_500 (first 450 elements): 23369.95 requests per second
LRANGE_600 (first 600 elements): 22256.84 requests per second
MSET (10 keys): 18460.40 requests per second
```

#### x86_64

##### set and get

- 测试命令同上
- 0710

| Operation | Op number | Concurrency | Round | Result(request per seconds) |
| --------- | --------- | ----------- | ----- | --------------------------- |
| SET       | 100K      | 30          | 1     | 30931.02                    |
|           |           |             | 2     | 32258.07                    |
|           |           |             | 3     | 30571.69                    |
|           |           |             | 4     | 33344.45                    |
|           |           |             | 5     | 31655.59                    |
| GET       | 100K      | 30          | 1     | 33523.30                    |
|           |           |             | 2     | 33134.53                    |
|           |           |             | 3     | 30450.67                    |
|           |           |             | 4     | 33178.50                    |
|           |           |             | 5     | 32268.47                    |

- 0714（网络改成异步之后）

| Operation | Op number | Concurrency | Round | Result(request per seconds) |
| --------- | --------- | ----------- | ----- | --------------------------- |
| SET       | 100K      | 30          | 1     | 105263.16                   |
|           |           |             | 2     | 105263.16                   |
|           |           |             | 3     | 103950.10                   |
|           |           |             | 4     | 107758.62                   |
|           |           |             | 5     | 105820.11                   |
| GET       | 100K      | 30          | 1     | 103199.18                   |
|           |           |             | 2     | 104058.27                   |
|           |           |             | 3     | 99502.48                    |
|           |           |             | 4     | 106951.88                   |
|           |           |             | 5     | 105263.16                   |

##### 其他测试

- 0710

| Operation   | Op number | Concurrency | Round | Result(request per seconds) |
| ----------- | --------- | ----------- | ----- | --------------------------- |
| PING_INLINE | 100K      | 30          | 1     | 32992.41                    |
| INCR        | 100K      | 30          | 1     | 32467.53                    |
| LPUSH       | 100K      | 30          | 1     | 29815.14                    |
| RPUSH       | 100K      | 30          | 1     | 30864.20                    |
| LPOP        | 100K      | 30          | 1     | 34094.78                    |
| RPOP        | 100K      | 30          | 1     | 31133.25                    |
| SADD        | 100K      | 30          | 1     | 32948.93                    |
| HSET        | 100K      | 30          | 1     | 31036.62                    |
| SPOP        | 100K      | 30          | 1     | 32916.39                    |
| ZADD        | 100K      | 30          | 1     | 30693.68                    |
| ZPOPMIN     | 100K      | 30          | 1     | 31525.85                    |
| LRANGE_100  | 100K      | 30          | 1     | 22925.26                    |
| LRANGE_300  | 100K      | 30          | 1     | 7404.12                     |
| LRANGE_500  | 100K      | 30          | 1     | 9320.53                     |
| LRANGE_600  | 100K      | 30          | 1     | 7760.96                     |
| MSET        | 100K      | 30          | 1     | 31269.54                    |

- 0714

```
PING_INLINE: 111607.14 requests per second
PING_BULK: 102880.66 requests per second
SET: 80971.66 requests per second
GET: 103519.66 requests per second
INCR: 98425.20 requests per second
LPUSH: 70274.07 requests per second
RPUSH: 108108.11 requests per second
LPOP: 53561.86 requests per second
RPOP: 100200.40 requests per second
SADD: 62150.41 requests per second
HSET: 99009.90 requests per second
SPOP: 104712.05 requests per second
ZADD: 105263.16 requests per second
ZPOPMIN: 110497.24 requests per second
LPUSH (needed to benchmark LRANGE): 74682.60 requests per second
LRANGE_100 (first 100 elements): 62305.30 requests per second
LRANGE_300 (first 300 elements): 8822.23 requests per second
LRANGE_500 (first 450 elements): 22446.69 requests per second
LRANGE_600 (first 600 elements): 17280.11 requests per second
MSET (10 keys): 92081.03 requests per second
```

##### 本机 Redis 测试结果（仅做大致参考）

```
PING_INLINE: 176056.33 requests per second
PING_BULK: 173611.12 requests per second
SET: 175131.36 requests per second
GET: 174825.17 requests per second
INCR: 177304.97 requests per second
LPUSH: 176678.45 requests per second
RPUSH: 176056.33 requests per second
LPOP: 178253.12 requests per second
RPOP: 176678.45 requests per second
SADD: 175746.92 requests per second
HSET: 176991.16 requests per second
SPOP: 176991.16 requests per second
ZADD: 177619.89 requests per second
ZPOPMIN: 176056.33 requests per second
LPUSH (needed to benchmark LRANGE): 178253.12 requests per second
LRANGE_100 (first 100 elements): 113895.21 requests per second
LRANGE_300 (first 300 elements): 50942.43 requests per second
LRANGE_500 (first 450 elements): 35186.49 requests per second
LRANGE_600 (first 600 elements): 28320.59 requests per second
MSET (10 keys): 183150.19 requests per second
```

#### Unikraft

##### AARCH64:

- `redis-benchmark -h 172.44.0.2 -q -n 100000 -c 30`
- Separately: 
  ```
  SET: 13814.06 requests per second
  GET: 15297.54 requests per second
  ```
- The whole benchmark:
  ```
  PING_INLINE: 14369.88 requests per second
  PING_BULK: 13335.11 requests per second
  SET: 13650.01 requests per second
  GET: 12103.61 requests per second
  INCR: 13395.85 requests per second
  LPUSH: 10279.61 requests per second
  RPUSH: 12536.04 requests per second
  LPOP: 9541.07 requests per second
  RPOP: 12540.76 requests per second
  SADD: 11880.72 requests per second
  HSET: 12318.30 requests per second
  SPOP: 12235.41 requests per second
  ZADD: 12130.03 requests per second
  ZPOPMIN: 12223.45 requests per second
  LPUSH (needed to benchmark LRANGE): 11125.95 requests per second
  LRANGE_100 (first 100 elements): 6791.17 requests per second
  LRANGE_300 (first 300 elements): 3772.30 requests per second
  LRANGE_500 (first 450 elements): 2779.71 requests per second
  LRANGE_600 (first 600 elements): 2230.80 requests per second
  MSET (10 keys): 9215.74 requests per second
  ```

##### X86_64（直接按照 ReadMe 运行性能很差，约 3K）

## 真机环境

### 编译、运行

- `make A=apps/c/redis LOG=info NET=y BLK=y PLATFORM=x86_64-pc-oslab APP_FEATURES=use-ramdisk IP=10.2.2.2 GW=10.2.2.1 SMP=4 ARCH=x86_64`
- 将生成的 `redis_x86_64-pc-oslab.elf` 放到 /boot 下，重启系统后进入 grub，选择启动 ArceOS Redis
- server.c 的 2344 行需要把 bindaddr 改成 `0.0.0.0`
- 连接：
  - `redis-cli -p 5555 -h 10.2.2.2`
  - `redis-benchmark -p 5555 -h 10.2.2.2`

### 其他说明

- 注释掉了 spt_init() 函数

## unimplemented （截止 0730）

- 113 个左右，在代码中标注出了 unimplemented()

### dirent.c - 2

- opendir()
- readdir()

### dlfcn.c - 5

- 5个 dlxxx()

### env.c - 2

- 需要设置/初始化 environ 变量
- setenv(), unsetenv()

### fcntl.c - 2

- posix_fadvise()
- sync_file_range()

### file.c - 1

- flock()

### fnmatch.c - 1

- fnmatch()

### ioctl.c - 1

- ioctl()

### math.c - 21

- 21 个

### mmap.c - 5

- mmap()
- munmap()
- mremap()
- mprotect()
- madvise()

### poll.c - 1

- poll()

### pthread.c - 11

- get_tp()
- pthread_testcancel()
- pthread_setname_np()
- init_cancellation()
- pthread_cond_signal()
- pthread_mutex_trylock()
- pthread_cancel()
- pthread_cond_wait()
- pthread_testcancel()
- pthread_kill()
- pthread_cond_broadcast()

### resource.c - 1

- getrusage()

### sched.c - 1

- sched_setaffinity()

### signal.c - 3

- kill()
- raise()
- pthread_sigmask()

### socket.c - 4

- getsockopt()
- setsockopt()
- accept4()
- sendmsg()

### stat.c - 4

- fchmod()
- mkdir()
- chmod()
- fstatat()

### stdio.c - 19

- getchar()
- sscanf()
- feof()
- fseek()
- ftello()
- tmpnam()
- clearerr()
- ferror()
- freopen()
- fscanf()
- ftell()
- getc()
- remove()
- setvbuf()
- tmpfile()
- ungetc()
- getdelim()
- getc_unlocked()
- fdopen()

### stdlib.c - 3

- qsort()
- mkostemp()
- system()

### syslog.c - 2

- syslog()
- openlog()

### time.c - 5

- utimes()
- tzset()
- setitimer()
- ctime_r()
- clock()

### unistd.c - 16

- geteuid()
- getuid()
- access()
- readlink()
- unlink()
- rmdir()
- fsync()
- fdatasync()
- fchown()
- ftruncate()
- execve()
- setsid()
- isatty()
- fork()
- chdir()
- truncate()

### utsname.c - 1

- uname()

### wait.c - 2

- waitpid()
- wait3()
