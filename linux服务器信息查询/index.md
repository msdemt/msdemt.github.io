# Linux服务器信息查询



> 参考：  
> <https://blog.csdn.net/l1394049664/article/details/81811642>  
> <https://blog.csdn.net/zhangliao613/article/details/79021606>

1. 查看 CPU 型号

    ```shell
    cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
    ```

    以下表示该主机的 CPU 型号为 Intel(R) Xeon(R) CPU E5-2620 v2 @ 2.10GHz ，并且该主机共有 24 个逻辑 CPU。

    ```shell
    [root@k8s-m1 ~]# cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
     24  Intel(R) Xeon(R) CPU E5-2620 v2 @ 2.10GHz
    ```

2. 查看物理 CPU 个数

    ```shell
    cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
    ```

    以下表示该主机存在两个物理 CPU：

    ```shell
    [root@k8s-m1 ~]# cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
    2
    ```

3. 查看每个 CPU 中 core 的个数（即核数）

    ```shell
    cat /proc/cpuinfo| grep "cpu cores"| uniq
    ```

    以下表示该主机每个物理 CPU 存在 6 个核心。

    ```shell
    [root@k8s-m1 ~]# cat /proc/cpuinfo| grep "cpu cores"| uniq
    cpu cores    : 6
    ```

4. 查看逻辑 CPU 的个数

    ```shell
    cat /proc/cpuinfo| grep "processor"| wc -l
    ```

    以下表示该主机存在 24 个逻辑 CPU

    ```shell
    [root@k8s-m1 ~]# cat /proc/cpuinfo| grep "processor"| wc -l
    24
    ```

    **CPU 总核数** = 物理 CPU 个数 x 每颗物理 CPU 的核数  
    **逻辑 CPU 总数** = 物理 CPU 个数 x 每颗物理 CPU 的核数 x 超线程数

5. CPU 架构
    1. 单核 CPU，每个 CPU 通过总线进行通信，效率比较低，如下图

        ![image](../../images/cpu-architecture-1-20191013.png)

    2. 多核 CPU，每个 CPU 内不同的核心通过 L2 cache 进行通信，存储和外设通过总线与 CPU 通信，如下图

        ![image](../../images/cpu-architecture-2-20191013.png)

    3. 多核超线程 CPU，每个 CPU 内的每个核心都有两个逻辑处理单元（逻辑 CPU），两个逻辑 CPU 共享一个核心的资源，如下图

        ![image](../../images/cpu-architecture-3-20191013.png)

6. 查看内存信息

    ```shell
    cat /proc/meminfo
    ```

    如下所示：

    ```shell
    [root@k8s-m1 ~]# cat /proc/meminfo
    MemTotal:       16379240 kB
    MemFree:         9860512 kB
    MemAvailable:   14610972 kB
    Buffers:            2128 kB
    Cached:          4662128 kB
    SwapCached:            0 kB
    Active:          2300852 kB
    Inactive:        3258944 kB
    Active(anon):     763292 kB
    Inactive(anon):      964 kB
    Active(file):    1537560 kB
    Inactive(file):  3257980 kB
    Unevictable:           0 kB
    Mlocked:               0 kB
    SwapTotal:             0 kB
    SwapFree:              0 kB
    Dirty:               332 kB
    Writeback:             0 kB
    AnonPages:        874296 kB
    Mapped:           469584 kB
    Shmem:              2140 kB
    KReclaimable:     351024 kB
    Slab:             703168 kB
    SReclaimable:     351024 kB
    SUnreclaim:       352144 kB
    KernelStack:       15824 kB
    PageTables:         9076 kB
    NFS_Unstable:          0 kB
    Bounce:                0 kB
    WritebackTmp:          0 kB
    CommitLimit:     8189620 kB
    Committed_AS:    3189116 kB
    VmallocTotal:   34359738367 kB
    VmallocUsed:           0 kB
    VmallocChunk:          0 kB
    Percpu:            41472 kB
    HardwareCorrupted:     0 kB
    AnonHugePages:    428032 kB
    ShmemHugePages:        0 kB
    ShmemPmdMapped:        0 kB
    HugePages_Total:       0
    HugePages_Free:        0
    HugePages_Rsvd:        0
    HugePages_Surp:        0
    Hugepagesize:       2048 kB
    Hugetlb:               0 kB
    DirectMap4k:      313004 kB
    DirectMap2M:     8040448 kB
    DirectMap1G:     8388608 kB
    ```

    MemTotal 为当前主机的总内存，16379240 kB = 15GB

    或者使用 `free -h` 查看内存容量情况

    ```shell
    [root@k8s-m1 ~]# free -h
                  total        used        free      shared  buff/cache   available
    Mem:            15G        1.1G        9.4G        2.1M        5.1G         13G
    Swap:            0B          0B          0B
    ```

7. 更多命令

    ```shell
    uname -a # 查看内核/操作系统/CPU信息的linux系统信息  
    head -n l /etc/issue # 查看操作系统版本  
    cat /proc/cpuinfo # 查看CPU信息  
    hostname # 查看计算机名的linux系统信息命令  
    lspci -tv # 列出所有PCI设备  
    lsusb -tv # 列出所有USB设备的linux系统信息命令  
    lsmod # 列出加载的内核模块  
    env # 查看环境变量资源  
    free -m # 查看内存使用量和交换区使用量  
    df -h # 查看各分区使用情况  
    du -sh # 查看指定目录的大小  
    grep MemTotal /proc/meminfo # 查看内存总量  
    grep MemFree /proc/meminfo # 查看空闲内存量  
    uptime # 查看系统运行时间、用户数、负载  
    cat /proc/loadavg # 查看系统负载磁盘和分区  
    mount | column -t # 查看挂接的分区状态  
    fdisk -l # 查看所有分区  
    swapon -s # 查看所有交换分区  
    hdparm -i /dev/hda # 查看磁盘参数(仅适用于IDE设备)  
    dmesg | grep IDE # 查看启动时IDE设备检测状况网络  
    ifconfig # 查看所有网络接口的属性
    iptables -L # 查看防火墙设置
    route -n # 查看路由表
    netstat -lntp # 查看所有监听端口  
    netstat -antp # 查看所有已经建立的连接
    netstat -s # 查看网络统计信息进程  
    ps -ef # 查看所有进程
    top # 实时显示进程状态用户  
    w # 查看活动用户
    id # 查看指定用户信息  
    last # 查看用户登录日志
    cut -d: -f1 /etc/passwd # 查看系统所有用户  
    cut -d: -f1 /etc/group # 查看系统所有组
    crontab -l # 查看当前用户的计划任务服务  
    chkconfig –list # 列出所有系统服务
    chkconfig –list | grep on # 列出所有启动的系统服务程序  
    rpm -qa # 查看所有安装的软件包
    cat /proc/cpuinfo ：查看CPU相关参数的linux系统命令  
    cat /proc/partitions ：查看linux硬盘和分区信息的系统信息命令
    cat /proc/meminfo ：查看linux系统内存信息的linux系统命令  
    cat /proc/version ：查看版本，类似uname -r
    cat /proc/ioports ：查看设备io端口  
    cat /proc/interrupts ：查看中断
    cat /proc/pci ：查看pci设备的信息  
    cat /proc/swaps ：查看所有swap分区的信息  
    ```

