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



### swap_memory()





## Disks

### disk_partitions(all=False)



### disk_usage(path)



### disk_io_counters(perdisk=False, nowrap=True)



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

