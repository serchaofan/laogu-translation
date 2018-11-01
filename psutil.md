* [与系统相关的方法](#与系统相关的方法)
  * [CPU](#CPU)
  * [Memory](#Memory)
  * [Disks](#Disks)
  * [Network](Network)
  * [Sensors](Sensors)
  * [Other system info](#Other system info)

psutil（进程和系统实用程序）是一个跨平台库，用于在Python中检索有关正在运行的进程和系统利用率（CPU，内存，磁盘，网络，传感器）的信息。它主要用于系统监视，分析和限制流程资源以及运行流程的管理。它实现了UNIX命令行工具提供的许多功能，例如：ps，top，lsof，netstat，ifconfig，who，df，kill，free，nice，ionice，iostat，iotop，uptime，pidof，tty，taskset，pmap。



# 与系统相关的方法

## CPU

### cpu_times(percpu=False)

返回系统CPU时间的元组。每个属性表示CPU在给定模式下花费的时间（单位：秒）。属性可用性因平台而异：

通用属性：

* **user**：正常进程在用户模式下执行所花费的时间，在Linux上，这还包括访客时间（guest time）
* **system**：进程在内核模式下执行所花费的时间
* **idle**：空闲时间

平台特定属性：

* **nice** (Unix)：在用户模式下执行的niced（优先级）进程所花费的时间。在Linux上，这还包括`guest_nice`时间

* **iowait** (Linux)：等待I/O完成所花费的时间

* **irq** (Linux, BSD)：处理硬件中断所花费的时间

* **softirq** (Linux)：处理软件中断所花费的时间

* **steal** (Linux)：在虚拟化环境中运行其他操作系统所花费的时间

* **guest** (Linux)：在Linux内核的控制下为客户操作系统运行虚拟CPU所花费的时间

* **guest_nice** (Linux)：运行niced guest虚拟机所花费的时间（Linux内核控制下的客户操作系统的虚拟CPU）

* **interrupt** (Windows)：处理硬件中断所花费的时间（类似于UNIX上的**irq**）

* **dpc** (Windows)：处理延迟过程调用（DPC）所花费的时间，DPC是以比标准中断低的优先级运行的中断。

  > 延迟过程调用Delayed Procedure Call，是Windows的机制，高级别的中断优先交给中断处理程序，低级别的中断交给DPC处理

当参数`percpu`为`True`时，方法返回系统上每个逻辑CPU的元组列表

```
>>> psutil.cpu_times()
scputimes(user=17411.7, nice=77.99, system=3797.02, idle=51266.57, iowait=732.58, irq=0.01, softirq=142.43, steal=0.0, guest=0.0, guest_nice=0.0)
```

### cpu_percent(interval=None, percpu=False)

返回一个浮点数，表示当前系统范围的CPU利用率百分比。

当`interval > 0.0`时，比较间隔（interval）之前和之后经过的系统CPU时间（会阻塞）。

当`interval`为`0.0`或`None`时，比较自上次调用或模块导入后经过的系统CPU时间，并立即返回（非阻塞）。

这意味着第一次调用它将返回一个无意义的`0.0`，应该忽略它。为了准确性，建议调用间隔至少为`0.1`秒。

当`percpu`为`True`时，返回一个浮点列表，表示利用率为每个CPU的百分比。

```
>>> # 阻塞
>>> psutil.cpu_percent(interval=1)
2.0
>>> # 非阻塞，返回的是自上一次调用的百分比
>>> psutil.cpu_percent(interval=None)
2.9
>>> # 阻塞，并返回每个cpu的百分比
>>> psutil.cpu_percent(interval=1, percpu=True)
[2.0, 1.0]
```

### cpu_times_percent(interval=None, percpu=False)

与`cpu_percent()`类似，将`psutil.cpu_times`返回的每个特定CPU时间转换为利用率百分比。 
`interval`和`percpu`参数与`cpu_percent()`中的含义相同。在Linux上，`guest`和`guest_nice`百分比不计入`user`和`user_nice`百分比。

> 第一次使用`interval = 0.0`或`None`调用此函数时，它将返回一个无意义的`0.0`值

### cpu_count(logical=True)

返回系统中的逻辑CPU数量，如果未确定则返回`None`。如果`logical`为`False`，则仅返回物理内核的数量（不包含超线程CPU），如果未确定则返回`None`。在OpenBSD和NetBSD上，`psutil.cpu_count(logical=False)`始终返回`None`。

具有2个物理超线程CPU核心的系统上的示例：

```
>>> import psutil
>>> psutil.cpu_count()
4
>>> psutil.cpu_count(logical=False)
2
```

请注意，此数字**不等于当前进程实际可以使用的CPU数**。如果进程CPU关联性（addinity）已更改，或者正在使用Linux cgroup，或使用处理器组，或者具有64个以上的CPU，则可能会有所不同。可以通过以下方式获得可用CPU的数量：

```
>>> len(psutil.Process().cpu_affinity())
1
```

### cpu_stats()

返回各种CPU统计信息的元组数据。

- **ctx_switches**: 自从系统启动以来的上下文切换次数，包括自愿（voluntary）与非自愿（involuntary）。
- **interrupts**: 自从系统启动以来的中断数量
- **soft_interrupts**: 自从系统启动以来的软件中断数量。在Windows和SunOS上总是为0
- **syscalls**: 自从系统启动以来的系统调用（system calls）次数。在Linux上总为0

```
>>> import psutil
>>> psutil.cpu_stats()
scpustats(ctx_switches=20455687, interrupts=6598984, soft_interrupts=2134212, syscalls=0)
```

### cpu_freq(percpu=False)

将CPU频率作为元组返回，包含当前（current），最小（min）和最大（max）频率，单位为Mhz。在Linux上，当前（current）频率报告的就是实时（real-time）值，在所有其他平台上它代表名义上的“固定”（fixed）值。

如果`percpu`为`True`，且系统支持每个CPU频率检索（per-cpu frequency retrieval）（仅限Linux），则为每个CPU都会返回一个频率列表，否则返回包含单个元素的列表。如果无法确定最小值和最大值，则值会被设为0。

```
>>> import psutil
>>> psutil.cpu_freq()
scpufreq(current=931.42925, min=800.0, max=3500.0)
>>> psutil.cpu_freq(percpu=True)
[scpufreq(current=2394.945, min=800.0, max=3500.0),
 scpufreq(current=2236.812, min=800.0, max=3500.0),
 scpufreq(current=1703.609, min=800.0, max=3500.0),
 scpufreq(current=1754.289, min=800.0, max=3500.0)]
```



## Memory

### virtual_memory()

返回系统内存使用情况的统计信息的元组。主要指标如下：

- **total**: 物理内存总量
- **available**: 不需要调用swap而直接能提供给进程的内存可用大小。这个值是对内存占用量求和得出的，应该用于跨平台监视实际的内存使用情况。

其他指标：

- **used**: 已被占用的内存大小。 **total**和**free**不一定能匹配**used**的值。
- **free**: 未被使用的内存大小。这个值并不反映能够直接使用的内存大小，**available** 才是。  **total**和**used**不一定能匹配**free**的值。
- **active** *(UNIX)*: 目前正在使用或最近使用的内存，存在于RAM中。
- **inactive** *(UNIX)*: 当前未使用的内存大小
- **buffers** *(Linux, BSD)*: 文件系统元数据（metadata）的缓冲大小
- **cached** *(Linux, BSD)*: 各种数据的缓存大小
- **shared** *(Linux, BSD)*: 可以由多个进程同时访问的内存大小
- **slab** *(Linux)*: 内核中数据结构的缓存（in-kernel data structures cache）
- **wired** *(BSD, macOS)*: 永远只待在RAM中而不会移动到磁盘的内存大小（memory that is marked to always stay in RAM. It is never moved to disk）

**used**和**available**之和并不一定等于**total**。在Windows上，**available**和**free**是相等的。

```
>>> mem = psutil.virtual_memory()
>>> mem
svmem(total=10367352832, available=6472179712, percent=37.6, used=8186245120, free=2181107712, active=4748992512, inactive=2758115328, buffers=790724608, cached=3500347392, shared=787554304, slab=199348224)
```

### swap_memory()

返回系统交换内存（swap）统计信息的元组。

- **total**: swap总量，单位字节
- **used**: 已用的swap大小，单位字节
- **free**: 空闲的swap大小，单位字节
- **percent**: swap使用百分比，计算方法：`(total - available) / total * 100`
- **sin**: 累计从磁盘调入swap的大小，单位字节
- **sout**: 累计从swap调出到磁盘的大小，单位字节

在Windows上**sin**和**sout**始终为0。

```
>>> psutil.swap_memory()
sswap(total=2097147904L, used=886620160L, free=1210527744L, percent=42.3, sin=1050411008, sout=1906720768)
```



## Disks

### disk_partitions(all=False)

返回所有已安装的磁盘分区的元组列表，包括设备，挂载点和文件系统类型，类似于UNIX上`df`命令。

如果所有参数都为`False`，它会尝试区分并仅返回物理设备（例如硬盘，cd-rom驱动器，USB密钥）并忽略所有其他设备（例如`/dev/shm`等内存分区）。请注意，这可能在所有系统上都不完全可靠（例如，在BSD上会忽略此参数）。

`fstype`字段是一个字符串，根据平台的不同而不同。在Linux上，它可以是`/proc/filesystems`中的一个值（例如，对于CD-ROM驱动器，ext3硬盘驱动器`iso9660`为`ext3`）。在Windows上，它通过GetDriveType确定，可以是`removable`，`fixed`，`remote`，`cdrom`，`unmounted`或`ramdisk`。在macOS和BSD上，它的值会在`getfsstat`命令中选择。

```
>>> psutil.disk_partitions()
[sdiskpart(device='/dev/sda3', mountpoint='/', fstype='ext4', opts='rw,errors=remount-ro'),
 sdiskpart(device='/dev/sda7', mountpoint='/home', fstype='ext4', opts='rw')]
```

### disk_usage(path)

返回指定路径分区的磁盘使用情况统计信息的元组，包括以字节为单位的总空间，已用空间和可用空间，以及百分比使用情况。如果path不存在，则引发OSError。

```
>>> psutil.disk_usage('/')
sdiskusage(total=21378641920, used=4809781248, free=15482871808, percent=22.5)
```

> UNIX通常为root用户保留总磁盘空间的5％。 UNIX上的**total**字段和**used**字段表示总的总空间和已用空间，而**free**表示用户可用的空间，**percent**表示用户的利用率。这就是为什么百分比值可能看起来比预期的大5％。

### disk_io_counters(perdisk=False, nowrap=True)

返回系统范围的磁盘I/O统计信息的元组。通用字段如下：

- **read_count**: 读取的数量
- **write_count**: 写入的数量
- **read_bytes**: 读取的字节量
- **write_bytes**: 写入的字节量

平台特定属性：

- **read_time**: (不包括*NetBSD* 和 *OpenBSD*) 从磁盘读取数据的时间，单位毫秒
- **write_time**: (不包括 *NetBSD* 和 *OpenBSD*) 写入磁盘的时间，单位毫秒
- **busy_time**: (*Linux*, *FreeBSD*) 实际I/O所花的时间，单位毫秒
- **read_merged_count** (*Linux*): 合并读的数量（merged reads）
- **write_merged_count** (*Linux*): 合并写的数量（merged writes）

如果`perdisk`为`True`，则会返回系统上安装的每个物理磁盘信息的**字典**，其中**分区名称**为键，**信息元组**为值。

在某些系统（如Linux）上，在非常繁忙或已长期存在的系统上，内核返回的数字可能会溢出并清零（wrap）。如果`nowrap`为`True`，psutil将在函数调用中检测并调整这些数字，并将“旧值”添加到“新值”，以便返回的数字将**始终增加或保持不变**，但永远不会减少。`disk_io_counters.cache_clear()`可用于使`nowrap`缓存无效。在Windows上，可能需要首先从`cmd.exe`发出`diskperf -y`命令才能启用IO计数器。在无盘机器上，如果`perdisk`为`True`，则此函数将返回`None`或`{}`。

> Note：在Windows上“diskperf -y”命令可能需要先执行，否则此函数将找不到任何磁盘。

```
>>> psutil.disk_io_counters()
sdiskio(read_count=8141, write_count=2431, read_bytes=290203, write_bytes=537676, read_time=5868, write_time=94922)

>>> psutil.disk_io_counters(perdisk=True)
{'sda1': sdiskio(read_count=920, write_count=1, read_bytes=2933248, write_bytes=512, read_time=6016, write_time=4),
 'sda2': sdiskio(read_count=18707, write_count=8830, read_bytes=6060, write_bytes=3443, read_time=24585, write_time=1572),
 'sdb1': sdiskio(read_count=161, write_count=0, read_bytes=786432, write_bytes=0, read_time=44, write_time=0)}
```



## Network

### net_io_counters(pernic=False, nowrap=True)



### net_connections(kind='inet')



### net_if_addrs()



### net_if_stats()



## Sensors

### sensors_temperatures(fahrenheit=False)



###  sensors_fans()



### sensors_battery()





## Other system info

### boot_time()





### users()

