* [与系统相关的方法](#与系统相关的方法)
  * [CPU](#CPU)
  * Memory
  * Disks
  * Network
  * Sensors
  * Other system info

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





























