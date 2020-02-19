[toc]



# Linux性能调优

## IO性能篇

### 23 Linux文件系统如何工作

#### 索引节点和目录项

* 索引节点，简称inode，和文件一一对应，存储在磁盘中，记录文件的元数据
* 目录项，dentry，记录文件的名字、索引节点以及其他目录项的关联关系



举例说明，为文件创建的硬链接，会对应不同的目录项，他们都连接到同一个文件，索引节点相同



磁盘的最小单位是**扇区**，文件系统将连续的扇区组成逻辑块，以逻辑块为最小单位，来读写磁盘数据。常见的逻辑块4KB，由连续的8个扇区组成。

**示意图**

![img](https://static001.geekbang.org/resource/image/32/47/328d942a38230a973f11bae67307be47.png)



磁盘在执行文件系统格式化时，分为三个区域：超级块、索引节点和数据块区

* 超级块：整个文件系统的状态
* 索引节点区：存储索引节点
* 数据块区：存储文件数据



#### 虚拟文件系统

**示意图**

![img](https://static001.geekbang.org/resource/image/72/12/728b7b39252a1e23a7a223cdf4aa1612.png)

文件系统分类：

* 基于磁盘的文件系统：常见的 Ext4、XFS、OverlayFS 等，都是这类文件系统
* 基于内存的文件系统：常说的虚拟文件系统，不需要磁盘空间，但是占用内存。比如，/proc和/sys
* 网络文件系统：用来访问其他计算机的文件系统，比如NFS、SMB、iSCSI 等

**注意**：这些文件系统，要先挂载到 VFS 目录树中的某个子目录（称为**挂载点**），然后才能访问其中的文件。



#### 文件系统IO

根据是否利用标准库缓存，分为缓冲IO和非缓冲IO

* 缓存IO：利用标准库缓，加速文件访问，标准库内部利用系统调用访问文件
* 非缓存IO：直接通过系统调用访问文件，不再经过标准库缓存

**注意**：这里的“缓冲”，是指标准库内部实现的缓存，最终还是需要通过系统调用，而系统调用还会通过**页缓存**，来减少磁盘的IO操作



根据是否利用操作系统的页缓存，分为直接IO和非直接IO

* 直接IO：跳过操作系统的页缓存，直接和**文件系统**交互来访问文件
* 非直接IO：先通过页缓存，再通过内核或者额外的系统调用，真正和磁盘交互（`O_DIRECT`标志）



根据应用程序是否阻塞自身，分为阻塞IO和非阻塞IO



根据是否等待相应结果，分为同步IO和异步IO

* 同步IO：应用程序执行IO操作之后，要等到整个IO完成后，才能获得IO响应
* 异步IO：应用程序不用等待IO完成，会继续执行，等到IO执行完成，会以事件的方式通知应用程序

设置`O_SYNC`或者`O_DSYNC`，代表同步IO。如果是`O_DSYNC`，要等到文件数据写入磁盘之后，才能返回，如果是`O_SYNC`，是在`O_DSYNC`的基础上，要求文件**元数据**写入磁盘，才返回



设置`O_ASYNC`，代表异步IO，系统会再通过`SIGIO`或者`SIGPOLL`通知进程

#### 性能观测

##### 容量

`df`命令查看磁盘空间

```bash
$ df -h /dev/sda1 
Filesystem      Size  Used Avail Use% Mounted on 
/dev/sda1        29G  3.1G   26G  11% / 

# 查看索引节点所占的空间
$ df -i /dev/sda1 
Filesystem      Inodes  IUsed   IFree IUse% Mounted on 
/dev/sda1      3870720 157460 3713260    5% / 
```

当索引节点空间不足，但是磁盘空间充足时，很可能是过多小文件导致的。**解决方法**一般是删除这些小文件，或者移动到索引节点充足的其他磁盘区

##### 缓存

可以使用free或者vmstat，观察页缓存的大小

也可以查看/proc/meminfo

```bash
$ cat /proc/meminfo | grep -E "SReclaimable|Cached" 
Cached:           748316 kB 
SwapCached:            0 kB 
SReclaimable:     179508 kB 
```

内核使用slab机制，管理目录项和索引节点的缓存，/proc/meminfo给出了整体的slab大小，/proc/slabinfo可以查看每一种slab的缓存

```bash

$ cat /proc/slabinfo | grep -E '^#|dentry|inode' 
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail> 
xfs_inode              0      0    960   17    4 : tunables    0    0    0 : slabdata      0      0      0 
... 
ext4_inode_cache   32104  34590   1088   15    4 : tunables    0    0    0 : slabdata   2306   2306      0hugetlbfs_inode_cache     13     13    624   13    2 : tunables    0    0    0 : slabdata      1      1      0 
sock_inode_cache    1190   1242    704   23    4 : tunables    0    0    0 : slabdata     54     54      0 
shmem_inode_cache   1622   2139    712   23    4 : tunables    0    0    0 : slabdata     93     93      0 
proc_inode_cache    3560   4080    680   12    2 : tunables    0    0    0 : slabdata    340    340      0 
inode_cache        25172  25818    608   13    2 : tunables    0    0    0 : slabdata   1986   1986      0 
dentry             76050 121296    192   21    1 : tunables    0    0    0 : slabdata   5776   5776      0 
```

其中，dentry代表目录项缓存，inode_cache代表VFS索引节点缓存，其他的就是各种文件系统的索引节点缓存



实际性能分析中，更常使用slabtop命令，来找出占用内存最多的缓存类型

示例如下：可以看到，目录项和索引节点占用了最多的 Slab 缓存，总共大约23M

```bash

# 按下c按照缓存大小排序，按下a按照活跃对象数排序 
$ slabtop 
Active / Total Objects (% used)    : 277970 / 358914 (77.4%) 
Active / Total Slabs (% used)      : 12414 / 12414 (100.0%) 
Active / Total Caches (% used)     : 83 / 135 (61.5%) 
Active / Total Size (% used)       : 57816.88K / 73307.70K (78.9%) 
Minimum / Average / Maximum Object : 0.01K / 0.20K / 22.88K 

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME 
69804  23094   0%    0.19K   3324       21     13296K dentry 
16380  15854   0%    0.59K   1260       13     10080K inode_cache 
58260  55397   0%    0.13K   1942       30      7768K kernfs_node_cache 
   485    413   0%    5.69K     97        5      3104K task_struct 
  1472   1397   0%    2.00K     92       16      2944K kmalloc-2048 
```



