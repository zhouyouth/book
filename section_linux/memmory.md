# 内存监控

### 虚拟内存理解

虚拟内存是操作系统对进程地址空间进行管理而设计的逻辑内存空间概念，我们程序中的指针其实都是这个虚拟内存空间中的地址，因为程序还没有开始运行，所以根本就没有物理内存。

虚拟内存是如何映射到真是的物理内存上的，答案是页映射表，操作系统为每个进程维护一个页映射表，通过页映射表把虚拟内存地址映射到真是的物理内存地址
当程序开始运行的时候，内核首先去查找内存缓存和物理内存，如果数据已经在内存中，则忽略，如果数据不在内存里就引起一个缺页中断（Page Fault) 然后从硬盘读取缺页，并把缺页缓存到物理内存里
![file-list](http://www.bo56.com/wp-content/uploads/2013/08/t1.png)

### SWAP

注意SWAP和虚拟内存的区别， 当系统没有足够物理内存来应付所有请求的时候就会用到 swap 设备，swap 设备可以是一个文件，也可以是一个磁盘分区。不过要小心的是，使用 swap 的代价非常大。如果系统没有物理内存可用，就会频繁 swapping，如果 swap 设备和程序正要访问的数据在同一个文件系统上，那会碰到严重的 IO 问题，最终导致整个系统迟缓，甚至崩溃。swap 设备和内存之间的 swapping 状况是判断 Linux 系统性能的重要参考



总结一下就是，虚拟内存是一个假象的内存空间，在程序运行过程中虚拟内存空间中需要被访问的部分会被映射到物理内存空间中。虚拟内存空间大只能表示程序运行过程中可访问的空间比较大，不代表物理内存空间占用也大

### top

搞清楚了虚拟内存的概念之后解释VIRT的含义就很简单了。
VIRT表示的是进程虚拟内存空间大小。

对应到图1中的进程A来说就是A1、A2、A3、A4以及灰色部分所有空间的总和。也就是说VIRT包含了在已经映射到物理内存空间的部分和尚未映射到物理内存空间的部分总和

RES的含义是指进程虚拟内存空间中已经映射到物理内存空间的那部分的大小。

对应到图1中的进程A来说就是A1、A2、A3以及A4几个部分空间的总和。所以说，看进程在运行过程中占用了多少内存应该看RES的值而不是VIRT的值。

SHR是share（共享）的缩写，它表示的是进程占用的共享内存大小。
在上图1中我们看到进程A虚拟内存空间中的A4和进程B虚拟内存空间中的B3都映射到了物理内存空间的A4/B3部分。咋一看很奇怪。为什么会出现这样的情况呢？其实我们写的程序会依赖于很多外部的动态库（.so），比如libc.so、libld.so等等。这些动态库在内存中仅仅会保存/映射一份，如果某个进程运行时需要这个动态库，那么动态加载器会将这块内存映射到对应进程的虚拟内存空间中。多个进展之间通过共享内存的方式相互通信也会出现这样的情况。这么一来，就会出现不同进程的虚拟内存空间会映射到相同的物理内存空间。这部分物理内存空间其实是被多个进程所共享的，所以我们将他们称为共享内存，用SHR来表示。某个进程占用的内存除了和别的进程共享的内存之外就是自己的独占内存了。所以要计算进程独占内存的大小只要用RES的值减去SHR值即可。

### vmstat
```
# vmstat 1
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  3 252696   2432    268   7148 3604 2368  3608  2372  288  288  0  0 21 78  1
 0  2 253484   2216    228   7104 5368 2976  5372  3036  930  519  0  0  0 100  0
 0  1 259252   2616    128   6148 19784 18712 19784 18712 3821 1853  0  1  3 95  1
 1  2 260008   2188    144   6824 11824 2584 12664  2584 1347 1174 14  0  0 86  0
 2  1 262140   2964    128   5852 24912 17304 24952 17304 4737 2341 86 10  0  0  4

swpd，已使用的 SWAP 空间大小，KB 为单位；如果一个程序使用了太多了swap空间，就需要特别注意了
free，可用的物理内存大小，KB 为单位；
buff，物理内存用来缓存读写操作的 buffer 大小，KB 为单位；
cache，物理内存用来缓存进程地址空间的 cache 大小，KB 为单位；
si，数据从 SWAP 读取到 RAM（swap in）的大小，KB 为单位；
so，数据从 RAM 写到 SWAP（swap out）的大小，KB 为单位；
bi，磁盘块从文件系统或 SWAP 读取到 RAM（blocks in）的大小，block 为单位；
bo，磁盘块从 RAM 写到文件系统或 SWAP（blocks out）的大小，block 为单位；
```

物理可用内存 free 基本没什么显著变化，swapd 逐步增加，说明最小可用的内存始终保持在 256MB X 10％ = 2.56MB 左右，当脏页达到10％的时候（vm.dirty_background_ratio ＝ 10）就开始大量使用 swap


buff 稳步减少说明系统知道内存不够了，正在从 buff 那里借用部分内存；


### free
free命令可以查看系统内存情况
```
free -k [-m, -g, -t]

free -s 3: 每隔5s更新一下内存
```


### 硬盘管理

df:
```
df -h

Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G  4.0K  3.9G   1% /dev
tmpfs           798M  791M  7.5M 100% /run
/dev/xvda5       71G   45G   23G  67% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
none            5.0M     0  5.0M   0% /run/lock
none            3.9G     0  3.9G   0% /run/shm
none            100M     0  100M   0% /run/user
/dev/xvda1      236M  152M   68M  69% /boot
/dev/xvdb1      591G  564M  560G   1% /srv/www

```

du的英文原义为“disk usage”，含义为显示磁盘空间的使用情况，统计目录（或文件）所占磁盘空间的大小。
该命令的功能是逐级进入指定目录的每一个子目录并显示该目录占用文件系统数据块（1024字节）的情况
1块 = 1kB

[参考文章](http://www.vpsee.com/2009/11/linux-system-performance-monitoring-memory/)


