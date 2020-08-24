# Centos7搭建NFS-Server-20191019


1. 在k8s-1安装nfs server
    nfs客户端和服务端都安装nfs-utils包，安装时会同时安装rpcbind。安装后会创建nfsnobody用户和组，uid和gid都是65534。

    ```shell
    yum install -y nfs-utils
    ```

2. 配置端口
    nfs 除了主程序端口 2049 和 rpcbind 的端口 111 是固定的，还会使用一些随机端口，以下配置将定义这些端口，以便配置防火墙。

    ```shell
    vim /etc/sysconfig/nfs
    ```

    搜索PORT，取消注释

    结果如下：

    ```shell
    [root@k8s-1 nfs-client]# cat /etc/sysconfig/nfs
    #
    # Note: For new values to take effect the nfs-config service
    # has to be restarted with the following command:
    #    systemctl restart nfs-config
    #
    # Optional arguments passed to in-kernel lockd
    #LOCKDARG=
    # TCP port rpc.lockd should listen on.
    LOCKD_TCPPORT=32803
    # UDP port rpc.lockd should listen on.
    LOCKD_UDPPORT=32769
    #
    # Optional arguments passed to rpc.nfsd. See rpc.nfsd(8)
    RPCNFSDARGS=""
    # Number of nfs server processes to be started.
    # The default is 8.
    #RPCNFSDCOUNT=16
    #
    # Set V4 grace period in seconds
    #NFSD_V4_GRACE=90
    #
    # Set V4 lease period in seconds
    #NFSD_V4_LEASE=90
    #
    # Optional arguments passed to rpc.mountd. See rpc.mountd(8)
    RPCMOUNTDOPTS=""
    # Port rpc.mountd should listen on.
    MOUNTD_PORT=892
    #
    # Optional arguments passed to rpc.statd. See rpc.statd(8)
    STATDARG=""
    # Port rpc.statd should listen on.
    STATD_PORT=662
    # Outgoing port statd should used. The default is port
    # is random
    STATD_OUTGOING_PORT=2020
    # Specify callout program
    #STATD_HA_CALLOUT="/usr/local/bin/foo"
    #
    #
    # Optional arguments passed to sm-notify. See sm-notify(8)
    SMNOTIFYARGS=""
    #
    # Optional arguments passed to rpc.idmapd. See rpc.idmapd(8)
    RPCIDMAPDARGS=""
    #
    # Optional arguments passed to rpc.gssd. See rpc.gssd(8)
    # Note: The rpc-gssd service will not start unless the
    #       file /etc/krb5.keytab exists. If an alternate
    #       keytab is needed, that separate keytab file
    #       location may be defined in the rpc-gssd.service's
    #       systemd unit file under the ConditionPathExists
    #       parameter
    RPCGSSDARGS=""
    #
    # Enable usage of gssproxy. See gssproxy-mech(8).
    GSS_USE_PROXY="yes"
    #
    # Optional arguments passed to blkmapd. See blkmapd(8)
    BLKMAPDARGS=""
    ```

3. 添加防火墙端口（k8s集群防火墙已关闭，故不需执行）
    firewall-cmd --zone=public --add-port=111/tcp --add-port=111/udp --add-port=2049/tcp --add-port=2049/udp --add-port=662/tcp --add-port=662/udp --add-port=892/tcp --add-port=892/udp --add-port=32803/tcp --add-port=32769/udp --add-port=2020/tcp --add-port=2020/udp --permanent

    重载防火墙规则
    firewall-cmd --reload

    移除防火墙端口
    firewall-cmd --zone=public --remove-port=111/tcp --remove-port=111/udp --remove-port=2049/tcp --remove-port=2049/udp --remove-port=662/tcp --remove-port=662/udp --remove-port=892/tcp --remove-port=892/udp --remove-port=32803/tcp --remove-port=32769/udp --remove-port=2020/tcp --remove-port=2020/udp --permanent

4. 配置共享文件夹

    ```shell
    # cat /etc/exports
    /home/nfs/data 192.168.16.0/24(rw,sync,no_root_squash)
    ```

    - exports文件配置格式: NFS共享的目录 NFS客户端地址1(参数1,参数2,...) 客户端地址2(参数1,参数2,...)
  
        NFS客户端地址：
        - 指定IP: 192.168.0.1
        - 指定子网所有主机: 192.168.0.0/24
        - 指定域名的主机: test.com
        - 指定域名所有主机: *.test.com
        - 所有主机: *

    - 参数说明:
        - ro：共享目录只读
        - rw：共享目录可读可写
        - all_squash：所有访问用户都映射为匿名用户或用户组
        - no_all_squash（默认）：访问用户先与本机用户匹配，匹配失败后再映射为匿名用户或用户组
        - root_squash（默认）：将来访的root用户映射为匿名用户或用户组
        - no_root_squash：来访的root用户保持root帐号权限
        - `anonuid=<UID>`：指定匿名访问用户的本地用户UID，默认为nfsnobody（65534）
        - `anongid=<GID>`：指定匿名访问用户的本地用户组GID，默认为nfsnobody（65534）
        - secure（默认）：限制客户端只能从小于1024的tcp/ip端口连接服务器
        - insecure：允许客户端从大于1024的tcp/ip端口连接服务器
        - sync：将数据同步写入内存缓冲区与磁盘中，效率低，但可以保证数据的一致性
        - async：将数据先保存在内存缓冲区中，必要时才写入磁盘
        - wdelay（默认）：检查是否有相关的写操作，如果有则将这些写操作一起执行，这样可以提高效率
        - no_wdelay：若有写操作则立即执行，应与sync配合使用
        - subtree_check（默认） ：若输出目录是一个子目录，则nfs服务器将检查其父目录的权限
        - no_subtree_check ：即使输出目录是一个子目录，nfs服务器也不检查其父目录的权限，这样可以提高效率

    注意：使用nfs作为k8s storageclass provision是，参数必须指定为**no_root_squash**

5. 启动nfs server并配置开机启动

    ```shell
    systemctl enable --now rpcbind
    systemctl enable --now nfs
    ```

6. 客户端挂载测试
    在k8s-2上执行挂载命令

    ```shell
    [root@k8s-2 home]# mkdir test
    [root@k8s-2 home]# mount -t nfs 192.168.16.131:/home/nfs/data /home/test
    ```

    进入/home/test目录，新建文件，然后查看k8s-1的/home/nfs/data目录下是否存在该文件，若存在，则nfs server搭建成功。

7. 客户端配置开机自动挂载
    编辑 /etc/fstab，添加
    192.168.16.131:/home/nfs/data  /home/test  nfs  default  0 0

8. 其他命令：
    - 查看nfs服务器共享目录showmount
      - -e：显示指定的NFS服务器上所有输出的共享目录。
      - -a：显示指定的NFS服务器的所有客户端主机及其所连接的目录。
      - -d：显示指定的NFS服务器中已被客户端连接的所有输出目录

    - 管理当前NFS共享的文件系统列表 exportfs
      - -a：打开或取消所有目录共享。
      - -o：options,…指定一列共享选项，与 exports(5) 中讲到的类似。
      - -i：忽略 /etc/exports 文件，从而只使用默认的和命令行指定的选项。
      - -r：重新共享所有目录。它使 /var/lib/nfs/xtab 和 /etc/exports 同步。 它将 /etc/exports 中已删除的条目从 /var/lib/nfs/xtab 中删除，将内核共享表中任何不再有效的条目移除。
      - -u：取消一个或多个目录的共享。
      - -f：在“新”模式下，刷新内核共享表之外的任何东西。 任何活动的客户程序将在它们的下次请求中得到 mountd添加的新的共享条目。
      - -v：输出详细信息。当共享或者取消共享时，显示在做什么。 显示当前共享列表的时候，同时显示共享的选项。

    - 列出NFS客户端和服务器的工作状态 nfsstat
      - -s：仅列出NFS服务器端状态；
      - -c：仅列出NFS客户端状态；
      - -n：仅列出NFS状态，默认显示nfs客户端和服务器的状态；
      - -2：仅列出NFS版本2的状态；
      - -3：仅列出NFS版本3的状态；
      - -4：仅列出NFS版本4的状态；
      - -m：打印以加载的nfs文件系统状态；
      - -r：仅打印rpc状态。

