## Linux 文件系统是怎么工作的

- 概述
  - 磁盘为系统提供了最基本的持久化存储
  - 文件系统则在磁盘的基础上，提供了一个用来管理文件的树状结构
- 索引节点和目录项
  - 为了方便管理，Linux 文件系统为每个文件都分配两个数据结构，索引节点（index node）和目录项（directory entry）。它们主要用来记录文件的元信息和目录结构。
  - 换句话说，索引节点是每个文件的唯一标志，而目录项维护的正是文件系统的树状结构。
    - 索引节点，简称为 inode，用来记录文件的元数据，比如 inode 编号、文件大小、访问权限、修改日期、数据的位置等。索引节点和文件一一对应，它跟文件内容一样，都会被持久化存储到磁盘中。所以记住，索引节点同样占用磁盘空间。
    - 目录项，简称为 dentry，用来记录文件的名字、索引节点指针以及与其他目录项的关联关系。多个关联的目录项，就构成了文件系统的目录结构。不过，不同于索引节点，目录项是由内核维护的一个内存数据结构，所以通常也被叫做目录项缓存。
  - 实际上，磁盘读写的最小单位是扇区，然而扇区只有 512B 大小，如果每次都读写这么小的单位，效率一定很低。所以，文件系统又把连续的扇区组成了逻辑块，然后每次都以逻辑块为最小单元，来管理数据。常见的逻辑块大小为 4KB，也就是由连续的 8 个扇区组成。

- 磁盘在执行文件系统格式化时，会被分成三个存储区域，超级块、索引节点区和数据块区

  - 超级块，存储整个文件系统的状态。
  - 索引节点区，用来存储索引节点。
  - 数据块区，则用来存储文件数据。

- 虚拟文件系统

  - 为了支持各种不同的文件系统，Linux 内核在用户进程和文件系统的中间，又引入了一个抽象层，也就是虚拟文件系统 VFS（Virtual File System）。
    - 用户进程和内核中的其他子系统，只需要跟 VFS 提供的统一接口进行交互就可以了，而不需要再关心底层各种文件系统的实现细节。
  - 这些文件系统，要先挂载到 VFS 目录树中的某个子目录（称为挂载点），然后才能访问其中的文件。
    - 基于磁盘的文件系统为例，在安装系统时，要先挂载一个根目录（/），在根目录下再把其他文件系统（比如其他的磁盘分区、/proc 文件系统、/sys 文件系统、NFS 等）挂载进来。

- 文件系统 I/O

  - 文件读写方式的各种差异，导致 I/O 的分类多种多样。
    - 缓冲 I/O，是指利用标准库缓存来加速文件的访问，而标准库内部再通过系统调度访问文件。
    - 非缓冲 I/O，是指直接通过系统调用来访问文件，不再经过标准库缓存。
    - 无论缓冲 I/O 还是非缓冲 I/O，它们最终还是要经过系统调用来访问文件。我们知道，系统调用后，还会通过页缓存，来减少磁盘的 I/O 操作。
  - 第一种，根据是否利用标准库缓存，可以把文件 I/O 分为缓冲 I/O 与非缓冲 I/O。
  - 第二，根据是否利用操作系统的页缓存，可以把文件 I/O 分为直接 I/O 与非直接 I/O。
    - 直接 I/O，是指跳过操作系统的页缓存，直接跟文件系统交互来访问文件。
    - 非直接 I/O 正好相反，文件读写时，先要经过系统的页缓存，然后再由内核或额外的系统调用，真正写入磁盘
  - 第三，根据应用程序是否阻塞自身运行，可以把文件 I/O 分为阻塞 I/O 和非阻塞 I/O：
  - 第四，根据是否等待响应结果，可以把文件 I/O 分为同步和异步 I/O

- 你也应该可以理解，“Linux 一切皆文件”的深刻含义。无论是普通文件和块设备、还是网络套接字和管道等，它们都通过统一的 VFS 接口来访问。

- 性能观测

  - > 对文件系统来说，最常见的一个问题就是空间不足。用 df 命令，就能查看文件系统的磁盘空间使用情况
    > 	 $ df  -h /dev/sda1

  - 不过有时候，明明你碰到了空间不足的问题，可是用 df 查看磁盘空间后，却发现剩余空间还有很多。这是怎么回事呢？

    - 除了文件数据，索引节点也占用磁盘空间。你可以给 df 命令加上 -i 参数，查看索引节点的使用情况
    - 当你发现索引节点空间不足，但磁盘空间充足时，很可能就是过多小文件导致的。
    - 所以，一般来说，删除这些小文件，或者把它们移动到索引节点充足的其他磁盘中，就可以解决这个问题







## 磁盘I/O

- 通用块层

  - 为了减小不同块设备的差异带来的影响，Linux 通过一个统一的通用块层，来管理各种不同的块设备。
  - 通用块层，其实是处在文件系统和磁盘驱动中间的一个块设备抽象层。它主要有两个功能
    - 向上，为文件系统和应用程序，提供访问块设备的标准接口；向下，把各种异构的磁盘设备抽象为统一的块设备，并提供统一框架来管理这些设备的驱动程序
    - 对 I/O 请求排序的过程，也就是我们熟悉的 I/O 调度

- I/O 栈

  - 我们可以把 Linux 存储系统的 I/O 栈，由上到下分为三个层次，分别是文件系统层、通用块层和设备层

    - 文件系统层，包括虚拟文件系统和其他各种文件系统的具体实现。它为上层的应用程序，提供标准的文件访问接口；对下会通过通用块层，来存储和管理磁盘数据。

    - 通用块层，包括块设备 I/O 队列和 I/O 调度器。它会对文件系统的 I/O 请求进行排队，再通

      过重新排序和请求合并，然后才要发送给下一级的设备层。

    - 设备层，包括存储设备和相应的驱动程序，负责最终物理设备的 I/O 操作。

  - 存储系统的 I/O ，通常是整个系统中最慢的一环。所以， Linux 通过多种缓存机制来优化I/O 效率

    - 比方说，为了优化文件访问的性能，会使用页缓存、索引节点缓存、目录项缓存等多种缓存机制，以减少对下层块设备的直接调用。
    - 同样，为了优化块设备的访问效率，会使用缓冲区，来缓存块设备的数据

- 磁盘性能指标

  - 必须要提到五个常见指标，也就是我们经常用到的，使用率、饱和度、IOPS、吞吐量以及响应时间等。这五个指标，是衡量磁盘性能的基本指标。

    - 使用率，是指磁盘处理 I/O 的时间百分比。过高的使用率（比如超过 80%），通常意味着磁盘 I/O 存在性能瓶颈

      - 这里要注意的是，使用率只考虑有没有 I/O，而不考虑 I/O 的大小。换句话说，当使用率是100% 的时候，磁盘依然有可能接受新的 I/O 请求。

    - 饱和度，是指磁盘处理 I/O 的繁忙程度。过高的饱和度，意味着磁盘存在严重的性

      能瓶颈。当饱和度为 100% 时，磁盘无法接受新的 I/O 请求。

    - IOPS（Input/Output Per Second），是指每秒的 I/O 请求数。

    - 吞吐量，是指每秒的 I/O 请求大小。

    - 响应时间，是指 I/O 请求从发出到收到响应的间隔时间

- 磁盘 I/O 观测

  - iostat 是最常用的磁盘 I/O 性能观测工具，它提供了每个磁盘的使用率、IOPS、吞吐量等各种常见的性能指标，当然，这些指标实际上来自 /proc/diskstats。

  - > 1 # -d -x 表示显示所有磁盘 I/O 的指标
    > 2 $ iostat -d -x 1
    >
    > 
    >
    > %util ，就是我们前面提到的磁盘 I/O 使用率；
    > r/s+ w/s ，就是 IOPS；
    > rkB/s+wkB/s ，就是吞吐量；
    > r_await+w_await ，就是响应时间。

- 进程 I/O 观测

  - 要观察进程的 I/O 情况，你还可以使用 pidstat 和 iotop 这两个工具。

  - > $ pidstat -d 1

  -  iotop。它是一个类似于 top 的工具，你可以按照 I/O大小对进程排序，然后找到 I/O 较大的那些进程。





##  I/O瓶颈

- 我们可以先用 top ，来观察 CPU 和内存的使用情况；然后再用 iostat ，来观察磁盘的 I/O 情况。

- 观察 top 的输出，CPU0 的使用率非常高，它的系统 CPU 使用率（sys%）为6%，而 iowait 超过了 90%。这说明 CPU0 上，可能正在运行 I/O 密集型的进程

- 看内存的使用情况，总内存 8G，剩余内存只有 730 MB，而 Buffer/Cache 占用内存高达 6GB 之多，这说明内存主要被缓存占用。

- 到这一步，你基本可以判断出，CPU 使用率中的 iowait 是一个潜在瓶颈，而内存部分的缓存占比较大，那磁盘 I/O 又是怎么样的情况呢？

- 使用 pidstat 加上 -d 参数，就可以显示每个进程的 I/O 情况

-  lsof

  - 它专门用来查看进程打开文件列表，不过，这里的“文件”不只有普通文件，还包括了目录、块设备、动态库、网络套接字等

  - -p 参数需要指定进程号

    - > -t 表示显示线程，-a 表示显示命令行参数
      >
      > $ pstree -t -a -p [pid]
      >
      > 找到了原因，lsof 的问题就容易解决了。把线程号换成进程号，继续执行 lsof 命令

  - > FD 表示文件描述符号，TYPE 表示文件类型，NAME 表示文件路径
    >
    > python 18940 root 3w REG 8,1 117944320 303 /tmp/logtest.txt
    >
    > 再看最后一行，这说明，这个进程打开了文件 /tmp/logtest.txt，并且它的文件描述符是 3号，而 3 后面的 w ，表示以写的方式打开







## 磁盘I/O延迟很高

- 随便执行一个命令，比如执行 df 命令，查看一下文件系统的使用情况。奇怪的是，这么简单的命令，居然也要等好久才有输出

  - 写文件是由子线程执行的，所以直接strace跟踪进程没有看到write系统调用，可以通过pstree查看进程的线程信息，再用strace跟踪。或者，通过strace -fp pid 跟踪所有线程。
  - 从 strace 中，你可以看到大量的 stat 系统调用，并且大都为 python 的文件，但是，请注意，这里并没有任何 write 系统调用。
  - 我们只好综合 strace、pidstat 和 iostat 这三个结果来分析了。很明显，你应该发现了这里的矛盾：iostat 已经证明磁盘 I/O 有性能瓶颈，而 pidstat 也证明了，这个瓶颈是由12280 号进程导致的，但 strace 跟踪这个进程，却没有找到任何 write 系统调用
  - 文件写，明明应该有相应的 write 系统调用，但用现有工具却找不到痕迹，这时就该想想换工具的问题了。怎样才能知道哪里在写文件呢？

-  filetop

  - 它是 bcc 软件包的一部分，主要跟踪内核中文件的读写情况，并输出线程 ID（TID）、读写大小、读写类型以及文件名称。

  - > 1 # 切换到工具目录
    > 2 $ cd /usr/share/bcc/tools 
    > 34 # -C 选项表示输出新内容时不清空屏幕
    > 5 $ ./filetop -C
    >
    > 
    >
    > 多观察一会儿，你就会发现，每隔一段时间，线程号为 514 的 python 应用就会先写入大量的 txt 文件，再大量地读。

- opensnoop 

  - 它同属于 bcc 软件包，可以动态跟踪内核中的open 系统调用。这样，我们就可以找出这些 txt 文件的路径。
  - 这次，通过 opensnoop 的输出，你可以看到，这些 txt 路径位于 /tmp 目录下。你还能看到，它打开的文件数量，按照数字编号，从 0.txt 依次增大到 999.txt，这可远多于前面用filetop 看到的数量
  - 综合 filetop 和 opensnoop ，我们就可以进一步分析了。我们可以大胆猜测，案例应用在写入 1000 个 txt 文件后，又把这些内容读到内存中进行处理。
  - 结合前面的所有分析，我们基本可以判断，案例应用会动态生成一批文件，用来临时存储数据，用完就会删除它们。但不幸的是，正是这些文件读写，引发了 I/O 的性能瓶颈，导致整个处理过程非常慢

  





## SQL慢查询

- top、iostat、pidstat、strace
- lsof
  - 从输出中可以看到， mysqld 进程确实打开了大量文件，而根据文件描述符（FD）的编号，我们知道，描述符为 38 的是一个路径为/var/lib/mysql/test/products.MYD 的文件。这里注意， 38 后面的 u 表示， mysqld 以读写的方式访问文件。
    - MYD 文件，是 MyISAM 引擎用来存储表数据的文件；
    - 文件名就是数据表的名字；
    - 而这个文件的父目录，也就是数据库的名字。
    - 换句话说，这个文件告诉我们，mysqld 在读取数据库 test 中的 products 表。
- 既然已经找出了数据库和表，接下来要做的，就是弄清楚数据库中正在执行什么样的 SQL了







## Redis响应严重延迟

- top、iostat、pidstat、strace
- lsof
  - 结合磁盘写的现象，我们知道，只有 7 号普通文件才会产生磁盘写，而它操作的文件路径是 /data/appendonly.aof，相应的系统调用包括 write 和 fdatasync
  - 这对应着正是 Redis 持久化配置中的 appendonly 和 appendfsync 选项
- 查询 appendonly 和 appendfsync 的配置
  - 从这个结果你可以发现，appendfsync 配置的是 always，而 appendonly 配置的是yes。
  - appendfsync 配置的是 always，意味着每次写数据时，都会调用一次 fsync，从而造成比较大的磁盘 I/O 压力。
-  iowait不代表磁盘I/O存在瓶颈，只是代表CPU上I/O操作的时间占用的百分比。假如这时候没有其他进程在运行，那么很小的I/O就会导致iowait升高
- 进程iowait高，磁盘iowait不高，说明是单个进程使用了一些blocking的磁盘打开方式，比如每次都fsync







## 如何分析出系统I/O的瓶颈

- 性能指标
  - ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yMTEwNTgwNi05ZGIzYmZmNTgyNmRmNDBmLnBuZw?x-oss-process=image/format,png)
  - **文件系统 I/O 性能指标**
    - 首先，最容易想到的是存储空间的使用情况，包括容量、使用量以及剩余空间等
      - 不过要注意，这些只是文件系统向外展示的空间使用，而非在磁盘空间的真实用量，因为文件系统的元数据也会占用磁盘空间。
      - 而且，如果你配置了 RAID，从文件系统看到的使用量跟实际磁盘的占用空间，也会因为RAID 级别的不同而不一样。比方说，配置RAID10 后，你从文件系统最多也只能看到所有磁盘容量的一半。
      - 除了数据本身的存储空间，还有一个容易忽略的是索引节点的使用情况，它也包括容量、使用量以及剩余量等三个指标。如果文件系统中存储过多的小文件，就可能碰到索引节点容量已满的问题
    - 其次，你应该想到的是前面多次提到过的缓存使用情况，包括页缓存、目录项缓存、索引节点缓存以及各个具体文件系统（如 ext4、XFS 等）的缓存
    - 除了以上这两点，文件 I/O 也是很重要的性能指标，包括 IOPS（包括 r/s 和 w/s）、响应时间（延迟）以及吞吐量（B/s）等
  - **磁盘 I/O 性能指标**
    - 四个核心的磁盘 I/O 指标。
      - **使用率**，是指磁盘忙处理 I/O 请求的百分比。过高的使用率（比如超过 60%）通常意味 着磁盘 I/O 存在性能瓶颈。 
      - **IOPS**（Input/Output Per Second），是指每秒的 I/O 请求数。 
      - **吞吐量**，是指每秒的 I/O 请求大小。
      - **响应时间**，是指从发出 I/O 请求到收到响应的间隔时间。 
    - 考察这些指标时，一定要注意综合 I/O 的具体场景来分析，比如读写类型（顺序还是随机）、读写比例、读写大小、存储类型（有无RAID 以及 RAID 级别、本地存储还是网络存储）等
    - 缓冲区（Buffer）也是要重点掌握的指标，它经常出现在内存和磁盘问题的分析中
- **性能工具**
  - 第一，在文件系统的原理中，我介绍了查看文件系统容量的工具 df。它既可以查看文件系统数据的空间容量，也可以查看索引节点的容量。至于文件系统缓存，我们通过 /proc/meminfo、/proc/slabinfo 以及 slabtop 等各种来源，观察页缓存、目录项缓存、索引节点缓存以及具体文件系统的缓存情况。
  - 第二，在磁盘 I/O 的原理中，我们分别用 iostat 和 pidstat 观察了磁盘和进程的 I/O 情况。它们都是最常用的 I/O 性能分析工具。通过 iostat ，我们可以得到磁盘的 I/O 使用 率、吞吐量、响应时间以及 IOPS 等性能指标；而通过 pidstat ，则可以观察到进程的 I/O吞吐量以及块设备 I/O 的延迟等。 
  - 第三，在狂打日志的案例中，我们先用 top 查看系统的 CPU 使用情况，发现 iowait 比较 高；然后，又用 iostat 发现了磁盘的 I/O 使用率瓶颈，并用 pidstat 找出了大量 I/O 的进程；最后，通过 strace 和 lsof，我们找出了问题进程正在读写的文件，并最终锁定性能问题的来源——原来是进程在狂打日志。 
  - 第四，在磁盘 I/O 延迟的单词热度案例中，我们同样先用 top、iostat ，发现磁盘有 I/O 瓶颈，并用 pidstat 找出了大量 I/O 的进程。可接下来，想要照搬上次操作的我们失败 了。在随后的 strace 命令中，我们居然没看到 write 系统调用。于是，我们换了一个思路，用新工具 filetop 和 opensnoop ，从内核中跟踪系统调用，最终找出瓶颈的来源。 
  - 最后，在 MySQL 和 Redis 的案例中，同样的思路，我们先用 top、iostat 以及 pidstat ， 确定并找出 I/O 性能问题的瓶颈来源，它们正是 mysqld 和 redis-server。随后，我们又用 strace+lsof 找出了它们正在读写的文件。 
  - 关于 MySQL 案例，根据 mysqld 正在读写的文件路径，再结合 MySQL 数据库引擎的原理，我们不仅找出了数据库和数据表的名称，还进一步发现了慢查询的问题，最终通过优化索引解决了性能瓶颈。
  - 至于 Redis 案例，根据 redis-server 读写的文件，以及正在进行网络通信的 TCPSocket，再结合 Redis 的工作原理，我们发现 Redis 持久化选项配置有问题；从 TCP Socket 通信的数据中，我们还发现了客户端的不合理行为。于是，我们修改 Redis 配置选 项，并优化了客户端使用 Redis 的方式，从而减少网络通信次数，解决性能问题

## **磁盘** **I/O** **性能优化的几个思路**

- **I/O** **基准测试**

  - 为了更客观合理地评估优化效果，我们首先应该对磁盘和文件系统进行基准测试，得到文件系统或者磁盘 I/O 的极限性能。 

  - fio（Flexible I/O Tester）正是最常用的文件系统和磁盘 I/O 性能基准测试工具。它提供 了大量的可定制化选项，可以用来测试，裸盘或者文件系统在各种场景下的 I/O 性能，包括了不同块大小、不同 I/O 引擎以及是否使用缓存等场景。

  - fio 的选项非常多， 我会通过几个常见场景的测试方法，介绍一些最常用的选项。这些常见场景包括随机读、随机写、顺序读以及顺序写等，你可以执行下面这些命令来测试：

    ```text
    #随机读
    fio -name=randread -direct=1 -iodepth=64 -rw=randread -ioengine=libaio -bs=4k -size=1G
    #随机写
    fio -name=randwrite -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=4k -size=1G
    #顺序读
    fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=4k -size=1G -numjobs=
    #顺序写
    fio -name=write -direct=1 -iodepth=64 -rw=write -ioengine=libaio -bs=4k -size=1G -numjob
    ```

    在这其中，有几个参数需要你重点关注一下。

    - direct，表示是否跳过系统缓存。上面示例中，我设置的 1 ，就表示跳过系统缓存。

    - iodepth，表示使用异步 I/O（asynchronous I/O，简称 AIO）时，同时发出的 I/O 请求上限。在上面的示例中，我设置的是 64。

    - rw，表示 I/O 模式。我的示例中， read/write 分别表示顺序读 / 写，而randread/randwrite 则分别表示随机读 / 写。

    - ioengine，表示 I/O 引擎，它支持同步（sync）、异步（libaio）、内存映射（mmap）、网络（net）等各种 I/O 引擎。上面示例中，我设置的 libaio 表示使用异步I/O。

    - bs，表示 I/O 的大小。示例中，我设置成了 4K（这也是默认值）。

    - filename，表示文件路径，当然，它可以是磁盘路径（测试磁盘性能），也可以是文件路径（测试文件系统性能）。示例中，我把它设置成了磁盘 /dev/sdb。不过注意，用磁盘路径测试写，会破坏这个磁盘中的文件系统，所以在使用前，你一定要事先做好数据备份。

    - 这个报告中，需要我们重点关注的是， slat、clat、lat ，以及 bw 和 iops 这几行。

    - 先来看刚刚提到的前三个参数。事实上，slat、clat、lat 都是指 I/O 延迟（latency）。不同之处在于：

      - slat ，是指从 I/O 提交到实际执行 I/O 的时长（Submission latency）；
      - clat ，是指从 I/O 提交到 I/O 完成的时长（Completion latency）；
      - 而 lat ，指的是从 fio 创建 I/O 到 I/O 完成的总时长。

      这里需要注意的是，对同步 I/O 来说，由于 I/O 提交和 I/O 完成是一个动作，所以 slat 实际上就是 I/O 完成的时间，而 clat 是 0。而从示例可以看到，使用异步 I/O（libaio）时，lat 近似等于 slat + clat 之和。

      再来看 bw ，它代表吞吐量。在我上面的示例中，你可以看到，平均吞吐量大约是 16MB（17005 KiB/1024）。

      最后的 iops ，其实就是每秒 I/O 的次数，上面示例中的平均 IOPS 为 4250。

    - 幸运的是，fio 支持 I/O 的重放。借助前面提到过的 blktrace，再配合上 fio，就可以实现对应用程序 I/O 模式的基准测试。你需要先用 blktrace ，记录磁盘设备的 I/O 访问情况；然后使用 fio ，重放 blktrace 的记录。

      比如你可以运行下面的命令来操作：

      ```text
      #使用blktrace跟踪磁盘I/O，注意指定应用程序正在操作的磁盘
      $ blktrace /dev/sdb
      #查看blktrace记录的结果
      # ls
      sdb.blktrace.0  sdb.blktrace.1
      #将结果转化为二进制文件
      $ blkparse sdb -d sdb.bin
      #使用fio重放日志
      $ fio --name=replay --filename=/dev/sdb --direct=1 --read_iolog=sdb.bin
      ```

      这样，我们就通过 blktrace+fio 的组合使用，得到了应用程序 I/O 模式的基准测试报告。

- **I/O性能优化**

  - ![img](https://pic2.zhimg.com/80/v2-659237ee83559f10ff60837d591549fd_720w.jpg)

  - **应用程序优化**

    - 应用程序处于整个 I/O 栈的最上端，它可以通过系统调用，来调整 I/O 模式（如顺序还是随机、同步还是异步）， 同时，它也是 I/O 数据的最终来源。在我看来，可以有这么几种方式来优化应用程序的 I/O 性能。
    - 第一，可以用追加写代替随机写，减少寻址开销，加快 I/O 写的速度。
    - 第二，可以借助缓存 I/O ，充分利用系统缓存，降低实际 I/O 的次数。
    - 第三，可以在应用程序内部构建自己的缓存，或者用 Redis 这类外部缓存系统。这样，一方面，能在应用程序内部，控制缓存的数据和生命周期；另一方面，也能降低其他应用程序使用缓存对自身的影响。
      - 比如，在前面的 MySQL 案例中，我们已经见识过，只是因为一个干扰应用清理了系统缓存，就会导致 MySQL 查询有数百倍的性能差距（0.1s vs 15s）。
      - 再如， C 标准库提供的 fopen、fread 等库函数，都会利用标准库的缓存，减少磁盘的操作。而你直接使用 open、read 等系统调用时，就只能利用操作系统提供的页缓存和缓冲区等，而没有库函数的缓存可用。
    - 第四，在需要频繁读写同一块磁盘空间时，可以用 mmap 代替 read/write，减少内存的拷贝次数。
    - 第五，在需要同步写的场景中，尽量将写请求合并，而不是让每个请求都同步写入磁盘，即可以用 fsync() 取代 O_SYNC。
    - 第六，在多个应用程序共享相同磁盘时，为了保证 I/O 不被某个应用完全占用，推荐你使用 cgroups 的 I/O 子系统，来限制进程 / 进程组的 IOPS 以及吞吐量。
    - 最后，在使用 CFQ 调度器时，可以用 ionice 来调整进程的 I/O 调度优先级，特别是提高核心应用的 I/O 优先级。ionice 支持三个优先级类：Idle、Best-effort 和 Realtime。其中， Best-effort 和 Realtime 还分别支持 0-7 的级别，数值越小，则表示优先级别越高。

  - **文件系统优化**

    - 应用程序访问普通文件时，实际是由文件系统间接负责，文件在磁盘中的读写。所以，跟文件系统中相关的也有很多优化 I/O 性能的方式。
    - 第一，你可以根据实际负载场景的不同，选择最适合的文件系统。比如 Ubuntu 默认使用ext4 文件系统，而 CentOS 7 默认使用 xfs 文件系统。
      - 相比于 ext4 ，xfs 支持更大的磁盘分区和更大的文件数量，如 xfs 支持大于 16TB 的磁盘。但是 xfs 文件系统的缺点在于无法收缩，而 ext4 则可以。
    - 第二，在选好文件系统后，还可以进一步优化文件系统的配置选项，包括文件系统的特性（如 ext_attr、dir_index）、日志模式（如 journal、ordered、writeback）、挂载选项（如 noatime）等等。
      - 比如，使用 tune2fs 这个工具，可以调整文件系统的特性（tune2fs 也常用来查看文件系统超级块的内容）。 而通过 /etc/fstab ，或者 mount 命令行参数，我们可以调整文件系统的日志模式和挂载选项等。
    - 第三，可以优化文件系统的缓存。
      - 比如，你可以优化 pdflush 脏页的刷新频率（比如设置 dirty_expire_centisecs 和dirty_writeback_centisecs）以及脏页的限额（比如调整 dirty_background_ratio 和dirty_ratio 等）
      - 再如，你还可以优化内核回收目录项缓存和索引节点缓存的倾向，即调整vfs_cache_pressure（/proc/sys/vm/vfs_cache_pressure，默认值 100），数值越大，就表示越容易回收。
    - 最后，在不需要持久化时，你还可以用内存文件系统tmpfs，以获得更好的 I/O 性能 。tmpfs 把数据直接保存在内存中，而不是磁盘中。比如 /dev/shm/ ，就是大多数 Linux 默认配置的一个内存文件系统，它的大小默认为总内存的一半。

  - **磁盘优化**

    - 第一，最简单有效的优化方法，就是换用**性能更好的磁盘**，比如用 SSD 替代 HDD。

    - 第二，我们可以使用 **RAID** ，把多块磁盘组合成一个逻辑磁盘，构成冗余独立磁盘阵列。这样做既可以提高数据的可靠性，又可以提升数据的访问性能。

    - 第三，针对磁盘和应用程序 I/O 模式的特征，我们可以选择最适合的 **I/O 调度算法**。比方说，SSD 和虚拟机中的磁盘，通常用的是 noop 调度算法。而数据库应用，我更推荐使用deadline 算法。

    - 第四，我们可以对应用程序的数据，进行**磁盘级别的隔离**。比如，我们可以为日志、数据库等 I/O 压力比较重的应用，配置单独的磁盘。

    - 第五，**在顺序读比较多的场景中，我们可以增大磁盘的预读数据**，比如，你可以通过下面两种方法，调整 /dev/sdb 的预读大小。

      > 调整内核选项 /sys/block/sdb/queue/read_ahead_kb，默认大小是 128 KB，单位为KB。 使用 blockdev 工具设置，比如 blockdev --setra 8192 /dev/sdb，注意这里的单位是512B（0.5KB），所以它的数值总是 read_ahead_kb 的两倍。

    - 第六，我们可以**优化内核块设备 I/O 的选项**。比如，可以调整磁盘队列的长度/sys/block/sdb/queue/nr_requests，适当增大队列长度，可以提升磁盘的吞吐量（当然也会导致 I/O 延迟增大）。

    - 最后，要注意，**磁盘本身出现硬件错误**，也会导致 I/O 性能急剧下降，所以发现磁盘性能急剧下降时，你还需要确认，磁盘本身是不是出现了硬件错误。

      - 比如，你可以查看 dmesg 中是否有硬件 I/O 故障的日志。还可以使用 badblocks、smartctl 等工具，检测磁盘的硬件问题，或用 e2fsck 等来检测文件系统的错误。如果发现问题，你可以使用 fsck 等工具来修复。

    # 

























