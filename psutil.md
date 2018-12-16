**psutil 文档**

**仅翻译了方法与进程**

* [与系统相关的方法](#与系统相关的方法)
  * [CPU](#CPU)
  * [Memory](#Memory)
  * [Disks](#Disks)
  * [Network](#Network)
  * [Sensors](#Sensors)
  * [Other system info](#Other system info)
* [进程](#进程)
  * [方法](#方法)
  * [异常](#异常)
  * [Process类](#Process类)
  * [Popen类](#Popen类)



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

> Note：在Windows上`diskperf -y`命令可能需要先执行，否则此函数将找不到任何磁盘。

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

将系统范围的网络I/O统计信息作为命名元组返回，包括以下属性：

* `bytes_sent`：发送的字节数
* `bytes_recv`：接收的字节数
* `packets_sent`：发送的包数
* `packets_recv`：接收的包数
* `errin`：接收时的错误总数
* `errout`：发送时的错误总数
* `dropin`：丢弃的传入数据包总数
* `dropout`：丢弃的传出数据包总数（macOS和BSD总是0）

如果`pernic`为`True`，则将系统上安装的每个网络接口的相同信息作为字典返回，其中网络接口名称为键，上面描述的命名元组为值。 在某些系统（如Linux）上，在非常繁忙或长期存在的系统上，内核返回的数字可能会溢出并换行（重新从零开始）。如果`nowrap`为`True`，psutil将在函数调用中检测并调整这些数字，并将“旧值”添加给“新值”，使返回的数字始终增加或保持不变，但永远不会减少。`net_io_counters.cache_clear()`可用于使`nowrap`缓存无效。 在没有网络接口的机器上，如果`pernic`为`True`，则此函数将返回`None`或`{}`。

```
>>> import psutil
>>> psutil.net_io_counters()
snetio(bytes_sent=14508483, bytes_recv=62749361, packets_sent=84311, packets_recv=94888, errin=0, errout=0, dropin=0, dropout=0)
>>>
>>> psutil.net_io_counters(pernic=True)
{'lo': snetio(bytes_sent=547971, bytes_recv=547971, packets_sent=5075, packets_recv=5075, errin=0, errout=0, dropin=0, dropout=0),
'wlan0': snetio(bytes_sent=13921765, bytes_recv=62162574, packets_sent=79097, packets_recv=89648, errin=0, errout=0, dropin=0, dropout=0)}
```

### net_connections(kind='inet')

将系统范围的套接字连接作为命名元组列表返回。 每个命名元组都提供7个属性：

* fd：套接字文件描述符。 如果连接引用当前进程，则可以将其传递给`socket.fromfd()`以获取可用的套接字对象。 在Windows和SunOS上，它始终设置为`-1`。
* family：地址族，AF_INET，AF_INET6或AF_UNIX。
* type：地址类型，SOCK_STREAM或SOCK_DGRAM。
* laddr：本地地址为`(ip，port)`命名元组或AF_UNIX套接字的路径。 
* raddr：远程地址，作为`(ip，port)`命名元组，或者是UNIX套接字的绝对路径。 当远程端点未连接时，您将获得一个空元组（AF_INET *）或`""`（AF_UNIX）。
* status：表示TCP连接的状态。 返回值是`psutil.CONN_ *`常量之一（字符串）。 对于UDP和UNIX套接字，这始终是`psutil.CONN_NONE`。
* pid：打开套接字的进程的PID，如果可检索，否则为None。 在某些平台（例如Linux）上，此字段的可用性会根据进程权限（需要root）而更改。

kind参数是一个字符串，用于过滤符合以下条件的连接：

| **Kind value** | **Connections using**                              |
| -------------- | -------------------------------------------------- |
| `"inet"`       | IPv4 and IPv6                                      |
| `"inet4"`      | IPv4                                               |
| `"inet6"`      | IPv6                                               |
| `"tcp"`        | TCP                                                |
| `"tcp4"`       | TCP over IPv4                                      |
| `"tcp6"`       | TCP over IPv6                                      |
| `"udp"`        | UDP                                                |
| `"udp4"`       | UDP over IPv4                                      |
| `"udp6"`       | UDP over IPv6                                      |
| `"unix"`       | UNIX socket (both UDP and TCP protocols)           |
| `"all"`        | the sum of all the possible families and protocols |

在macOS和AIX上，此功能需要root权限。 要获得每个进程的连接，请使用`Process.connections()`。例：

```
>>> import psutil
>>> psutil.net_connections()
[pconn(fd=115, family=<AddressFamily.AF_INET: 2>, type=<SocketType.SOCK_STREAM: 1>, laddr=addr(ip='10.0.0.1', port=48776), raddr=addr(ip='93.186.135.91', port=80), status='ESTABLISHED', pid=1254),
 pconn(fd=117, family=<AddressFamily.AF_INET: 2>, type=<SocketType.SOCK_STREAM: 1>, laddr=addr(ip='10.0.0.1', port=43761), raddr=addr(ip='72.14.234.100', port=80), status='CLOSING', pid=2987),
 pconn(fd=-1, family=<AddressFamily.AF_INET: 2>, type=<SocketType.SOCK_STREAM: 1>, laddr=addr(ip='10.0.0.1', port=60759), raddr=addr(ip='72.14.234.104', port=80), status='ESTABLISHED', pid=None),
 pconn(fd=-1, family=<AddressFamily.AF_INET: 2>, type=<SocketType.SOCK_STREAM: 1>, laddr=addr(ip='10.0.0.1', port=51314), raddr=addr(ip='72.14.234.83', port=443), status='SYN_SENT', pid=None)
 ...]
```

> 注意：
>
> * （macOS和AIX）除非以root用户身份运行，否则始终会引发`psutil.AccessDenied`。 这是操作系统的限制，而`lsof`也是如此。
>
> * （Solaris）不支持UNIX套接字。
> * （Linux，FreeBSD）UNIX套接字的“raddr”字段始终设置为“”。 这是操作系统的限制。
> * （OpenBSD）UNIX套接字的“laddr”和“raddr”字段始终设置为“”。 这是操作系统的限制。

### net_if_addrs()

将与系统上安装的每个NIC（网络接口卡）关联的地址作为字典返回，该字典的键是NIC名称，值是分配给NIC的每个地址的命名元组列表。 每个命名元组包括5个字段：

* family：地址族，AF_INET，AF_INET6或`psutil.AF_LINK`，指MAC地址。

* address：主NIC地址（始终设置）。

* netmask：网络掩码地址（可以是None）。

* 广播：广播地址（可以是None）。

* ptp：代表“点对点”; 它是点对点接口（通常是VPN）上的目标地址。 广播和ptp是互斥的。 可能是None。

例：

```
>>> import psutil
>>> psutil.net_if_addrs()
{'lo': [snicaddr(family=<AddressFamily.AF_INET: 2>, address='127.0.0.1', netmask='255.0.0.0', broadcast='127.0.0.1', ptp=None),
        snicaddr(family=<AddressFamily.AF_INET6: 10>, address='::1', netmask='ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff', broadcast=None, ptp=None),
        snicaddr(family=<AddressFamily.AF_LINK: 17>, address='00:00:00:00:00:00', netmask=None, broadcast='00:00:00:00:00:00', ptp=None)],
 'wlan0': [snicaddr(family=<AddressFamily.AF_INET: 2>, address='192.168.1.3', netmask='255.255.255.0', broadcast='192.168.1.255', ptp=None),
           snicaddr(family=<AddressFamily.AF_INET6: 10>, address='fe80::c685:8ff:fe45:641%wlan0', netmask='ffff:ffff:ffff:ffff::', broadcast=None, ptp=None),
           snicaddr(family=<AddressFamily.AF_LINK: 17>, address='c4:85:08:45:06:41', netmask=None, broadcast='ff:ff:ff:ff:ff:ff', ptp=None)]}
```

>注意：
>
>* 如果你对其他族感兴趣（例如AF_BLUETOOTH），你可以使用更强大的netifaces扩展。
>* 您可以拥有与每个接口关联的同一系列的多个地址（这就是dict值为列表的原因）。
>* Windows不支持broadcast和ptp，并且始终为None。

### net_if_stats()

将有关每个NIC（网络接口卡）的信息作为字典返回，该字典的键是NIC名称，值是带有以下字段的命名元组：

* isup：指示NIC是否已启动并运行的布尔值。

* duplex：双工通信类型; 它可以是NIC_DUPLEX_FULL，NIC_DUPLEX_HALF或NIC_DUPLEX_UNKNOWN。
* speed：以兆字节（MB）表示的NIC速度，如果无法确定（例如'localhost'），它将被设置为0。
* mtu：NIC的最大传输单位，以字节为单位。

例：

```
>>> import psutil
>>> psutil.net_if_stats()
{'eth0': snicstats(isup=True, duplex=<NicDuplex.NIC_DUPLEX_FULL: 2>, speed=100, mtu=1500),
 'lo': snicstats(isup=True, duplex=<NicDuplex.NIC_DUPLEX_UNKNOWN: 0>, speed=0, mtu=65536)}
```



## Sensors

### sensors_temperatures(fahrenheit=False)

返回硬件温度。 每个条目都是一个名为元组，代表某个硬件温度传感器（它可能是CPU，硬盘或其他东西，具体取决于操作系统及其配置）。 除非将华氏温度设置为真，否则所有温度均以摄氏度表示。 如果OS不支持传感器，则返回空的dict。 例：

```
>>> import psutil
>>> psutil.sensors_temperatures()
{'acpitz': [shwtemp(label='', current=47.0, high=103.0, critical=103.0)],
 'asus': [shwtemp(label='', current=47.0, high=None, critical=None)],
 'coretemp': [shwtemp(label='Physical id 0', current=52.0, high=100.0, critical=100.0),
              shwtemp(label='Core 0', current=45.0, high=100.0, critical=100.0),
              shwtemp(label='Core 1', current=52.0, high=100.0, critical=100.0),
              shwtemp(label='Core 2', current=45.0, high=100.0, critical=100.0),
              shwtemp(label='Core 3', current=47.0, high=100.0, critical=100.0)]}
```

可用系统：Linux, FreeBSD

###  sensors_fans()

返回硬件风扇速度。 每个条目都是一个名为元组，代表某个硬件传感器风扇。 风扇速度以RPM（每分钟轮数）表示。 如果OS不支持传感器，则返回空的dict。 例：

```
>>> import psutil
>>> psutil.sensors_fans()
{'asus': [sfan(label='cpu_fan', current=3200)]}
```

可用系统：Linux, macOS

### sensors_battery()

将电池状态信息作为命名元组返回，包括以下值。 如果未安装电池或无法确定指标，则返回None。

* percent：电池剩余百分比。

* secsleft：电池电量耗尽前剩下的秒数的粗略近似值。 如果连接了交流电源线，则将其设置为`psutil.POWER_TIME_UNLIMITED`。 如果无法确定，则将其设置为`psutil.POWER_TIME_UNKNOWN`。

* power_plugged：如果连接了交流电源线，则为True，否则为False。如果无法确定，则为None。

 例：

````
>>> import psutil
>>>
>>> def secs2hours(secs):
...     mm, ss = divmod(secs, 60)
...     hh, mm = divmod(mm, 60)
...     return "%d:%02d:%02d" % (hh, mm, ss)
...
>>> battery = psutil.sensors_battery()
>>> battery
sbattery(percent=93, secsleft=16628, power_plugged=False)
>>> print("charge = %s%%, time left = %s" % (battery.percent, secs2hours(battery.secsleft)))
charge = 93%, time left = 4:37:08
````

可用系统: Linux, Windows, FreeBSD



## Other system info

### boot_time()

返回自纪元（1970.1.1）以来以秒表示的系统启动时间。例：

```
>>> import psutil, datetime
>>> psutil.boot_time()
1389563460.0
>>> datetime.datetime.fromtimestamp(psutil.boot_time()).strftime("%Y-%m-%d %H:%M:%S")
'2014-01-12 22:51:00'
```

> 注：在Windows上，如果在不同进程中使用该函数，则该函数可能会返回1秒的时间

### users()

将当前在系统上连接的用户作为命名元组列表返回，包括以下字段：

* user：用户名。
* terminal：返回与用户关联的tty或伪tty（如果有），否则为None。
* host：与条目关联的主机名（如果有）。
* start：创建时间的浮点数，自纪元起，以秒为单位。
* pid：登录过程的PID（如sshd，tmux，gdm-session-worker，...）。在Windows和OpenBSD上，它始终设置为None。

例：

```
>>> import psutil
>>> psutil.users()
[suser(name='giampaolo', terminal='pts/2', host='localhost', started=1340737536.0, pid=1352),
 suser(name='giampaolo', terminal='pts/3', host='localhost', started=1340737792.0, pid=1788)]
```



# 进程

## 方法

### pids()

返回当前运行的PID列表。 要迭代所有进程并避免竞争条件，应首选`process_iter()`。

```
>>> import psutil
>>> psutil.pids()
[1, 2, 3, 5, 7, 8, 9, 10, 11, 12, 13, 14, 15, 17, 18, 19, ..., 32498]
```

### process_iter(attr=None, ad_value=None)

返回一个迭代器，为本地计算机上的所有正在运行的进程生成一个`Process`类实例。 每个实例只创建一次，然后缓存到内部表中，每次生成一个元素时都会更新。 检查缓存的进程实例的身份，以便在另一个进程重用PID时保证安全，在这种情况下，缓存的实例会更新。 这比`psutil.pids()`更适合迭代进程。返回进程的排序顺序基于其PID。  `attrs`和`ad_value`与`Process.as_dict()`中的含义相同。 如果指定了`attrs`，则在内部调用`Process.as_dict()`，并将生成的dict存储为`info`属性，该属性附加到返回的`Process`实例。如果`attrs`是一个空列表，它将检索所有进程信息（慢）。 用法示例：

```
>>> import psutil
>>> for proc in psutil.process_iter():
...     try:
...         pinfo = proc.as_dict(attrs=['pid', 'name', 'username'])
...     except psutil.NoSuchProcess:
...         pass
...     else:
...         print(pinfo)
...
{'name': 'systemd', 'pid': 1, 'username': 'root'}
{'name': 'kthreadd', 'pid': 2, 'username': 'root'}
{'name': 'ksoftirqd/0', 'pid': 3, 'username': 'root'}
...
```

使用`attrs`参数的更紧凑版本：

```
>>> import psutil
>>> for proc in psutil.process_iter(attrs=['pid', 'name', 'username']):
...     print(proc.info)
...
{'name': 'systemd', 'pid': 1, 'username': 'root'}
{'name': 'kthreadd', 'pid': 2, 'username': 'root'}
{'name': 'ksoftirqd/0', 'pid': 3, 'username': 'root'}
...
```

用于创建`{pid：info，...}`数据结构的dict理解示例：

```
>>> import psutil
>>> procs = {p.pid: p.info for p in psutil.process_iter(attrs=['name', 'username'])}
>>> procs
{1: {'name': 'systemd', 'username': 'root'},
 2: {'name': 'kthreadd', 'username': 'root'},
 3: {'name': 'ksoftirqd/0', 'username': 'root'},
 ...}
```

显示如何按名称过滤进程的示例：

```
>>> import psutil
>>> [p.info for p in psutil.process_iter(attrs=['pid', 'name']) if 'python' in p.info['name']]
[{'name': 'python3', 'pid': 21947},
 {'name': 'python', 'pid': 23835}]
```

### pid_exists(pid)

检查当前进程列表中是否存在给定的PID。 这比`pid in psutil.pids()`更快，应该是首选。

### wait_procs(procs, timeout=None, callback=None)

等待`Process`实例列表终止的便捷方法。 返回一个`(gone,alive)`元组，指示哪些进程消失了，哪些进程仍然存活。  已离去的将有一个新的返回码属性，指示进程退出状态（对于不是子进程将为None）。   `callback`是一个函数，当一个正在等待的进程被终止并且一个`Process`实例作为回调参数传递时，它被调用。  一旦所有进程终止或发生超时（秒），此函数将立即返回。 与`Process.wait()`不同，如果发生超时，它不会引发`TimeoutExpired`。  典型的用例可能是：

* 将SIGTERM发送到进程列表

* 给它们一些时间来终止

* 将SIGKILL发送给那些还活着的进程

 终止并等待此进程的所有子进程的示例：

```
import psutil

def on_terminate(proc):
    print("process {} terminated with exit code {}".format(proc, proc.returncode))

procs = psutil.Process().children()
for p in procs:
    p.terminate()
gone, alive = psutil.wait_procs(procs, timeout=3, callback=on_terminate)
for p in alive:
    p.kill()
```



## 异常

### *class* psutil.Error

基本异常类。 所有其他异常都继承自此异常。

### *class* psutil.NoSuchProcess(*pid*, *name=None*, *msg=None*)

当在当前进程列表中找不到具有给定pid的进程或进程不再存在时，由`Process`类方法引发。  name是进程在消失之前具有的名称，仅在先前调用`Process.name()`时才设置。

### *class* psutil.ZombieProcess(*pid*, *name=None*, *ppid=None*, *msg=None*)

在UNIX上查询僵尸进程时，可能会由`Process`类方法引发这种情况（Windows没有僵尸进程）。 根据所调用的方法，OS可能能够成功检索过程信息。 注意：这是`NoSuchProcess`的子类，因此如果您对检索僵尸进程不感兴趣（例如，在使用`process_iter()`时），您可以忽略此异常并捕获`NoSuchProcess`。

### *class* psutil.AccessDenied(*pid=None*, *name=None*, *msg=None*)

当执行操作的权限被拒绝时，由`Process`类方法引发。  name是进程的名称（可以是None）。

### *class* psutil.TimeoutExpired(*seconds*, *pid=None*, *name=None*, *msg=None*)

如果超时到期且进程仍处于活动状态，则由`Process.wait()`引发。



## Process类

### *class* psutil.Process(*pid=None*)

表示具有给定pid的OS进程。 如果省略pid，则使用当前进程pid（`os.getpid()`）。 
如果pid不存在，则引发`NoSuchProcess`。 在Linux上，pid也可以引用线程ID（threads()方法返回的id字段）。访问此类的方法时，始终准备捕获`NoSuchProcess`，`ZombieProcess`和`AccessDenied`异常。`hash()`嵌入可以用于此类的实例，以便随着时间的推移单一地识别一个进程（哈希是通过混合进程PID和创建时间来确定的）。 因此它也可以与`set()`一起使用。

> 注：为了有效地同时获取有关进程的多个信息，请确保使用`as_dict()`或`oneshot()`上下文管理器。

> 这个类绑定到进程的方式是通过其PID唯一的。 这意味着如果进程终止并且OS重用其PID，则可能最终与另一个进程交互。 对进程标识进行抢先检查（通过PID 
> +创建时间）的唯一例外是以下方法：`nice()`（set），`ionice()`（set），`cpu_affinity()`（set），`rlimit()`（set），`children()`，`parent()`，`suspend()`，`resume()`，`send_signal()`，`terminate()`，`kill()`。 要防止所有其他方法出现此问题，可以在查询进程或`process_iter()`之前使用`is_running()`，以防迭代所有进程。必须注意的是，除非您处理非常“旧”（非活动）的流程实例，否则这几乎不会成为问题。



## Popen类

### *class* psutil.Popen(**args*, **\*kwargs*)

是stdlib中`subprocess.Popen`的更方便的接口。 它启动一个子进程，你完全像使用`subprocess.Popen`时那样处理它，但是它还提供了`psutil.Process`类的所有方法。 对于两个类共有的方法名称，例如`send_signal()`，`terminate()`和`kill()` `psutil.Process`实现优先。

> 注意：与`subprocess.Popen`不同，此类抢先检查PID是否已在`send_signal()`，`terminate()`和`kill()`上重用，以便您不会意外终止另一个进程

```
>>> import psutil
>>> from subprocess import PIPE
>>>
>>> p = psutil.Popen(["/usr/bin/python", "-c", "print('hello')"], stdout=PIPE)
>>> p.name()
'python'
>>> p.username()
'giampaolo'
>>> p.communicate()
('hello\n', None)
>>> p.wait(timeout=2)
0
```

通过`with`语句支持`psutil.Popen`对象作为上下文管理器：在退出时，将关闭标准文件描述符，并等待进程。 所有Python版本都支持此功能。

```
>>> import psutil, subprocess
>>> with psutil.Popen(["ifconfig"], stdout=subprocess.PIPE) as proc:
>>>     log.write(proc.stdout.read())
```

