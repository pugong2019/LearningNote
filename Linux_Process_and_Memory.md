# Linux 进程与内存学习
## 1 CPU hardware
1. **physical CPU, physical core, proecessor(logical core) and hyper-threading**  
   **physical CPU** is actual cpu number, in most PC, there is only 1 physical CPU
   **physical core** is actual cpu core number: NUC13(5-1340P Processor) core number 12: 4 Performence-cores and 8 Efficient-cores, total is 12  
   **logical core** processor(thread number: could get by top cmd: top and input 1): Intel® Hyper-Threading Technology is only available on Performance-cores. so total cpu threads is 4 P-Core * 2 + 8 E-core *1 = 16
   ![alt text](image.png)  
2. CPU info  
   ```shell
   cd /sys/devices/system/cpu/
   ```
3. CPU TLB  
   32 bit: 2 level page table  
   64 bit: 4 level page table  
   **PGD**: Manage 39~47 bit  
   **PUD**: Manage 30~38 bit  
   **PMD**: Manage 21~29 bit  
   **PTE**: Manage 12~20 bit  
   TODO: Need to draw a table to show this

## 2 Process And Thread

### Kernel thread:
1. 内核任务应该成为线程，因为内核任务没有独立的地址空间， 进程和线程最重要的区别就是有独立的地址空间，Linux上，线程也被称为轻量级进程

### task_struct
### task nameproxy:  
* namespace 是 Linux 内核用来隔离内核资源的方式。通过 namespace 可以让一些进程只能看到与自己相关的一部分资源，而另外一些进程也只能看到与它们自己相关的资源，这两拨进程根本就感觉不到对方的存在。具体的实现方式是把一个或多个进程的相关资源指定在同一个 namespace 中。  
* Linux namespaces 是对全局系统资源的一种封装隔离，使得处于不同 namespace 的进程拥有独立的全局系统资源，改变一个 namespace 中的系统资源只会影响当前 namespace 里的进程，对其他 namespace 中的进程没有影响。  
* Linux 内核实现 namespace 的一个主要目的就是实现轻量级虚拟化(容器)服务。在同一个 namespace 下的进程可以感知彼此的变化，而对外界的进程一无所知。这样就可以让容器中的进程产生错觉，认为自己置身于一个独立的系统中，从而达到隔离的目的。也就是说 linux 内核提供的 namespace 技术为 docker 等容器技术的出现和发展提供了基础条件。 

### PID  
1. PID keep use bitmap： 内存优化， 以时间换空间， 查询效率低， 但是由于内存紧凑，缓存命中效率高
2. PID use IDR（基数树）
3. Differnece between Process and Thread  

### Process
1. Create: fork(user) --> fork(kernel) --> kernel_clone --> copy_process  
2. Parameter: kernel_clone_args: SIGCHILD

### Thread
1. Create: pthread_create(user) --> do_clone(user) --> clone(kernel) --> kernel_clone --> copy_process  
2. Parameter: kernel_clone_agrs: CLONE_VM; CLONE_FS; CLONE_FILES ...， CLONE了所有参数，和创建它的进程共享虚拟内存空间， 文件系统信息，打开文件列表（fd）

## 进程加载启动原理
### ELF
1. ELF 由四部分祖成： ELF Header; Program Header Type; Section; Section Header Table;  
   * **ELF Header**： readelf --file-header hello_world (bin file)  
   * **Program Header**: readelf --program-headers hello-world
   * **Section Header**: Table: readelf --section-headers hello-world

2. shell启动用户进程
   * fork + execve (vfork + execve)

2. ELF文件加载  
   * 内核在内存的使用上和用户进程的虚拟地址空间是不一样的，kmalloc系列的函数都是直接在伙伴系统所管理的物理内存中分配的，不需要触发缺页中断 

## 3 Memory
### 面试问题： https://www.jishuchi.com/read/linux-interview/2830
### Virtual Memory Structure
![alt text](image.png) 

### 初期memblock 分配器
1. 在Linxu操作系统启动的时候，操作系统通过e820读取到内存的布局，并打印到日志中. 内核在启动时,在伙伴系统创建之前，所有的内存都是通过memblock内存分配器来分配的，只是Linux启动时运行的一个临时的内存分配器，管理内存的粒度很大，并不适用于内核运行时小块内存的分配
2. free -m 命令看到的内存小于实际的物理内存(dimdecode)， 原因是：Linux不会把全部物理内存给我们使用，linux为了维护自身的运行，需要消耗一些内存来存储一些数据结构
3. memblock管理的内存两种类型： Reserved 和 Usable， 当交接给伙伴系统的时候，是分开交接的

### 伙伴系统
1. NUMA
   * 操作系统启动时不知道NUMA的信息，通过ACPI接口读取固件中的SRAT(System Resource Affinity Table)表(表中存储的是CPU核和内存的关系图)， 将NUMA信息保存到numa_meminfo数组
   * 查看系统Node信息
   ```shell
   $ numactl --hardware #查看每个node的情况, 台式机上只有一个node, 多个node常见于服务器
      available: 1 nodes (0)
      node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11
      node 0 size: 64162 MB
      node 0 free: 54991 MB
      node distances:
      node   0 
      0:  10 
   ```
   * 内核不是只有一个伙伴系统，在每个zone下面都有一个伙伴系统，一个Node里面通常含有多个Zone: ZONE_DMA; ZONE_DMA32; ZONE_NORMAL
   ```shell
   $ cat /proc/zoneinfo | grep zone
      Node 0, zone      DMA
            nr_zone_inactive_anon 0
            nr_zone_active_anon 0
            nr_zone_inactive_file 0
            nr_zone_active_file 0
            nr_zone_unevictable 0
            nr_zone_write_pending 0
      Node 0, zone    DMA32
            nr_zone_inactive_anon 41
            nr_zone_active_anon 108
            nr_zone_inactive_file 0
            nr_zone_active_file 0
            nr_zone_unevictable 0
            nr_zone_write_pending 0
      Node 0, zone   Normal
            nr_zone_inactive_anon 32238
            nr_zone_active_anon 263364
            nr_zone_inactive_file 1550759
            nr_zone_active_file 207932
            nr_zone_unevictable 22299
            nr_zone_write_pending 9
      Node 0, zone  Movable
      Node 0, zone   Device
   ```
   * NUMA对性能的影响还是比较大的，比如多个CPU的情况下，当前CPU访问与其他CPU更近的内存，就会比较耗时，所以，在不少的公司对运行的服务进行了NUMA绑定

2. 操作系统在运行时经常需要管理小粒度的颗粒，比如4KB, 这时使用伙伴系统来进行物理页的管理
3. 伙伴系统总共管理11个不同大小的内存:4KB, 8KB, 16KB, 32KB, 64Kb, 128KB, 256KB, 512KB, 1024KB, 2048KB, 4096KB
   ```shell
   $ sudo cat /proc/pagetypeinfo  #查看当前系统里伙伴系统各个尺寸的可用连续内存块数量
      Free pages count per migrate type at order       0      1      2      3      4      5      6      7      8      9     10 
      Node    0, zone      DMA, type    Unmovable      0      0      0      0      0      0      0      0      1      0      0 
      Node    0, zone      DMA, type      Movable      0      0      0      0      0      0      0      0      0      1      2 
      Node    0, zone      DMA, type  Reclaimable      0      0      0      0      0      0      0      0      0      0      0 
      Node    0, zone      DMA, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0 
      Node    0, zone      DMA, type      Isolate      0      0      0      0      0      0      0      0      0      0      0 
      Node    0, zone    DMA32, type    Unmovable      0      1      1      1      1      1      1      1      1      1      5 
      Node    0, zone    DMA32, type      Movable      9     11      9      7      9      9      9      6      7      8    189 
      Node    0, zone    DMA32, type  Reclaimable      0      0      0      0      0      0      0      0      0      0      3 
      Node    0, zone    DMA32, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0 
      Node    0, zone    DMA32, type      Isolate      0      0      0      0      0      0      0      0      0      0      0 
      Node    0, zone   Normal, type    Unmovable    706    988   1064    437    801    759    343    133     39     59    167 
      Node    0, zone   Normal, type      Movable  21906  18699   8583   4546   2737   1564    541    144     45      8  12955 
      Node    0, zone   Normal, type  Reclaimable     80    156     52     16     13     39     44     17     13      5      0 
      Node    0, zone   Normal, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0 
      Node    0, zone   Normal, type      Isolate      0      0      0      0      0      0      0      0      0      0      0 
   ```
*  查看系统剩余物理内存
显示系统剩余内存，以2的幂次方的形式，分成11个块链表，分别对应为1、2、4、8、16、32、64、128、256、512、1024个页块。

```shell
$ cat /proc/buddyinfo
Node 0, zone      DMA      0      0      0      0      0      0      0      0      1      1      2 
Node 0, zone    DMA32      9     12     10      8     10     10     10      7      8      9    197 
Node 0, zone   Normal   3216   6675  13275   7331   3283   2687    819    276    110     80  13123 
```
 * 内核提供分配函数: alloc_pages到上面的多个链表中寻找可用连续页面，当内核或者用户进程幼物理页的时候，就可以调用alloc_pages申请真正的物理内存
 * 基于伙伴系统的内存分配中，有可能需要将大块内存拆分成两个小一级的内存尺寸，也可能将两个**连续的小尺寸内存**再次组成一个更大块的**连续内存**

### Memory Map
1. mmap
   * malloc大于128k的内存使用mmap进行分配
   * mmap是一种内存映射文件的方法，它将一个文件映射到进程的地址空间中，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。当磁盘地址和进程虚拟地址建立关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写数据到磁盘上，即直接完成了对文件的操作而不必在调用read/write等系统调用函数。同样的如果磁盘中内容有修改，也会直接反映到用户空间其数据改变了
   * mmap映射区域大小必须是物理页大小(page_size)的整倍数（在Linux中内存页通常是4k）。因为内存的最小粒度是页，而进程虚拟地址空间和内存的映射也是以页为单位。为了匹配内存的操作，mmap从磁盘到虚拟地址空间的映射也必须是页
   * mmap数据读写的性能提升就在于对数据的读写拷贝次数，mmap只需要一次系统调用（一次拷贝），后续操作不需要系统调用。并且访问的数据不需要在page cache和用户缓冲区之间拷贝。
   * mmap的适用场景是大文件的频繁读写，这样就可以节省很多IO的耗时
   * 在Android中Binder机制中其核心就使用了mmap内存映射。
   * 缺点：
      * 缺点：因为mmap是按照页存储方式进行存储，每页4096字节，如果数据只有100字节，则正页将有大大的浪费。
      * 写回文件的工作由系统负责，但是并不是实时的，是定期写回到磁盘的，中间如果发生内核崩溃、断电等，还是会丢失数据，不过可以通过msync将数据同步回磁盘。
2. brk
   * malloc小于128k的内存使用brk进行分配

### 进程如何使用内存
1. Linux使用的是四级页表来管理虚拟地址空间到物理内存之间的映射的： PGD, PUD，PMD, PTE
2. 匿名映射，文件映射的区别： 
   * **匿名映射**：没有对应的物理文件, 开发者申请的变量对应的就是匿名页映射处理 
   * **文件映射**：有对应的物理文件(在硬盘上)

### 进程和线程的异同
1. 进程和线程的相同之处
2. 进程和线程的不同之处
   * 进程有自己独立的地址空间， 同一个进程的所有线程共用进程的地址空间
   * 进程栈和线程栈的区别
      * 线程的栈需要先申请再使用，也就是提前申请出来的，就是在创建线程的时候，从用户空间（通过glibc）一次性申请一段设定的大小的栈(可以指定大小，默认通常为8M)，后续不能动态扩容，比如函数中局部变量， 就是在该栈中， 进程栈的申请底层是通过通过mmap系统调用从虚拟地址空间申请一块vm_area_struct所表示的一段虚拟内存，这里申请的还只是一段地址范围，真正用到的时候才会分配物理内存
      * 进程的栈是在进程创建的时候，从内核空间申请的，初始大小为4kb，后续可以动态扩大缩小
   * 相同之处：进程栈和线程栈的最大限制都是通过ulimit的stack size指定, 默认值通常为8M
      ```shell
      $ ulimit -a
         real-time non-blocking time  (microseconds, -R) unlimited
         core file size              (blocks, -c) 0
         data seg size               (kbytes, -d) unlimited
         scheduling priority                 (-e) 0
         file size                   (blocks, -f) unlimited
         pending signals                     (-i) 256370
         max locked memory           (kbytes, -l) 8212744
         max memory size             (kbytes, -m) unlimited
         open files                          (-n) 1048576
         pipe size                (512 bytes, -p) 8
         POSIX message queues         (bytes, -q) 819200
         real-time priority                  (-r) 0
         stack size                  (kbytes, -s) 8192
         cpu time                   (seconds, -t) unlimited
         max user processes                  (-u) 256370
         virtual memory              (kbytes, -v) unlimited
         file locks                          (-x) unlimited
      ```
### 进程堆内存管理
1. malloc 实际上是用户空间的函数，不是系统调用，malloc的底层调用的是brk和mmap的系统调用， 是GNU Libc提供的内存分配器的接口(ptmalloc)
   * 内存分配器：ptmalloc, tcmaolloc, jemaolloc
2. ptmalloc
   * chunk = headr + body
      * body部分就是返回给用户的userdata
      * 当开发者调用free释放内存，对应的chunk对象并不会归还给内核，会被glibc组织管理起来， 回将相似大小的空闲内存块chunk都串起来，等到用户下次申请的时候，进行快速分配。
   * ptmalloc根据管理内存的大小，总共有fastbins， smallbins，largebins， unsortedbins四类

## 进程调度器

### 经典调度算法： 先来先服务， 短作业优先，时间片轮转调度，优先级调度等
### 负载均衡模块

### 内核调度的实现
1. 为每个cpu核定义一个运行队列
2. 在每一个运行队列中都定义了红黑树来组织待运行任务
3. 基于定时器的调度节拍判断是否需要切换下一个任务
4. 需要考虑多个运行队列中的负载均衡问题


## 性能统计原理
   