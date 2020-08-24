# k8s v1.15.0 二进制方式部署


## 部署k8s集群

> 参考：  
> <https://github.com/opsnull/follow-me-install-kubernetes-cluster>  
> <https://zhangguanzhang.github.io/2019/03/03/kubernetes-1-13-4/>

**集群信息**：

主机名 | IP | 系统版本 | 角色
---|---|---|---
k8s-m1 | 192.168.15.5 | CentOS Linux release 7.6.1810 | master+worker
k8s-m2 | 192.168.15.6 | CentOS Linux release 7.6.1810 | master+worker
k8s-m3 | 192.168.15.7 | CentOS Linux release 7.6.1810 | master+worker
c48-1 | 192.168.15.161 | CentOS Linux release 7.6.1810 | worker
c48-2 | 192.168.15.143 | CentOS Linux release 7.6.1810 | worker

**网络信息**：

名称 | 地址
---|---
Cluster IP CIDR | 172.30.0.0/16
Service Cluster IP CIDR | 10.96.0.0/12
Service DNS IP | 10.96.0.10
DNS DN | cluster.local

### 准备工作

1. 所有主机彼此网络互通

2. 所有主机的hosts文件中添加主机名与IP的映射。

    执行后，在 k8s-m1 上查看，结果如下所示：

    ```shell
    [root@k8s-m1 ~]# cat /etc/hosts
    127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
    ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
    192.168.15.5  k8s-m1
    192.168.15.6  k8s-m2
    192.168.15.7  k8s-m3
    192.168.15.161 c48-1
    192.168.15.143 c48-2
    ```

    执行后，在 k8s-m1 上查看，结果如下所示：

    ```shell
    [root@k8s-m1 ~]# cat /etc/hosts
    127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
    ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
    192.168.15.5  k8s-m1
    192.168.15.6  k8s-m2
    192.168.15.7  k8s-m3
    192.168.15.161 c48-1
    192.168.15.143 c48-2
    ```

3. 为方便部署，在 k8s-m1 上配置无密码 ssh 登录其他节点。

    **方法一**：在 k8s-m1 上执行如下命令：

    ```shell
    ssh-keygen -t rsa
    ssh-copy-id root@k8s-1
    ssh-copy-id root@k8s-2
    ssh-copy-id root@k8s-3
    ssh-copy-id root@c48-1
    ssh-copy-id root@c48-2
    ```

    **方法二**：在 k8s-m1 上安装sshpass，然后通过配置别名来让 ssh 和 scp 不输入密码，其中 123456 为所有主机的密码。

    ```shell
    yum install -y sshpass
    alias ssh='sshpass -p 123456 ssh -o StrictHostKeyChecking=no'
    alias scp='sshpass -p 123456 scp -o StrictHostKeyChecking=no'
    ```

4. 所有主机关闭防火墙和 SELinux
    所有主机关闭防火墙和 SELinux，否则后续挂载目录时可能报错：Permission denied。

    ```shell
    systemctl disable --now firewalld NetworkManager
    setenforce 0
    sed -ri '/^[^#]*SELINUX=/s#=.+$#=disabled#' /etc/selinux/config
    ```

5. 关闭 dnsmasq
    linux 系统开启了 dnsmasq 后（如GUI环境），将系统 DNS Server 设置为127.0.0.1，这会导致 docker 容器无法解析域名，需要关闭它。（最小安装的 centos7 系统默认不启动 dnsmasq ）

    ```shell
    systemctl disable --now dnsmasq
    ```

6. 关闭 swap 分区
    Kubernetes v1.8+要求关闭系统Swap，若不关闭则需要修改kubelet设定参数（--fail-swap-on设置为false来忽略swap on），在所有主机上使用如下指令关闭swap并注释掉/etc/fstab中swap的行。

    ```shell
    swapoff -a && sysctl -w vm.swappiness=0
    sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
    ```

    > 打开 swap 分区 ：执行下 sysctl -w vm.swappiness=1 然后编辑 vi /etc/fstab 去掉注释 在执行下这个 swapon -a

7. 升级内核（可选）

    1. 更新系统，不升级内核，执行如下命令：

        ```shell
        yum install -y epel-release
        yum install -y wget git jq psmisc socat
        yum update -y
        reboot
        ```

    2. 更新系统，升级内核
        1. 更新系统

            ```shell
            yum install -y epel-release
            yum install -y wget git jq psmisc socat
            yum update -y --exclude=kernel*
            ```

        2. perl 是内核的依赖包，如下命令检测是否存在 perl，如果不存在则安装

            ```shell
            [ ! -f /usr/bin/perl ] && yum install perl -y
            ```

        3. 升级内核需要使用 elrepo 的 yum 源，首先我们导入 elrepo 的 key 并安装 elrepo 源

            ```shell
            rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
            rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
            ```

            查看可用的内核

            ```shell
            yum --disablerepo="*" --enablerepo="elrepo-kernel" list available  --showduplicates
            ```

            在 yum 的 elrepo 源中， mainline 为最新版本的内核， ipvs 依赖 nf_conntrack_ipv4 内核模块， 4.19即后续的内核版本将该内核模块改名为 nf_conntrack。

            > 下面的链接可以下载到其他的归档版本：  
            > ubuntu  <http://kernel.ubuntu.com/~kernel-ppa/mainline/>  
            > RHEL  <http://mirror.rc.usf.edu/compute_lock/elrepo/kernel/el7/x86_64/RPMS/>

            下面是 ml 的内核和上面归档内核版本任选其一的安装方法：
            1. 自选内核版本安装

                ```shell
                export Kernel_Version=4.18.9-1
                wget  http://mirror.rc.usf.edu/compute_lock/elrepo/kernel/el7/x86_64/RPMS/kernel-ml{,-devel}-${Kernel_Version}.el7.elrepo.x86_64.rpm
                yum localinstall -y kernel-ml*
                ```

            2. 最新内核版本安装

                ```shell
                yum --disablerepo="*" --enablerepo="elrepo-kernel" list available  --showduplicates | grep -Po '^kernel-ml.x86_64\s+\K\S+(?=.el7)'
                yum --disablerepo="*" --enablerepo=elrepo-kernel install -y kernel-ml{,-devel}
                ```

        4. 修改内核启动顺序，默认启动的顺序应该为 1，升级后内核是往前插入，为 0

            ```shell
            grub2-set-default  0 && grub2-mkconfig -o /etc/grub2.cfg
            ```

            使用下面的命令确认下启动的默认内核是否指向上面安装的内核

            ```shell
            grubby --default-kernel
            ```

        5. docker官方的内核检查脚本建议（RHEL7/CentOS7: User namespaces disabled; add 'user_namespace.enable=1' to boot command line），使用下面命令开启

            ```shell
            grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
            ```

        6. 重启加载新内核

            ```shell
            reboot
            ```

8. 所有主机安装 ipvs

    centos:

    ```shell
    yum install -y ipvsadm ipset sysstat conntrack libseccomp
    ```

    ubuntu

    ```shell
    sudo apt-get install -y wget git conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
    ```

9. 所有主机配置开机加载的内核模块

    每台主机上执行如下命令：

    ```shell
    :> /etc/modules-load.d/ipvs.conf
    module=(
    ip_vs
    ip_vs_rr
    ip_vs_wrr
    ip_vs_sh
    nf_conntrack
    br_netfilter
    rbd
      )
    for kernel_module in ${module[@]};do
        /sbin/modinfo -F filename $kernel_module |& grep -qv ERROR && echo $kernel_module >> /etc/modules-load.d/ipvs.conf || :
    done
    systemctl enable --now systemd-modules-load.service
    ```

    执行后生成的文件内容如下所示：

    ```shell
    [root@k8s-m1 ~]# cat /etc/modules-load.d/ipvs.conf
    ip_vs
    ip_vs_rr
    ip_vs_wrr
    ip_vs_sh
    nf_conntrack
    rbd
    ```

    > 生成的文件中没有 br_netfilter ，因为新版本系统已经将 br_netfilter 集成在了内核中。
    > rook-ceph 需要使用 rbd 模块，所以加上了 rbd 模块进行开机加载。

10. 所有主机配置系统参数

    配置 /etc/sysctl.d/k8s.conf 的系统参数

    ```shell
    cat <<EOF > /etc/sysctl.d/k8s.conf
    # https://github.com/moby/moby/issues/31208
    # ipvsadm -l --timout
    # 修复ipvs模式下长连接timeout问题 小于900即可
    net.ipv4.tcp_keepalive_time = 600
    net.ipv4.tcp_keepalive_intvl = 30
    net.ipv4.tcp_keepalive_probes = 10
    net.ipv6.conf.all.disable_ipv6 = 1
    net.ipv6.conf.default.disable_ipv6 = 1
    net.ipv6.conf.lo.disable_ipv6 = 1
    net.ipv4.neigh.default.gc_stale_time = 120
    net.ipv4.conf.all.rp_filter = 0
    net.ipv4.conf.default.rp_filter = 0
    net.ipv4.conf.default.arp_announce = 2
    net.ipv4.conf.lo.arp_announce = 2
    net.ipv4.conf.all.arp_announce = 2
    net.ipv4.ip_forward = 1
    net.ipv4.tcp_max_tw_buckets = 5000
    net.ipv4.tcp_syncookies = 1
    net.ipv4.tcp_max_syn_backlog = 1024
    net.ipv4.tcp_synack_retries = 2
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.netfilter.nf_conntrack_max = 2310720
    fs.inotify.max_user_watches=89100
    fs.may_detach_mounts = 1
    fs.file-max = 52706963
    fs.nr_open = 52706963
    net.bridge.bridge-nf-call-arptables = 1
    vm.swappiness = 0
    vm.overcommit_memory=1
    vm.panic_on_oom=0
    net.ipv4.neigh.default.gc_thresh1=8192
    net.ipv4.neigh.default.gc_thresh2=32768
    net.ipv4.neigh.default.gc_thresh3=65536
    EOF

    sysctl --system
    ```

    > 添加 net.ipv4.neigh.default.gc_thresh1-3 是为了解决在 c48-1 上无法 ping 通其他网段 ip 的问题。
    >
    > ```shell
    > [root@c48-1 ~]# ping 1.1.0.11
    > PING 1.1.0.11 (1.1.0.11) 56(84) bytes of data.
    > ping: sendmsg: 无效的参数
    > ping: sendmsg: 无效的参数
    > ping: sendmsg: 无效的参数
    > ```
    >
    > 1.1.0.11 为 c48-1 主机（双网卡）另一个网卡连接的网段的地址  
    > 参考：<https://mozillazg.com/2017/10/linux-a-way-to-fix-haproxy-network-connection-timeout-ping-sendmsg-invalid-argument-socket-errno-110-connection-timed-out>

11. 所有主机安装 docker

    检查系统内核和模块是否适合运行 docker (仅适用于 linux 系统)

    ```shell
    curl https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh > check-config.sh
    bash ./check-config.sh
    ```

    在官方查看 k8s 支持的 docker 版本： <https://github.com/kubernetes/kubernetes> 里进对应版本的 changelog 里搜 **The list of validated docker versions remain**

    1. 使用 docker 的官方安装脚本进行安装，可以使用如下命令查看可用的 docker 版本。

        ```shell
        yum list docker-ce --showduplicates | sort -r
        ```

        这里我使用的版本为 18.06.03

        ```shell
        export VERSION=18.06
        curl -fsSL "https://get.docker.com/" | bash -s -- --mirror Aliyun
        ```

        > 非 root 用户使用 docker  
        >If you would like to use Docker as a non-root user, you should now consider adding your user to the "docker" group with something like:
        >
        > ```shell
        > sudo usermod -aG docker your-user
        > ```
        >
        > Remember that you will have to log out and back in for this to take effect!

    2. 配置 docker 加速源并配置 docker 的启动参数使用 systemd

        ```shell
        mkdir -p /etc/docker/
        cat>/etc/docker/daemon.json<<EOF
        {
          "exec-opts": ["native.cgroupdriver=systemd"],
          "registry-mirrors": ["http://hub-mirror.c.163.com"],
          "storage-driver": "overlay2",
          "storage-opts": [
            "overlay2.override_kernel_check=true"
          ],
          "log-driver": "json-file",
          "log-opts": {
            "max-size": "100m",
            "max-file": "3"
          }
        }
        EOF
        ```

        > k8s 官方建议使用 systemd，详见：<https://kubernetes.io/docs/setup/cri/>

    3. 修改 docker 的数据存储目录

        docker 默认的数据存储目录为 /var/lib/docker，由于该目录所属的磁盘分区较小，所以将其软链接到 /home/data/docker 目录

        ```shell
        rm -rf /var/lib/docker
        mkdir -p /home/data/docker
        ln -s /home/data/docker /var/lib/docker
        ```

    4. 设置 docker 命令补全和开机启动

        ```shell
        yum install -y epel-release bash-completion && cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/
        systemctl enable --now docker
        ```

### 配置 NTP 时间同步

略

### 使用环境变量声明集群信息

将集群信息写进 `env.sh` 文件中，使用 source 命令将文件中的信息加载到环境变量中。

`env.sh` 内容如下所示：

```shell
[root@k8s-m1 k8s]# pwd
/home/k8s
[root@k8s-m1 k8s]# cat env.sh
# 声明集群成员信息
declare -A MasterArray otherMaster NodeArray AllNode Other
MasterArray=(['k8s-m1']=192.168.15.5 ['k8s-m2']=192.168.15.6 ['k8s-m3']=192.168.15.7)
otherMaster=(['k8s-m2']=192.168.15.6 ['k8s-m3']=192.168.15.7)
NodeArray=(['k8s-m1']=192.168.15.5 ['k8s-m2']=192.168.15.6 ['k8s-m3']=192.168.15.7 ['c48-1']=192.168.15.161 ['c48-2']=192.168.15.143)
# 下面复制上面的信息粘贴即可
AllNode=(['k8s-m1']=192.168.15.5 ['k8s-m2']=192.168.15.6 ['k8s-m3']=192.168.15.7 ['c48-1']=192.168.15.161 ['c48-2']=192.168.15.143)
Other=(['k8s-m2']=192.168.15.6 ['k8s-m3']=192.168.15.7 ['c48-1']=192.168.15.161 ['c48-2']=192.168.15.143)

export VIP=127.0.0.1

[ "${#MasterArray[@]}" -eq 1 ]  && export VIP=${MasterArray[@]} || export API_PORT=8443
export KUBE_APISERVER=https://${VIP}:${API_PORT:=6443}

#声明需要安装的的k8s版本
export KUBE_VERSION=v1.15.0

# 网卡名
export interface=ens33

# cni
export CNI_URL="https://github.com/containernetworking/plugins/releases/download"
export CNI_VERSION=v0.7.5

# etcd
export ETCD_version=v3.3.10

# DNS DN
export CLUSTER_DNS_DOMAIN=cluster.local

# DNS IP
export CLUSTER_DNS_SVC_IP=10.96.0.10
```

> 注意：API Server的负载均衡和高可用的方案若使用 haproxy+keepalived ，需要将 VIP 指定为当前局域网内未使用的 ip 。
> 网卡名 interface 为 keepalived 绑定的网卡。

使用如下命令将文件中的信息加载到环境变量中

```shell
source env.sh
```

### 分发二进制文件

1. 在 k8s-m1 上通过 git 获取部署需要的二进制配置和yml文件

    ```shell
    git clone https://github.com/msdemt/k8s-manual-files.git /home/k8s/k8s-manual-files -b v1.15.0
    ```

2. 在 k8s-m1 上下载 k8s v1.15.0 server端的发布包，并将 k8s 的二进制文件复制到 /usr/local/bin 目录下

    ```shell
    wget https://storage.googleapis.com/kubernetes-release/release/${KUBE_VERSION}/kubernetes-server-linux-amd64.tar.gz
    tar -zxvf kubernetes-server-linux-amd64.tar.gz  --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}
    ```

    > 需要翻墙下载

    也可以使用 k8s-manual-files 中提供的 v1.15.0 的二进制文件

    ```shell
    cp /home/k8s/k8s-manual-files/master/bin/* /usr/local/bin/
    ```

3. 将 k8s master 相关的二进制组件分发到其它 master 上

    ```shell
    for NODE in "${!otherMaster[@]}"; do
        echo "--- $NODE ${otherMaster[$NODE]} ---"
        scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} ${otherMaster[$NODE]}:/usr/local/bin/
    done
    ```

4. 将 k8s worker 相关的二进制组件分发到其他的 worker 上

    ```shell
    for NODE in "${!NodeArray[@]}"; do
        echo "--- $NODE ${NodeArray[$NODE]} ---"
        scp /usr/local/bin/kube{let,-proxy} ${NodeArray[$NODE]}:/usr/local/bin/
    done
    ```

5. 在 k8s-m1 上下载 kubernetes cni 二进制文件并分发到其他主机

    ```shell
    mkdir -p /opt/cni/bin
    wget  "${CNI_URL}/${CNI_VERSION}/cni-plugins-amd64-${CNI_VERSION}.tgz"
    tar -zxf cni-plugins-amd64-${CNI_VERSION}.tgz -C /opt/cni/bin

    # 分发cni文件
    for NODE in "${!Other[@]}"; do
        echo "--- $NODE ${Other[$NODE]} ---"
        ssh ${Other[$NODE]} 'mkdir -p /opt/cni/bin'
        scp /opt/cni/bin/* ${Other[$NODE]}:/opt/cni/bin/
    done
    ```

### 建立集群 CA keys 与 Certificates

在这个部分，将需要产生多个元件的 Certificates，这包含 Etcd、 Kubernetes 元件等,并且每个集群都会有一个根数位凭证认证机构（Root Certificate Authority）被用在认证 API Server 与 Kubelet 端的凭证。

PS: 注意 CA JSON 中的 CN(Common Name) 与 O(Organization) 等内容是会影响 Kubernetes 元件认证的。  
**CN (Common Name)**: apiserver 会从证书中提取该字段作为请求的用户名 (User Name)
**O (Organization)**： apiserver  会从证书中提取该字段作为请求用户所属的组 (Group)

**CA (Certificate Authority)** 是自签名的根证书，用来签名后续创建的其它证书。

本文使用**openssl**创建所有证书。

#### 准备 openssl 证书配置文件

将 IP 信息注入到 openssl.cnf 文件中

```shell
mkdir -p /etc/kubernetes/pki/etcd
sed -i "/IP.2/a IP.3 = $VIP" /home/k8s/k8s-manual-files/pki/openssl.cnf
sed -ri '/IP.3/r '<( paste -d '' <(seq -f 'IP.%g = ' 4 $[${#AllNode[@]}+3])  <(xargs -n1<<<${AllNode[@]} | sort) ) /home/k8s/k8s-manual-files/pki/openssl.cnf
sed -ri '$r '<( paste -d '' <(seq -f 'IP.%g = ' 2 $[${#MasterArray[@]}+1])  <(xargs -n1<<<${MasterArray[@]} | sort) ) /home/k8s/k8s-manual-files/pki/openssl.cnf
cp /home/k8s/k8s-manual-files/pki/openssl.cnf /etc/kubernetes/pki/
cd /etc/kubernetes/pki
```

#### 生成证书

1. 生成根证书

    Path | Default CN | Description
    ---|---|---
    ca.crt/key | kubernetes-ca | Kubernetes general CA
    etcd/ca.crt/key | etcd-ca | For all etcd-related functions
    front-proxy-ca.crt/key | kubernetes-front-proxy-ca | For the front-end proxy

    **kubernetes-ca**

    ```shell
    openssl genrsa -out ca.key 2048
    openssl req -x509 -new -nodes -key ca.key -config openssl.cnf -subj "/CN=kubernetes-ca" -extensions v3_ca -out ca.crt -days 10000
    ```

    **etcd-ca**

    ```shell
    openssl genrsa -out etcd/ca.key 2048
    openssl req -x509 -new -nodes -key etcd/ca.key -config openssl.cnf -subj "/CN=etcd-ca" -extensions v3_ca -out etcd/ca.crt -days 10000
    ```

    **front-proxy-ca**

    ```shell
    openssl genrsa -out front-proxy-ca.key 2048
    openssl req -x509 -new -nodes -key front-proxy-ca.key -config openssl.cnf -subj "/CN=kubernetes-ca" -extensions v3_ca -out front-proxy-ca.crt -days 10000
    ```

2. 使用根证书签发证书

    生成所有的证书信息

    Default CN | Parent CA | O (in Subject) | kind
    ---|---|---|---
    kube-etcd | etcd-ca |  | server, client
    kube-etcd-peer | etcd-ca |  | server, client
    kube-etcd-healthcheck-client | etcd-ca |  | client
    kube-apiserver-etcd-client | etcd-ca | system:masters | client
    kube-apiserver | kubernetes-ca |  | server
    kube-apiserver-kubelet-client | kubernetes-ca | system:masters | client
    front-proxy-client | kubernetes-front-proxy-ca |  | client

    证书路径

    Default CN | recommend key path | recommended cert path | command | key argument | cert argument
    ---|---|---|---|---|---
    etcd-ca |  | etcd/ca.crt | kube-apiserver |  | --etcd-cafile
    etcd-client | apiserver-etcd-client.key | apiserver-etcd-client.crt | kube-apiserver | --etcd-keyfile | --etcd-cafile
    kubernetes-ca |  | ca.crt | kube-apiserver |  | --client-ca-file
    kube-apiserver | apiserver.key | apiserver.crt | kube-apiserver | --tls-private-key-file | --tls-cert-file
    apiserver-kubelet-client |  | apiserver-kubelet-client.crt | kube-apiserver |  | --kubelet-client-certificate
    front-proxy-ca |  | front-proxy-ca.crt | kube-apiserver |  | --requestheader-client-ca-file
    front-proxy-client | front-proxy-client.key | front-proxy-client.crt | kube-apiserver | --proxy-client-key-file | --proxy-client-cert-file
    etcd-ca |  | etcd/ca.crt | etcd |  | --trusted-ca-file, –peer-trusted-ca-file
    kube-etcd | etcd/server.key | etcd/server.crt | etcd | --key-file | --cert-file
    kube-etcd-peer | etcd/peer.key | etcd/peer.crt | etcd | --peer-key-file | --peer-cert-file
    etcd-ca |  | etcd/ca.crt | etcdctl |  | -cacert
    kube-etcd-healthcheck-client | etcd/healthcheck-client.key | etcd/healthcheck-client.crt | etcdctl | --key | --cert

    生成证书

    **apiserver-etcd-client**

    ```shell
    openssl genrsa -out apiserver-etcd-client.key 2048
    openssl req -new -key apiserver-etcd-client.key -subj "/CN=apiserver-etcd-client/O=system:masters" -out apiserver-etcd-client.csr
    openssl x509 -in apiserver-etcd-client.csr -req -CA etcd/ca.crt -CAkey etcd/ca.key -CAcreateserial -extensions v3_req_etcd -extfile openssl.cnf -out apiserver-etcd-client.crt -days 10000
    ```

    **kube-etcd**

    ```shell
    openssl genrsa -out etcd/server.key 2048
    openssl req -new -key etcd/server.key -subj "/CN=etcd-server" -out etcd/server.csr
    openssl x509 -in etcd/server.csr -req -CA etcd/ca.crt -CAkey etcd/ca.key -CAcreateserial -extensions v3_req_etcd -extfile openssl.cnf -out etcd/server.crt -days 10000
    ```

    **kube-etcd-peer**

    ```shell
    openssl genrsa -out etcd/peer.key 2048
    openssl req -new -key etcd/peer.key -subj "/CN=etcd-peer" -out etcd/peer.csr
    openssl x509 -in etcd/peer.csr -req -CA etcd/ca.crt -CAkey etcd/ca.key -CAcreateserial -extensions v3_req_etcd -extfile openssl.cnf -out etcd/peer.crt -days 10000
    ```

    **kube-etcd-healthcheck-client**

    ```shell
    openssl genrsa -out etcd/healthcheck-client.key 2048
    openssl req -new -key etcd/healthcheck-client.key -subj "/CN=etcd-client" -out etcd/healthcheck-client.csr
    openssl x509 -in etcd/healthcheck-client.csr -req -CA etcd/ca.crt -CAkey etcd/ca.key -CAcreateserial -extensions v3_req_etcd -extfile openssl.cnf -out etcd/healthcheck-client.crt -days 10000
    ```

    **kube-apiserver**

    ```shell
    openssl genrsa -out apiserver.key 2048
    openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -config openssl.cnf -out apiserver.csr
    openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_apiserver -extfile openssl.cnf -out apiserver.crt
    ```

    **apiserver-kubelet-client**

    ```shell
    openssl genrsa -out  apiserver-kubelet-client.key 2048
    openssl req -new -key apiserver-kubelet-client.key -subj "/CN=apiserver-kubelet-client/O=system:masters" -out apiserver-kubelet-client.csr
    openssl x509 -req -in apiserver-kubelet-client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out apiserver-kubelet-client.crt
    ```

    **front-proxy-client**

    ```shell
    openssl genrsa -out  front-proxy-client.key 2048
    openssl req -new -key front-proxy-client.key -subj "/CN=front-proxy-client" -out front-proxy-client.csr
    openssl x509 -req -in front-proxy-client.csr -CA front-proxy-ca.crt -CAkey front-proxy-ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out front-proxy-client.crt
    ```

    **kube-scheduler**

    ```shell
    openssl genrsa -out  kube-scheduler.key 2048
    openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
    openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out kube-scheduler.crt
    ```

    **sa.pub sa.key**

    ```shell
    openssl genrsa -out  sa.key 2048
    openssl ecparam -name secp521r1 -genkey -noout -out sa.key
    openssl ec -in sa.key -outform PEM -pubout -out sa.pub
    openssl req -new -sha256 -key sa.key -subj "/CN=system:kube-controller-manager" -out sa.csr
    openssl x509 -req -in sa.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out sa.crt
    ```

    **admin**

    ```shell
    openssl genrsa -out  admin.key 2048
    openssl req -new -key admin.key -subj "/CN=kubernetes-admin/O=system:masters" -out admin.csr
    openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out admin.crt
    ```

    **清理 csr srl**

    ```shell
    find . -name "*.csr" -o -name "*.srl"|xargs  rm -f
    ```

3. 利用证书生成组件的 kubeconfig

    filename | credential name | Default CN | O(In Subject)
    ---|---|---|---
    admin.kubeconfig | default-admin | kubernetes-admin | system:masters
    controller-manager.kubeconfig | default-controller-manager | system:kube-controller-manager |
    scheduler.kubeconfig | default-manager | system:kube-scheduler |

    **kube-controller-manager**

    ```  shell
    CLUSTER_NAME="kubernetes"
    KUBE_USER="system:kube-controller-manager"
    KUBE_CERT="sa"
    KUBE_CONFIG="controller-manager.kubeconfig"

    # 设置集群参数
    kubectl config set-cluster ${CLUSTER_NAME} \
      --certificate-authority=/etc/kubernetes/pki/ca.crt \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置客户端认证参数
    kubectl config set-credentials ${KUBE_USER} \
      --client-certificate=/etc/kubernetes/pki/${KUBE_CERT}.crt \
      --client-key=/etc/kubernetes/pki/${KUBE_CERT}.key \
      --embed-certs=true \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置上下文参数
    kubectl config set-context ${KUBE_USER}@${CLUSTER_NAME} \
      --cluster=${CLUSTER_NAME} \
      --user=${KUBE_USER} \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置当前使用的上下文
    kubectl config use-context ${KUBE_USER}@${CLUSTER_NAME} --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 查看生成的配置文件
    kubectl config view --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}
    ```

    **kube-scheduler**

    ```shell
    CLUSTER_NAME="kubernetes"
    KUBE_USER="system:kube-scheduler"
    KUBE_CERT="kube-scheduler"
    KUBE_CONFIG="scheduler.kubeconfig"

    # 设置集群参数
    kubectl config set-cluster ${CLUSTER_NAME} \
      --certificate-authority=/etc/kubernetes/pki/ca.crt \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置客户端认证参数
    kubectl config set-credentials ${KUBE_USER} \
      --client-certificate=/etc/kubernetes/pki/${KUBE_CERT}.crt \
      --client-key=/etc/kubernetes/pki/${KUBE_CERT}.key \
      --embed-certs=true \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置上下文参数
    kubectl config set-context ${KUBE_USER}@${CLUSTER_NAME} \
      --cluster=${CLUSTER_NAME} \
      --user=${KUBE_USER} \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置当前使用的上下文
    kubectl config use-context ${KUBE_USER}@${CLUSTER_NAME} --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 查看生成的配置文件
    kubectl config view --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}
    ```

    **admin(kubectl)**

    ```shell
    CLUSTER_NAME="kubernetes"
    KUBE_USER="kubernetes-admin"
    KUBE_CERT="admin"
    KUBE_CONFIG="admin.kubeconfig"

    # 设置集群参数
    kubectl config set-cluster ${CLUSTER_NAME} \
      --certificate-authority=/etc/kubernetes/pki/ca.crt \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置客户端认证参数
    kubectl config set-credentials ${KUBE_USER} \
      --client-certificate=/etc/kubernetes/pki/${KUBE_CERT}.crt \
      --client-key=/etc/kubernetes/pki/${KUBE_CERT}.key \
      --embed-certs=true \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置上下文参数
    kubectl config set-context ${KUBE_USER}@${CLUSTER_NAME} \
      --cluster=${CLUSTER_NAME} \
      --user=${KUBE_USER} \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置当前使用的上下文
    kubectl config use-context ${KUBE_USER}@${CLUSTER_NAME} --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 查看生成的配置文件
    kubectl config view --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}
    ```

#### 分发证书

分发到 kubeconfig 及证书其他 master 节点

```shell
for NODE in "${!otherMaster[@]}"; do
    echo "--- $NODE ${otherMaster[$NODE]} ---"
    scp -r /etc/kubernetes ${otherMaster[$NODE]}:/etc
done
```

### 安装 etcd

etcd 是用来保存集群所有状态的 Key/Value 存储系统,所有 Kubernetes 组件会通过 API Server 来跟 Etcd 进行沟通从而保存或读取资源状态。

在 k8s 集群中的 master 节点上部署 etcd 集群。

1. 在 k8s-m1 上下载 etcd 的发布包，并将二进制文件解压到 /usr/local/bin 目录下

    ```shell
    wget https://github.com/etcd-io/etcd/releases/download/${ETCD_version}/etcd-${ETCD_version}-linux-amd64.tar.gz
    tar -zxvf etcd-${ETCD_version}-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-${ETCD_version}-linux-amd64/etcd{,ctl}
    ```

2. 在 k8s-m1 上将 etcd 的二进制文件分发到其他 master 上

    ```shell
    for NODE in "${!otherMaster[@]}"; do
        echo "--- $NODE ${otherMaster[$NODE]} ---"
        scp /usr/local/bin/etcd* ${otherMaster[$NODE]}:/usr/local/bin/
    done
    ```

3. 在 k8s-m1 上修改 etcd 配置文件,注入 etcd 集群 ip 到 yml 文件中

    ```shell
    cd /home/k8s/k8s-manual-files/master/
    etcd_servers=$( xargs -n1<<<${MasterArray[@]} | sort | sed 's#^#https://#;s#$#:2379#;$s#\n##' | paste -d, -s - )
    etcd_initial_cluster=$( for i in ${!MasterArray[@]};do  echo $i=https://${MasterArray[$i]}:2380; done | sort | paste -d, -s - )
    sed -ri "/initial-cluster:/s#'.+'#'${etcd_initial_cluster}'#" etc/etcd/config.yml
    ```

    > 参考: <https://github.com/etcd-io/etcd/blob/master/etcd.conf.yml.sample>

4. 将相关文件分发到其他 master 上

    ```shell
    for NODE in "${!MasterArray[@]}"; do
        echo "--- $NODE ${MasterArray[$NODE]} ---"
        ssh ${MasterArray[$NODE]} "mkdir -p /etc/etcd /var/lib/etcd"
        scp systemd/etcd.service ${MasterArray[$NODE]}:/usr/lib/systemd/system/etcd.service
        scp etc/etcd/config.yml ${MasterArray[$NODE]}:/etc/etcd/etcd.config.yml
        ssh ${MasterArray[$NODE]} "sed -i "s/{HOSTNAME}/$NODE/g" /etc/etcd/etcd.config.yml"
        ssh ${MasterArray[$NODE]} "sed -i "s/{PUBLIC_IP}/${MasterArray[$NODE]}/g" /etc/etcd/etcd.config.yml"
        ssh ${MasterArray[$NODE]} 'systemctl daemon-reload'
    done
    ```

5. 在 k8s-m1 上启动所有 master 节点的 etcd  

    ```shell
    for NODE in "${!MasterArray[@]}"; do
        echo "--- $NODE ${MasterArray[$NODE]} ---"
        ssh ${MasterArray[$NODE]} 'systemctl enable --now etcd' &
    done
    wait
    ```

    输出到终端的时候多按几下回车直到等光标回到终端状态

    > etcd 进程首次启动时会等待其它节点的 etcd 加入集群，命令 systemctl start etcd 会卡住一段时间，为正常现象。  

6. k8s-m1上执行下面命令验证 ETCD 集群状态

    ```shell
    etcdctl \
      --cert-file /etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key-file /etc/kubernetes/pki/etcd/healthcheck-client.key \
      --ca-file /etc/kubernetes/pki/etcd/ca.crt \
      --endpoints $etcd_servers cluster-health
    ```

    使用 etcd v3版 API 查看当前 etcd 中存在的 key

    ```shell
    ETCDCTL_API=3 \
    etcdctl \
      --cert /etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key /etc/kubernetes/pki/etcd/healthcheck-client.key \
      --cacert /etc/kubernetes/pki/etcd/ca.crt \
      --endpoints $etcd_servers get / --prefix --keys-only
    ```

### Kubernetes Master

本部分将说明如何建立与设定 Kubernetes Master 角色，过程中会部署以下元件：

**kubelet:**  
负责管理容器的生命周期，定期从 API Server 获取节点上的预期状态（如网络、存储等等配置）资源，并让对应的容器插件（CRI、CNI 等）来达成这个状态。任何 Kubernetes 节点（node）都会拥有这个元件。  
关闭只读端口，在安全端口 10250 接收 https 请求，对请求进行认证和授权，拒绝匿名访问和非授权访问。  
使用 kubeconfig 访问 apiserver 的安全端口

**kube-apiserver:**  
以 REST APIs 提供 Kubernetes 资源的 CRUD，如授权、认证、存取控制与 API 注册等机制。  
关闭非安全端口，在安全端口 6443 接收 https 请求  
严格的认证和授权策略 （x509、token、RBAC）  
开启 bootstrap token 认证，支持 kubelet TLS bootstrapping  
使用 https 访问 kubelet、etcd，加密通信  

**kube-controller-manager:**  
通过核心控制循环（Core Control Loop）监听 Kubernetes API 的资源来维护集群的状态，这些资源会被不同的控制器所管理，如 Replication Controller、Namespace Controller 等等。而这些控制器会处理着自动扩展、滚动更新等等功能。  
关闭非安全端口，在安全端口 10252 接收 https 请求。  
使用 kubeconfig 访问 apiserver 的安全端口。  

**kube-scheduler:**  
负责将一个(或多个)容器依据调度策略分配到对应节点上让容器引擎(如 Docker)执行。而调度受到 QoS 要求、软硬性约束、亲和性(Affinity)等等因素影响。

**HAProxy:**
提供多个 API Server 的负载均衡(Load Balance),确保haproxy的端口负载到所有的apiserver的6443端口

**Keepalived:**
提供虚拟IP位址(VIP),来让vip落在可用的master主机上供所有组件都能访问到可用的master,结合haproxy能访问到master上的apiserver的6443端口

**kube-nginx**
使用 nginx 4 层透明代理功能实现 K8S 节点（ master 节点和 worker 节点）高可用访问 kube-apiserver 的策略。（等于 Haproxy + Keepalived）

**kube-nginx 方案和 haproxy+keepalived 方案选一个即可。**  
kube-nginx 方案需要在每一台主机上部署，而 haproxy+keepalived 方案只需要部署在 master 节点的主机上。  
kube-nginx 方案不需要额外的 IP ，而 haproxy+keepalived 方案需要额外的一个当前局域网未使用的 IP 做为 VIP。  
若使用 haproxy + keepalived 方案，则 master 节点的主机的网卡名需要相同。  

本文档使用 kube-nginx 方案，haproxy + keepalived 方案只做介绍。

#### 安装 haproxy + keepalived

在 k8s-m1 上执行如下命令，在 master 节点上安装 haproxy 和 keepalived

```shell
for NODE in "${!MasterArray[@]}"; do
    echo "--- $NODE ${MasterArray[$NODE]} ---"
    ssh ${MasterArray[$NODE]} 'yum install haproxy keepalived -y' &
done
wait
```

> 如果没输出的话，多按几次

在 k8s-m1 上把相关配置文件修改和分发

```shell
cd /home/k8s/k8s-manual-files/master/etc

# 修改haproxy.cfg配置文件
sed -i '$r '<(paste <( seq -f'  server k8s-api-%g'  ${#MasterArray[@]} ) <( xargs -n1<<<${MasterArray[@]} | sort | sed 's#$#:6443  check#')) haproxy/haproxy.cfg

# 修改keepalived(网卡和VIP写进去,使用下面命令)

sed -ri "s#\{\{ VIP \}\}#${VIP}#" keepalived/*
sed -ri "s#\{\{ interface \}\}#${interface}#" keepalived/keepalived.conf
sed -i '/unicast_peer/r '<(xargs -n1<<<${MasterArray[@]} | sort | sed 's#^#\t#') keepalived/keepalived.conf

# 分发文件
for NODE in "${!MasterArray[@]}"; do
    echo "--- $NODE ${MasterArray[$NODE]} ---"
    scp -r haproxy/ ${MasterArray[$NODE]}:/etc
    scp -r keepalived/ ${MasterArray[$NODE]}:/etc
    ssh ${MasterArray[$NODE]} 'systemctl enable --now haproxy keepalived'
done
```

等待四五秒后，ping 下 vip 看看是否能通

```shell
ping $VIP
```

如果 vip ping 不通，就是 keepalived 未启动或启动失败，在每个 master 节点上 restart 下 keepalived 或者确认下配置文件 /etc/keepalived/keepalived.conf 里网卡名和 ip 是否注入成功。

```shell
for NODE in "${!MasterArray[@]}"; do
    echo "--- $NODE ${MasterArray[$NODE]} ---"
    ssh ${MasterArray[$NODE]}  'systemctl restart haproxy keepalived'
done
```

#### 在所有节点安装 kube-nginx

**基于 nginx 代理的 kube-apiserver 高可用方案：**

- 控制节点（master 节点）的 kube-controller-manager、kube-scheduler 是多实例部署，所以只要有一个实例正常，就可以保证高可用；
- 集群内的 Pod 使用 K8S 服务域名 kubernetes 访问 kube-apiserver， kube-dns 会自动解析出多个 kube-apiserver 节点的 IP，所以也是高可用的；
- 在每个节点起一个 nginx 进程，后端对接多个 apiserver 实例，nginx 对它们做健康检查和负载均衡；
- kubelet、kube-proxy、controller-manager、scheduler 通过本地的 nginx（监听 127.0.0.1）访问 kube-apiserver，从而实现 kube-apiserver 的高可用；

**下载和编译 nginx ：**

1. 下载源码

    ```shell
    cd /home/k8s
    wget http://nginx.org/download/nginx-1.15.3.tar.gz
    tar -xzvf nginx-1.15.3.tar.gz
    ```

2. 配置编译参数

    ```shell
    cd /home/k8s/nginx-1.15.3
    mkdir nginx-prefix
    ./configure --with-stream --without-http --prefix=$(pwd)/nginx-prefix --without-http_uwsgi_module --without-http_scgi_module --without-http_fastcgi_module
    ```

    > `--with-stream`：开启 4 层透明转发(TCP Proxy)功能；  
    > `--without-xxx`：关闭所有其他功能，这样生成的动态链接二进制程序依赖最小；

3. 编译和安装

    ```shell
    cd /home/k8s/nginx-1.15.3
    make && make install
    ```

4. 验证编译的 nginx

    ```shell
    cd /home/k8s/nginx-1.15.3
    ./nginx-prefix/sbin/nginx -v
    ```

    输出：

    ```shell
    nginx version: nginx/1.15.3
    ```

    查看 nginx 动态链接的库：

    ```shell
    ldd ./nginx-prefix/sbin/nginx
    ```

    输出：

    ```shell
    linux-vdso.so.1 =>  (0x00007ffc945e7000)
    libdl.so.2 => /lib64/libdl.so.2 (0x00007f4385072000)
    libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f4384e56000)
    libc.so.6 => /lib64/libc.so.6 (0x00007f4384a89000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f4385276000)
    ```

    > 由于只开启了 4 层透明转发功能，所以除了依赖 libc 等操作系统核心 lib 库外，没有对其它 lib 的依赖(如 libz、libssl 等)，这样可以方便部署到各版本操作系统中；

**安装和部署 nginx ：**

1. 创建目录结构并拷贝二进制程序：

    ```shell
    cd /home/k8s
    source env.sh
    for NODE in "${!AllNode[@]}"; do
        echo "--- $NODE ${AllNode[$NODE]} ---"
        mkdir -p /usr/local/kube-nginx/{conf,logs,sbin}
        scp /home/k8s/nginx-1.15.3/nginx-prefix/sbin/nginx  ${AllNode[$NODE]}:/usr/local/kube-nginx/sbin/kube-nginx
        ssh ${AllNode[$NODE]} "chmod a+x /opt/k8s/kube-nginx/sbin/*"
    done
    ```

    > 重命名二进制文件为 kube-nginx；

2. 配置 nginx，开启 4 层透明转发功能：

    ```shell
    cd /home/k8s
    cat > kube-nginx.conf << \EOF
    worker_processes 1;

    events {
        worker_connections  1024;
    }

    stream {
        upstream backend {
            hash $remote_addr consistent;
            server 192.168.15.5:6443        max_fails=3 fail_timeout=30s;
            server 192.168.15.6:6443        max_fails=3 fail_timeout=30s;
            server 192.168.15.7:6443        max_fails=3 fail_timeout=30s;
        }

        server {
            listen 127.0.0.1:8443;
            proxy_connect_timeout 1s;
            proxy_pass backend;
        }
    }
    EOF
    ```

    > 需要根据集群 kube-apiserver 的实际情况，替换 backend 中 server 列表；

3. 分发配置文件

    ```shell
    cd /home/k8s
    source env.sh
    for NODE in "${!AllNode[@]}"; do
        echo "--- $NODE ${AllNode[$NODE]} ---"
        scp kube-nginx.conf  ${AllNode[$NODE]}:/usr/local/kube-nginx/conf/kube-nginx.conf
    done
    ```

**配置 systemd unit 文件，启动服务：**

配置 kube-nginx systemd unit 文件:

```shell
cd /home/k8s
cat > kube-nginx.service <<EOF
[Unit]
Description=kube-apiserver nginx proxy
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
ExecStartPre=/usr/local/kube-nginx/sbin/kube-nginx -c /usr/local/kube-nginx/conf/kube-nginx.conf -p /usr/local/kube-nginx -t
ExecStart=/usr/local/kube-nginx/sbin/kube-nginx -c /usr/local/kube-nginx/conf/kube-nginx.conf -p /usr/local/kube-nginx
ExecReload=/usr/local/kube-nginx/sbin/kube-nginx -c /usr/local/kube-nginx/conf/kube-nginx.conf -p /usr/local/kube-nginx -s reload
PrivateTmp=true
Restart=always
RestartSec=5
StartLimitInterval=0
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

分发 systemd unit 文件：

```shell
cd /home/k8s
source env.sh
for NODE in "${!AllNode[@]}"; do
    echo "--- $NODE ${AllNode[$NODE]} ---"
    scp kube-nginx.service  ${AllNode[$NODE]}:/usr/lib/systemd/system/
done
```

启动 kube-nginx 服务：

```shell
cd /home/k8s
source env.sh
for NODE in "${!AllNode[@]}"; do
    echo "--- $NODE ${AllNode[$NODE]} ---"
    ssh ${AllNode[$NODE]} "systemctl daemon-reload && systemctl enable kube-nginx && systemctl restart kube-nginx"
done
```

**检查 kube-nginx 服务运行状态：**

```shell
cd /home/k8s
source env.sh
for NODE in "${!AllNode[@]}"; do
    echo "--- $NODE ${AllNode[$NODE]} ---"
    ssh ${AllNode[$NODE]} "systemctl status kube-nginx |grep 'Active:'"
done
```

确保状态为 active (running)，否则查看日志，确认原因：

```shell
journalctl -u kube-nginx
```

#### Master 组件

1. 在k8s-1节点把相关配置文件修改后再分发

    ```shell
    cd /home/k8s/k8s-manual-files/master/
    etcd_servers=$( xargs -n1<<<${MasterArray[@]} | sort | sed 's#^#https://#;s#$#:2379#;$s#\n##' | paste -d, -s - )

    # 注入VIP和etcd_servers,apiserver数量
    sed -ri '/--etcd-servers/s#=.+#='"$etcd_servers"' \\#' systemd/kube-apiserver.service
    sed -ri '/apiserver-count/s#=[^\]+#='"${#MasterArray[@]}"' #' systemd/kube-apiserver.service

    # 分发文件
    for NODE in "${!MasterArray[@]}"; do
        echo "--- $NODE ${MasterArray[$NODE]} ---"
        ssh ${MasterArray[$NODE]} 'mkdir -p /etc/kubernetes/manifests /var/lib/kubelet /var/log/kubernetes'
        scp systemd/kube-*.service ${MasterArray[$NODE]}:/usr/lib/systemd/system/
        #注入网卡ip
        ssh ${MasterArray[$NODE]} "sed -ri '/bind-address/s#=[^\]+#=${MasterArray[$NODE]} #' /usr/lib/systemd/system/kube-apiserver.service && sed -ri '/--advertise-address/s#=[^\]+#=${MasterArray[$NODE]} #' /usr/lib/systemd/system/kube-apiserver.service"

    done
    ```

2. 在 k8s-1 上给所有 master 机器启动 kube-apiserver、kube-controller-manager、kube-scheduler 和 kubelet 服务并设置 kubectl 补全脚本:

    ```shell
    for NODE in "${!MasterArray[@]}"; do
        echo "--- $NODE ${MasterArray[$NODE]} ---"
        ssh ${MasterArray[$NODE]} 'systemctl enable --now  kube-apiserver kube-controller-manager kube-scheduler;
        mkdir -p ~/.kube/
        cp /etc/kubernetes/admin.kubeconfig ~/.kube/config;
        kubectl completion bash > /etc/bash_completion.d/kubectl'
    done
    ```

    > 注意： k8s v1.14.0版本已经将admission-plugins: Initializers移除。  
    > 详见：<https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.14.md#v1140>

3. 验证组件
    在任意一台 master 节点通过简单指令验证：

    ```shell
    $ kubectl get cs
    NAME                 STATUS    MESSAGE              ERROR
    scheduler            Healthy   ok
    controller-manager   Healthy   ok
    etcd-2               Healthy   {"health": "true"}
    etcd-0               Healthy   {"health": "true"}
    etcd-1               Healthy   {"health": "true"}

    $ kubectl get svc
    NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   36s
    ```

#### 配置 bootstrap

由于本次安装启用了 TLS 认证，因此每个节点的 kubelet 都必须使用 kube-apiserver 的 CA 的凭证后，才能与 kube-apiserver 进行沟通，而该过程需要手动针对每台节点单独签署凭证是一件繁琐的事情，且一旦节点增加会延伸出管理不易问题；而 TLS bootstrapping 目标就是解决该问题，通过让 kubelet 先使用一个预定低权限使用者连接到 kube-apiserver，然后在对 kube-apiserver 申请凭证签署，当授权 Token 一致时，Node 节点的 kubelet 凭证将由 kube-apiserver 动态签署提供。具体作法可以参考 [TLS Bootstrapping](https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping/) 与 [Authenticating with Bootstrap Tokens](https://kubernetes.io/docs/admin/bootstrap-tokens/)。

**以下步骤在一个 master 节点上执行即可：**

1. 首先在 k8s-m1 建立一个变数来产生 BOOTSTRAP_TOKEN ，并建立 bootstrap 的 kubeconfig 文件，接着在 k8s-m1 建立 TLS bootstrap secret 来提供自动签证使用：

    ```bash
    TOKEN_PUB=$(openssl rand -hex 3)
    TOKEN_SECRET=$(openssl rand -hex 8)
    BOOTSTRAP_TOKEN="${TOKEN_PUB}.${TOKEN_SECRET}"

    kubectl -n kube-system create secret generic bootstrap-token-${TOKEN_PUB} \
            --type 'bootstrap.kubernetes.io/token' \
            --from-literal description="cluster bootstrap token" \
            --from-literal token-id=${TOKEN_PUB} \
            --from-literal token-secret=${TOKEN_SECRET} \
            --from-literal usage-bootstrap-authentication=true \
            --from-literal usage-bootstrap-signing=true
    ```

2. 建立 bootstrap 的 kubeconfig 文件

    ```bash
    CLUSTER_NAME="kubernetes"
    KUBE_USER="kubelet-bootstrap"
    KUBE_CONFIG="bootstrap.kubeconfig"

    # 设置集群参数
    kubectl config set-cluster ${CLUSTER_NAME} \
      --certificate-authority=/etc/kubernetes/pki/ca.crt \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置上下文参数
    kubectl config set-context ${KUBE_USER}@${CLUSTER_NAME} \
      --cluster=${CLUSTER_NAME} \
      --user=${KUBE_USER} \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置客户端认证参数
    kubectl config set-credentials ${KUBE_USER} \
      --token=${BOOTSTRAP_TOKEN} \
      --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 设置当前使用的上下文
    kubectl config use-context ${KUBE_USER}@${CLUSTER_NAME} --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

    # 查看生成的配置文件
    kubectl config view --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}
    ```

    > - 若想要用手动签署凭证来进行授权的话，可以参考 [Certificate](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)。  
    > - 向 kubeconfig 写入的是 token，bootstrap 结束后 kube-controller-manager 为 kubelet 创建 client 和 server 证书；  
    > - token 有效期为 1 天，超期后将不能再被用来 boostrap kubelet，且会被 kube-controller-manager 的 tokencleaner 清理；  
    > - kube-apiserver 接收 kubelet 的 bootstrap token 后，将请求的 user 设置为 system:bootstrap:\<Token ID>，group 设置为 system:bootstrappers，后续将为这个 group 设置 ClusterRoleBinding；

3. 授权 kubelet 可以创建 csr

    ```shell
    kubectl create clusterrolebinding kubeadm:kubelet-bootstrap \
            --clusterrole system:node-bootstrapper --group system:bootstrappers
    ```

4. 批准 csr 请求，允许 system:bootstrappers 组的所有 csr

    ```shell
    cat <<EOF | kubectl apply -f -
    # Approve all CSRs for the group "system:bootstrappers"
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: auto-approve-csrs-for-group
    subjects:
    - kind: Group
      name: system:bootstrappers
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
      apiGroup: rbac.authorization.k8s.io
    EOF
    ```

5. 允许 kubelet 能够更新自己的证书

    ```shell
    cat <<EOF | kubectl apply -f -
    # Approve renewal CSRs for the group "system:nodes"
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: auto-approve-renewals-for-nodes
    subjects:
    - kind: Group
      name: system:nodes
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
      apiGroup: rbac.authorization.k8s.io
    EOF
    ```

### All Kubernetes Nodes

本部分将说明如何建立与设定 Kubernetes Node 角色，Node 是主要执行容器实例（Pod）的工作节点。  
在开始部署前，先在 k8-m1 将需要用到的文件复制到所有其他节点上。  
kubelet的配置选项官方建议大多数的参数写一个 yaml 里用 --config 去指定，详见： <https://godoc.org/k8s.io/kubernetes/pkg/kubelet/apis/config#KubeletConfiguration>

```shell
for NODE in "${!Other[@]}"; do
    echo "--- $NODE ${Other[$NODE]} ---"
    ssh ${Other[$NODE]} "mkdir -p /etc/kubernetes/pki /etc/kubernetes/manifests /var/lib/kubelet/"
    for FILE in /etc/kubernetes/pki/ca.crt /etc/kubernetes/bootstrap.kubeconfig; do
      scp ${FILE} ${Other[$NODE]}:${FILE}
    done
done
```

在 k8s-m1 节点分发 kubelet.service 文件和其他配置文件到每台主机上管理kubelet：

```shell
cd /home/k8s/k8s-manual-files/
for NODE in "${!AllNode[@]}"; do
    echo "--- $NODE ${AllNode[$NODE]} ---"
    scp master/systemd/kubelet.service ${AllNode[$NODE]}:/lib/systemd/system/kubelet.service
    scp master/etc/kubelet/kubelet-conf.yml ${AllNode[$NODE]}:/etc/kubernetes/kubelet-conf.yml
    ssh ${AllNode[$NODE]} "sed -ri '/0.0.0.0/s#\S+\$#${MasterArray[$NODE]}#' /etc/kubernetes/kubelet-conf.yml"
    ssh ${AllNode[$NODE]} "sed -ri '/127.0.0.1/s#\S+\$#${MasterArray[$NODE]}#' /etc/kubernetes/kubelet-conf.yml"
done
```

在 k8s-m1 上去启动每个 node 节点的 kubelet 服务:

```shell
for NODE in "${!AllNode[@]}"; do
    echo "--- $NODE ${AllNode[$NODE]} ---"
    ssh ${AllNode[$NODE]} 'systemctl enable --now kubelet.service'
done
```

在任意一台master节点并通过简单指令验证集群状态：

```shell
[root@k8s-m1 k8s-manual-files]# kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
c48-1    Ready    <none>   19s   v1.15.0
c48-2    Ready    <none>   20s   v1.15.0
k8s-m1   Ready    <none>   32s   v1.15.0
k8s-m2   Ready    <none>   31s   v1.15.0
k8s-m3   Ready    <none>   31s   v1.15.0
[root@k8s-m1 k8s-manual-files]# kubectl get csr
NAME        AGE   REQUESTOR                 CONDITION
csr-cwsqq   70s   system:bootstrap:e006fa   Approved,Issued
csr-dqw9s   86s   system:bootstrap:e006fa   Approved,Issued
csr-lchwz   69s   system:bootstrap:e006fa   Approved,Issued
csr-pg4tq   86s   system:bootstrap:e006fa   Approved,Issued
csr-s2ts5   86s   system:bootstrap:e006fa   Approved,Issued
```

> - kubelet 中的 --allow-privileged 参数在 1.15.0 版本已被废弃，详见：<https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.15.md#v1154>
> - 若不想让（没有声明容忍该污点的）pod 跑在 master 上，则执行如下命令给 master 节点加上污点 traint
>
```shell
kubectl taint nodes ${!MasterArray[@]} node-role.kubernetes.io/master="":NoSchedule
```
>
> - 容忍与污点详见：<https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/>

为 node 打上标签，标记其为 master 或 worker （master 节点也作为 worker节点）

```shell
kubectl label node ${!MasterArray[@]} node-role.kubernetes.io/master=""
kubectl label node ${!NodeArray[@]} node-role.kubernetes.io/worker=worker
```

查看当前集群 node 的状态

```shell
[root@k8s-m1 k8s-manual-files]# kubectl get nodes
NAME     STATUS   ROLES           AGE     VERSION
c48-1    Ready    worker          2m47s   v1.15.0
c48-2    Ready    worker          2m48s   v1.15.0
k8s-m1   Ready    master,worker   3m      v1.15.0
k8s-m2   Ready    master,worker   2m59s   v1.15.0
k8s-m3   Ready    master,worker   2m59s   v1.15.0
```

kubelet 的数据存储目录默认为 /var/lib/kubelet，为了避免 /var 目录所在的分区空间不足，可以通过 --root-dir 参数（使用 kubelet --help 查看）修改 kubelet 的默认数据存储目录，但是由于 rook-ceph 等插件默认读取的是 /var/lib/kubelet 目录，所以方便起见，本文使用软链接的方式修改 kubelet 的存储目录。

```shell
systemctl stop kubelet
rm -rf /var/lib/kubelet
mkdir -p /home/data/kubelet
ln -s /home/data/kubelet /var/lib/kubelet
```

上述命令将 /home/data/kubelet 软链接到 /var/lib/kubelet。  
若删除 /var/lib/kubelet 时报错，提示挂载问题，可以手动使用 umount 命令卸载正在挂载的目录。

## Kubernetes Core Addons

### kube-proxy

Kube-proxy 是实现 Service 的关键插件，kube-proxy 会在运行在每台 worker 节点，通过监听 API Server 的 Service 与 Endpoint 资源物件的改变，来依据变化执行 iptables 来实现网路的转发。

本文档使用 ipvs 模式部署 kube-proxy。提供了二进制部署方式和 daemonset 部署方式。建议使用 daemonset 方式进行部署。

#### 二进制方式部署 kube-proxy

在 k8s-m1 上新建一个 kube-proxy 的 serviceaccount：

```shell
kubectl -n kube-system create serviceaccount kube-proxy
```

将 kube-proxy 的 serviceaccount 绑定到 clusterrole system:node-proxier 以运行 RBAC ：

```shell
kubectl create clusterrolebinding kubeadm:kube-proxy \
        --clusterrole system:node-proxier \
        --serviceaccount kube-system:kube-proxy
```

创建 kube-proxy 的 kubeconfig

```shell
CLUSTER_NAME="kubernetes"
KUBE_CONFIG="kube-proxy.kubeconfig"

SECRET=$(kubectl -n kube-system get sa/kube-proxy \
    --output=jsonpath='{.secrets[0].name}')

JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET \
    --output=jsonpath='{.data.token}' | base64 -d)

kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

kubectl config set-context ${CLUSTER_NAME} \
  --cluster=${CLUSTER_NAME} \
  --user=${CLUSTER_NAME} \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

kubectl config set-credentials ${CLUSTER_NAME} \
  --token=${JWT_TOKEN} \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

kubectl config use-context ${CLUSTER_NAME} --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}
kubectl config view --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}
```

在 k8s-m1 上将相关文件分发到所有 worker 节点

```shell
cd /home/k8s/k8s-manual-files/
for NODE in "${!Other[@]}"; do
    echo "--- $NODE ${Other[$NODE]} ---"
    scp /etc/kubernetes/kube-proxy.kubeconfig ${Other[$NODE]}:/etc/kubernetes/kube-proxy.kubeconfig
done

for NODE in "${!AllNode[@]}"; do
    echo "--- $NODE ${AllNode[$NODE]} ---"
    scp addons/kube-proxy/kube-proxy.conf ${AllNode[$NODE]}:/etc/kubernetes/kube-proxy.conf
    scp addons/kube-proxy/kube-proxy.service ${AllNode[$NODE]}:/usr/lib/systemd/system/kube-proxy.service
    ssh ${AllNode[$NODE]} "sed -ri '/0.0.0.0/s#\S+\$#${MasterArray[$NODE]}#' /etc/kubernetes/kube-proxy.conf"
done
```

在 k8s-m1 上启动所有 worker 节点的 kube-proxy 服务

```shell
for NODE in "${!AllNode[@]}"; do
    echo "--- $NODE ${AllNode[$NODE]} ---"
    ssh ${AllNode[$NODE]} 'systemctl enable --now kube-proxy'
done
```

#### daemonset 方式部署 kube-proxy

在 k8s-m1 上执行如下命令：

```shell
cd /home/k8s/k8s-manual-files
source /home/k8s/env.sh
# 注入变量
sed -ri "/server:/s#(: ).+#\1${KUBE_APISERVER}#" addons/kube-proxy/kube-proxy.yml
sed -ri "/image:.+kube-proxy/s#:[^:]+\$#:$KUBE_VERSION#" addons/kube-proxy/kube-proxy.yml
kubectl apply -f addons/kube-proxy/kube-proxy.yml
```

输出内容如下：

```shell
serviceaccount "kube-proxy" created
clusterrolebinding.rbac.authorization.k8s.io "system:kube-proxy" created
configmap "kube-proxy" created
daemonset.apps "kube-proxy" created
```

查看当前 kube-proxy pod 的状态

```shell
[root@k8s-m1 k8s-manual-files]# kubectl -n kube-system get po -l k8s-app=kube-proxy -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
kube-proxy-52hkz   1/1     Running   0          54s   192.168.15.161   c48-1    <none>           <none>
kube-proxy-54f8d   1/1     Running   0          54s   192.168.15.143   c48-2    <none>           <none>
kube-proxy-757j7   1/1     Running   0          54s   192.168.15.6     k8s-m2   <none>           <none>
kube-proxy-c2ht9   1/1     Running   0          54s   192.168.15.7     k8s-m3   <none>           <none>
kube-proxy-fdx48   1/1     Running   0          54s   192.168.15.5     k8s-m1   <none>           <none>
```

通过 ipvsadm 查看当前 proxy 的规则

```shell
[root@k8s-m1 k8s-manual-files]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 192.168.15.5:6443            Masq    1      0          0
  -> 192.168.15.6:6443            Masq    1      0          0
  -> 192.168.15.7:6443            Masq    1      0          0
```

确认使用的是 ipvs 模式

```shell
[root@k8s-m1 k8s-manual-files]# curl localhost:10249/proxyMode
ipvs
```

### 网络插件 flannel 或 calico

Kubernetes 在默认情況下与 Docker 的网络有所不同。在 Kubernetes 中有四个问题是需要被解決的,分別为：

- 高耦合的容器到容器通信：通过 Pods 内 localhost 的來解決。  
- Pod 到 Pod 的通信：通过实现网络模型来解决。
- Pod 到 Service 通信：由 Service objects 结合 kube-proxy 解決。
- 外部到 Service 通信：一样由 Service objects 结合 kube-proxy 解決。

而 Kubernetes 对于任何网络的实现都需要满足以下基本要求(除非是有意调整的网络分段策略)：

- 所有容器能够在沒有 NAT 的情況下与其他容器通信。
- 所有节点能够在沒有 NAT 情況下与所有容器通信(反之亦然)。
- 容器看到的 IP 与其他人看到的 IP 是一样的。

庆幸的是 Kubernetes 已经有非常多种的[网络模型](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)作为[网络插件（Network Plugins）](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)方式被实现，因此可以选用满足自己需求的网络功能来使用。另外 Kubernetes 中的网络插件有以下两种形式：

- CNI plugins：以 appc/CNI 标准规范所实现的网络，详细可以阅读 [CNI Specification](https://github.com/containernetworking/cni/blob/master/SPEC.md)。
- Kubenet plugin：使用 CNI plugins 的 bridge 与 host-local 来实现基本的 cbr0。这通常被用在公有云服务上的 Kubernetes 集群网络。

如果想了解如何选择可以如阅读 Chris Love 的 [Choosing a CNI Network Provider for Kubernetes](https://chrislovecnm.com/kubernetes/cni/choosing-a-cni-provider/) 文章。

#### flannel

flannel 使用 vxlan 技术为各节点创建一个可以互通的 Pod 网络，使用的端口为 UDP 8472，需要开放该端口（如公有云 AWS 等）。

flannel 第一次启动时，从 etcd 获取 Pod 网段信息，为本节点分配一个未使用的 /24 段地址，然后创建 flannel.1（也可能是其它名称，如 flannel1 等） 接口。

查看使用的 flannel 容器

```shell
$ grep -Pom1 'image:\s+\K\S+' addons/flannel/kube-flannel.yml
quay.io/coreos/flannel:v0.11.0-amd64
```

使用 daemonset 创建 flannel，yaml 来源于[官方](https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml)，删除了非  amd64 的 daemonset

将环境变量中的 interface 配置到 yml 中，然后安装 flannel

```shell
sed -ri "s#\{\{ interface \}\}#${interface}#" addons/flannel/kube-flannel.yml
kubectl apply -f addons/flannel/kube-flannel.yml
```

检查 flannel 启动是否成功

```shell
$ kubectl -n kube-system get po -l k8s-app=flannel
NAME                READY     STATUS    RESTARTS   AGE
kube-flannel-ds-27jwl   2/2       Running   0          59s
kube-flannel-ds-4fgv6   2/2       Running   0          59s
kube-flannel-ds-mvrt7   2/2       Running   0          59s
kube-flannel-ds-p2q9g   2/2       Running   0          59s
kube-flannel-ds-zchsz   2/2       Running   0          59s
```

#### calico

Calico 是一款纯 Layer 3 的网络，其好处是它整合了各种云原生平台（Docker、Mesos 与 OpenStack 等），且 Calico 不采用 vSwitch，而是在每个 Kubernetes 节点使用 vRouter 功能，并通过 Linux Kernel 既有的 L3 forwarding 功能，而当资料中心复杂度增加时，Calico 也可以利用 BGP route reflector 来达成。

> - 想了解 Calico 与传统 overlay networks 的差异，可以阅读 [Difficulties with traditional overlay networks](https://www.projectcalico.org/learn/) 文章
> - [calico 官方安装指南](https://docs.projectcalico.org/latest/getting-started/kubernetes/installation/)

calico 官方关于使用 calico  提供策略和网络的安装方式（Installing Calico for policy and networking）有三种，根据数据存储类型（datastore type）和节点数量（number of nodes）进行划分：

- [使用 kubernetes api 存储数据 且节点数量少于50](https://docs.projectcalico.org/v3.9/getting-started/kubernetes/installation/calico#installing-with-the-kubernetes-api-datastore50-nodes-or-less)
- [使用 kubernetes api 存储数据，且节点数量大于50](https://docs.projectcalico.org/v3.9/getting-started/kubernetes/installation/calico#installing-with-the-kubernetes-api-datastoremore-than-50-nodes)
- [使用 etcd 存储数据](https://docs.projectcalico.org/v3.9/getting-started/kubernetes/installation/calico#installing-with-the-etcd-datastore)

本文使用 etcd 作为 calico 后端存储数据。

1. 下载 calico 适用于 etcd 后端存储的清单文件

    ```shell
    curl https://docs.projectcalico.org/v3.9/manifests/calico-etcd.yaml -o calico-etcd.yaml
    ```

2. 如果你使用的 pod 的 CIDR 为 `10.244.0.0/16` ，则跳过该步骤。如果你使用的 pod 的 CIDR 不同，使用如下命令设置一个名为 `POD_CIDR` 的环境变量，将清单文件中的 `10.244.0.0/16` 替换为你的 pod 的 CIDR。

    ```shell
    POD_CIDR="<your-pod-cidr>"
    sed -i -e "s?10.244.0.0/16?$POD_CIDR?g" calico-etcd.yaml
    ```

    > 注意：实际的清单文件中，pod cidr 为 192.168.0.0/16

    因为本文中 pod 的 CIDR 为 172.30.0.0/16，所以需要替换下。

    ```shell
    POD_CIDR="172.30.0.0/16"
    sed -i -e "s?192.168.0.0/16?$POD_CIDR?g" calico-etcd.yaml
    ```

3. 在清单文件（calico-etcd.yaml）中的名为 calico-config 的 ConfigMap中，设置你的 etcd 服务的 ip 地址和端口，可以通过逗号（英文逗号）作为分隔符指定多个 etcd 服务。

    ```shell
    source /home/k8s/env.sh
    etcd_servers=$( xargs -n1<<<${MasterArray[@]} | sort | sed 's#^#https://#;s#$#:2379#;$s#\n##' | paste -d, -s - )
    sed -ri "s#http:\/\/<ETCD_IP>:<ETCD_PORT>#${etcd_servers}#" calico-etcd.yaml
    ```

    修改后如下所示

    ```shell
    # Source: calico/templates/calico-config.yaml
    # This ConfigMap is used to configure a self-hosted Calico installation.
    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: calico-config
      namespace: kube-system
    data:
      # Configure this with the location of your etcd cluster.
      etcd_endpoints: "https://192.168.15.5:2379,https://192.168.15.6:2379,https://192.168.15.7:2379"
    ```

4. 因为本文的 etcd 服务使用的是 https，所以需要在清单文件中配置下证书，这样 calico 才可以访问 etcd。

    ```shell
    etcd_key=$(cat /etc/kubernetes/pki/etcd/server.key | base64 | tr -d '\n')
    etcd_cert=$(cat /etc/kubernetes/pki/etcd/server.crt | base64 | tr -d '\n')
    etcd_ca=$(cat /etc/kubernetes/pki/etcd/ca.crt | base64 | tr -d '\n')
    sed -i "s/# etcd-key: null/etcd-key: ${etcd_key}/" calico-etcd.yaml
    sed -i "s/# etcd-cert: null/etcd-cert: ${etcd_cert}/" calico-etcd.yaml
    sed -i "s/# etcd-ca: null/etcd-ca: ${etcd_ca}/" calico-etcd.yaml
    sed -i 's?etcd_ca: ""?etcd_ca: "/calico-secrets/etcd-ca"?' calico-etcd.yaml
    sed -i 's?etcd_cert: ""?etcd_cert: "/calico-secrets/etcd-cert"?' calico-etcd.yaml
    sed -i 's?etcd_key: ""?etcd_key: "/calico-secrets/etcd-key"?' calico-etcd.yaml
    ```

5. 若主机存在多个网卡设备，calico 可能绑定错误的网卡，需要在清单文件中添加 `IP_AUTODETECTION_METHOD` 环境变量，因为本文没有使用 ipv6 所以没有配置 `IP6_AUTODETECTION_METHOD`。

    > calico 官网环境变量参考：<https://docs.projectcalico.org/v3.9/reference/node/configuration#environment-variables>  
    > 自动检测 ip 机制参考：<https://docs.projectcalico.org/v3.9/reference/node/configuration#ip-autodetection-methods>

    **IP_AUTODETECTION_METHOD**: The method to use to autodetect the IPv4 address for this host. This is only used when the IPv4 address is being autodetected. See [IP Autodetection methods](https://docs.projectcalico.org/v3.9/reference/node/configuration#ip-autodetection-methods) for details of the valid methods. [Default: `first-found`]

    **IP6_AUTODETECTION_METHOD**: The method to use to autodetect the IPv6 address for this host. This is only used when the IPv6 address is being autodetected. See [IP Autodetection methods](https://docs.projectcalico.org/v3.9/reference/node/configuration#ip-autodetection-methods) for details of the valid methods. [Default: `first-found`]

    因为本文的 k8s 集群中不同的主机网卡名称不一样，k8s-m1、k8s-m2、k8s-m3 中每台主机存在4个网卡，网卡名为eno1、eno2、eno3、eno4。而 c48-1、c48-2 中每台主机存在2个网卡，网卡名为 eth0、eth1。所以本文使用了 [IP Autodetection methods](https://docs.projectcalico.org/v3.9/reference/node/configuration#ip-autodetection-methods) 中的 `can-reach=DESTINATION` 方法，绑定能抵达 192.168.15.5 （k8s-m1 的 eno1） 的网卡。

    ```shell
    sed -ri '/"autodetect"/a\            - name: IP_AUTODETECTION_METHOD\n              value: "can-reach=192.168.15.5"' calico-etcd.yaml
    ```

6. 修改清单文件中容器的时区，使用宿主机的时区（可选）。（容器中默认的时间与北京时间相差8个小时）

    修改后对比于原文的差异：

    ```shell
    [root@k8s-m1 k8s]# diff calico.yaml calico-etcd.yaml
    17,19c17,19
    <   # etcd-key: null
    <   # etcd-cert: null
    <   # etcd-ca: null
    ---
    >   etcd-key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBNGVnVUJwTFB1akpRaXJFOEFkU1psTmhjb1R1K0N5ZWJtUDE0ZXYyRmpYN1BVRnhNCkc4TEV0N2J1NEd1c3NNcFcwN1NtSTQvOFNSL0ZkNkFXdjYwMTFLTmNmWEJScFJFVENhRUdoU2UvVFJtclVzNVcKR1VtNE9jb2N6UUY0bDkrejIxSjF2bk5zR0xINkgvQXpNN3hQL0RUMzZKaDVvR0o4eGg2Wk1wWm8rTTVBSzBxdgppSUhRNXZSV0U3YVIzU1djdFVKQUJDZHhvUWZqMi9ta3ZzZ1kxUnRINVp0aTNMTExjTkFMKzZXRHFXQy9ZY0JDCjVZd0dPT0Q2M0pINk5yY1J6OXdEZGJsaFhaL3RnN3Y3a1Rid3V3NVg5TmFwUFRtbHdlQ1MvU084UE9lLzk3NlAKeTFlOExLa1pSRHJkakJqZ3RzZUI3b2txT0RjdVlLa1dMNFNIQVFJREFRQUJBb0lCQURGL3dLT1FGNlFjMGprUgpqS3g2QVF6ME81ZTRsM09xUWhYTHRGSitxbnpPaEc1L1NzM3FaMkE1M1MyZmFqOXlsb1BjMldxQmFpNDduL0VPClN1M0U3ajZoYk9xdmFiUlpnV3BpdGlNSENvdkNUQi9neGt6VU1tRzNQNGhNQWppRTg4dml6Wm5sZ0pJSXJWM0MKSy9YeUZUU1dCcHdZak0zdnhwZENyUjdBaGsrOXVDWjJjcXFicUhuQmxINnpjNlVVMXlac0ZSN1ZmamJjMG9jdgpEeWU4S3RabzA5STdab1VHbSs2eENVM05JUll5dkVpK3JxOXZBUm5kSGlPbFd5MVR4d3c0ZTB3ZVNkNXM2ODVTCmtIRXoxM1BXT2VvVjFtempDRzhWTnZMUkF3VUttQ1BZdFMwa3R1ZWVGam5RRWlEaFBuNmxmTXBmU1BMb2tadHYKK3hNMXpJRUNnWUVBK3Q1TEM4VHYzUXl0T1NqUDY3TVlkdERBbzY2am9XamJNaTV1YmlOUE1LZDJqWk9PRm8rTgpSeVpORXJlMm1rc09zVXVkT0RyRjFrY283eVYwR3RlMkY5OS9IYUwwNkcyeGdNMHkvV0NPZTh4cUx2NTR5ejZsCmdaY2dzcnhLNnZMbWkrdndkUlZOSG9uZFh1cGFYQ0QxTkx6ckp4UWVTYUpFQ040enY0ZXUrVTBDZ1lFQTVvY1IKdTJ0Mk9zZXpaRGpGaFJFNkk5WjQyUW1ZWnl5MkFzR2Y3QzZZWVRjaU1UV3ZUSlE0d004elpJajlEUGEySVA0Zgp4WWZmdE5sYzZaMnJGZkkwWWpGbnZzNTE3UEhPbExzZ3dzc3VGQm1jV3lpU29Ib3o2L0FJWEhrdWtkdk5vQzI3CnNEYW1TR3BmWUZ4VTlGb2JqYjRzMzMya2xCcHFTNFJ0UEZ0Z0NvVUNnWUFnTkRzVUJyTDRBSEdZUGRuN0d1R1EKRnhvenFPNk9nT1JxbTdWSFpEYjlPdklvR0lJTCtWK2NlNWszUnVnbEJHK2RhT1NFM0Y2Yk5FVlg5Y25pekVBdQo3bHptRkE0MmJDWjJMMkZWVDNqYkFaRzcrS1RQQ25xNm1RajBpT0ZoS2M5WXRQQUlSN1MvcjlrQUh6dDhTaXJRCkcxUmdqdCtZZWtFYmxsSzBTcG0yblFLQmdRQ3Rhc3gvRmk4aHR0c1B1TmwxNmVpM3p2Nm9IdHpFT05GUEw0TnoKcy9XenBEc1hrOUFrcHBndkMzQVk0Q2lrMk85WDBIUHNML09zNDV0T3J1cG1Id2NqR3hGMWEzRXc1eExGdGlQRwpCZnpLNkIxRVFqaFRlcnFXY2NLSWRpei9Vci9VRUxOUnN6clIzUnVVck1ESDlRVW5Vdm9Fd2tyTmt6V0ZTOEMxCkYvUWUxUUtCZ1FDQU4zeDBjVFhJUGVQbjVxUjlEQjlsWG51R2VNcURVcmdDc1FuVTFlWDdqbjNXMUlXUWttWGIKSWoxcGN3bzhPVk9HRVpMYzgvcVFYMjFjeU9PS3NzR0lrNlR0R0F0Z3RPMEJocklIdXdQNlJjM1oxRXJQQUk2cgpWK1BIamtlMEF5bFh0OGxoUG03TWNVOElKZWV2cTdBdWxzeHBNaVlwY3dsd1NBMU80NjM3d1E9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
    >   etcd-cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURGVENDQWYyZ0F3SUJBZ0lKQU1OcEpWVHJBNUNDTUEwR0NTcUdTSWIzRFFFQkN3VUFNQkl4RURBT0JnTlYKQkFNTUIyVjBZMlF0WTJFd0hoY05NVGt4TURFd01UUXhPRFExV2hjTk5EY3dNakkxTVRReE9EUTFXakFXTVJRdwpFZ1lEVlFRRERBdGxkR05rTFhObGNuWmxjakNDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DCmdnRUJBT0hvRkFhU3o3b3lVSXF4UEFIVW1aVFlYS0U3dmdzbm01ajllSHI5aFkxK3oxQmNUQnZDeExlMjd1QnIKckxES1Z0TzBwaU9QL0VrZnhYZWdGcit0TmRTalhIMXdVYVVSRXdtaEJvVW52MDBacTFMT1ZobEp1RG5LSE0wQgplSmZmczl0U2RiNXpiQml4K2gvd016TzhUL3cwOStpWWVhQmlmTVllbVRLV2FQak9RQ3RLcjRpQjBPYjBWaE8yCmtkMGxuTFZDUUFRbmNhRUg0OXY1cEw3SUdOVWJSK1diWXR5eXkzRFFDL3VsZzZsZ3YySEFRdVdNQmpqZyt0eVIKK2phM0VjL2NBM1c1WVYyZjdZTzcrNUUyOExzT1YvVFdxVDA1cGNIZ2t2MGp2RHpudi9lK2o4dFh2Q3lwR1VRNgozWXdZNExiSGdlNkpLamczTG1DcEZpK0Vod0VDQXdFQUFhTnFNR2d3Q1FZRFZSMFRCQUl3QURBT0JnTlZIUThCCkFmOEVCQU1DQmFBd0hRWURWUjBsQkJZd0ZBWUlLd1lCQlFVSEF3RUdDQ3NHQVFVRkJ3TUNNQ3dHQTFVZEVRUWwKTUNPQ0NXeHZZMkZzYUc5emRJY0Vmd0FBQVljRXdLZ1BCWWNFd0tnUEJvY0V3S2dQQnpBTkJna3Foa2lHOXcwQgpBUXNGQUFPQ0FRRUFaUU1PVVRCeDZoZlBaNTFsL3BTeGtoamE4NXUwalRGOHdaK0hTYkFNRDlDNElpN1NhVlRWCmhvZkg3ZjVNUlBvMjRKSHlLZWt1RXNNYVpkRWdJUDZxeE5IbE00SkNvN3hhUGN1Y2dCV05veVQxclVrOU9oYloKTXRQbnk3SEppV3hNUWxtbzJSVk56NVVIODFXdUxXejZZOXJYcXM1M0E4YWNqZGtKaEpxRml2RUN4bllCQ0svMApjcGtxZndjS3djcitONWtJbmZjUEhpeGlWQVAvTmgxOGkyMzRZSUFYUklqQXRsZGdGdlRmMmFIb01iUUZtcG1uCnkzUzVLOWtNM01GMkVIWmMwYm5jU05hMnM5bzNGUExzVVN1ckU4emtIMWNSOXBiNDFsQlk5a1RxejgxMURLTDkKWUNQczc4TEFMVEptSS8vOWxUbFhTMm42VEtqZGxvaC9Mdz09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    >   etcd-ca: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5akNDQWJLZ0F3SUJBZ0lKQU53d240aFNJZ3BlTUEwR0NTcUdTSWIzRFFFQkN3VUFNQkl4RURBT0JnTlYKQkFNTUIyVjBZMlF0WTJFd0hoY05NVGt4TURFd01UUXhPREV6V2hjTk5EY3dNakkxTVRReE9ERXpXakFTTVJBdwpEZ1lEVlFRRERBZGxkR05rTFdOaE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBCndlbnhncGVwRmRjU0kzdzg4UlpNT0QrRVVHSEtjVXVmZ1ZSTXlZalJadHBDdzByU2ViWFBKMndTY2NnNVRkL3QKQXVLc3dIQXRtRWI3SjZrR3pTNGVjdXF1ZzFrMC9xY21WQWpGODVyb1UvaGVYcXZjeUx4U05ZbFpNQURURDNwLwpROUZwU3FlT2lZcVIrQjMzMWpuUmR0MXV0YTJyc2VGQTRuOVp3VXV5U2xaWEVuNGtsRXpPVUQ2MG9sVUFHUnNjCjVMSUQ0SnZwQUlTckEwd0p1NWNqbjBwRmtiZlVzUk9ueU14cDlPODh6MzhrK1RpUEN2SFBWTExKbE50cjdBSTQKUFdGRmh2UVczVjlLZEk5YWUrTXZlSXlzRHFmTnlQZ3ZWTTRJMGNJQkJ2SlV4eXVLV2lIazFCbW5lbGw1MisxTwpsaDE2V2FNSjlLZWxKVjF0SDdxeVJ3SURBUUFCb3lNd0lUQVBCZ05WSFJNQkFmOEVCVEFEQVFIL01BNEdBMVVkCkR3RUIvd1FFQXdJQ3BEQU5CZ2txaGtpRzl3MEJBUXNGQUFPQ0FRRUFqcUZUL1NxdHVGdWJsa0o4SlgxYjhtSlEKNCtsTjJIY0VibXF2dFVKViszVGk3Rk1tTFJ4SiszNFd0SW5GNUJPUVlzdFYxdEVEcmVXQjR0Z2R3MXBIOERGUApWZFNnZXZLSkFhUzNMOVRaejBobUxhOW5CQ1d0T3VLeTd3TXlxS1k1NTA2K3Vsd010aTRzY0o3bXlFcnpicWg4ClN0c3VPTTFMbkhTcC94MnVGMkVaL1Q1NG05a2RkcENXaWhXQVpUMmNVOWVlK0FUQzFzbVNIaE41TlNUZ2FJVkQKMHYvclhFWS81UVZaZFdHYmN6M21rcklISXo2LzVjaHR2Yjh2NmlpYktTbWtRMHd5TFZEVnl1Zm9pWXFodEZsQgp6bEt3dnlPMTh0SWdhOVBKdVNDbGtrbWFUaThSMXk4WWtMZzdBYkZZYnZTbUUxLzY2OXY3TjE5a2ZvVzJVZz09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    30c30
    <   etcd_endpoints: "http://<ETCD_IP>:<ETCD_PORT>"
    ---
    >   etcd_endpoints: "https://192.168.15.5:2379,https://192.168.15.6:2379,https://192.168.15.7:2379"
    33,35c33,35
    <   etcd_ca: ""   # "/calico-secrets/etcd-ca"
    <   etcd_cert: "" # "/calico-secrets/etcd-cert"
    <   etcd_key: ""  # "/calico-secrets/etcd-key"
    ---
    >   etcd_ca: "/calico-secrets/etcd-ca"   # "/calico-secrets/etcd-ca"
    >   etcd_cert: "/calico-secrets/etcd-cert" # "/calico-secrets/etcd-cert"
    >   etcd_key: "/calico-secrets/etcd-key"  # "/calico-secrets/etcd-key"
    303a304,305
    >             - name: IP_AUTODETECTION_METHOD
    >               value: "can-reach=192.168.15.5"
    317c319
    <               value: "192.168.0.0/16"
    ---
    >               value: "172.30.0.0/16"
    368a371,372
    >             - name: host-time
    >               mountPath: /etc/localtime
    406a411,414
    >         # Used for time synchronization
    >         - name: host-time
    >           hostPath:
    >             path: /etc/localtime
    490a499,500
    >             - name: host-time
    >               mountPath: /etc/localtime
    503c513,516
    <
    ---
    >         # Used for time synchronization
    >         - name: host-time
    >           hostPath:
    >             path: /etc/localtime
    ```

7. 使用如下命令执行清单文件

    ```shell
    kubectl apply -f calico-etcd.yaml
    ```

### CoreDNS: DNS and Service Discovery

1.11 后 CoreDNS 已取代 Kube DNS 作为集群服务发现元件，由于 Kubernetes 需要让 Pod 与 Pod 之间能够互相通信，要能够通信需要知道彼此的 IP，通常是通过 Kubernetes API 来获取，但是 Pod IP 会因为生命周期变化而改变，因此这种做法无法弹性使用，且还会增加 API Server 负担，基于此问题 Kubernetes 提供了 DNS 服务来作为查询，让 Pod 能够以 Service 名称作为域名来查询 IP 位址，因此使用者就再不需要关心实际 Pod IP，而且 DNS 也会根据 Pod 变化更新资源记录（Record resources）。

1. 执行如下命令，修改 coredns.yaml 文件中的 dns 信息和镜像名（k8s.gcr.io的镜像需要翻墙才能下载）。

    ```shell
    source /home/k8s/env.sh
    sed -i -e "s/__PILLAR__DNS__DOMAIN__/${CLUSTER_DNS_DOMAIN}/" -e "s/__PILLAR__DNS__SERVER__/${CLUSTER_DNS_SVC_IP}/" -e "s/__PILLAR__DNS__MEMORY__LIMIT__/170Mi/" /home/k8s/k8s-manual-files/addons/coredns/coredns.yaml
    sed -i "s#k8s.gcr.io/coredns:1.3.1#coredns/coredns:1.4.0#" /home/k8s/k8s-manual-files/addons/coredns/coredns.yaml
    ```

2. 修改 coredns.yaml 中容器的时区，使用宿主机的时区，从而使时间和宿主机保持一致（可选）。修改后与原文对比差异如下：

    ```shell
    [root@k8s-m1 coredns]# diff coredns.yaml coredns.yaml.base
    67c67
    <         kubernetes cluster.local in-addr.arpa ip6.arpa {
    ---
    >         kubernetes __PILLAR__DNS__DOMAIN__ in-addr.arpa ip6.arpa {
    119c119
    <         image: coredns/coredns:1.4.0
    ---
    >         image: k8s.gcr.io/coredns:1.3.1
    123c123
    <             memory: 170Mi
    ---
    >             memory: __PILLAR__DNS__MEMORY__LIMIT__
    132,134d131
    <         - name: host-time
    <           mountPath: /etc/localtime
    <           readOnly: true
    175,177d171
    <         - name: host-time
    <           hostPath:
    <             path: /etc/localtime
    195c189
    <   clusterIP: 10.96.0.10
    ---
    >   clusterIP: __PILLAR__DNS__SERVER__
    ```

    > coredns.yaml 原文（coredns.yaml.base）地址：<https://github.com/kubernetes/kubernetes/blob/release-1.15/cluster/addons/dns/coredns/coredns.yaml.base>

3. 执行 coredns.yaml 清单文件，安装 coredns

    ```shell
    kubectl apply -f /home/k8s/k8s-manual-files/addons/coredns/coredns.yaml
    ```

4. 检查 coredns 是否启动成功

    ```shell
    [root@k8s-m1 coredns]# kubectl get pods --all-namespaces
    NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
    kube-system   calico-kube-controllers-c84b7d67c-hbb65   1/1     Running   0          53m
    kube-system   calico-node-8gxlw                         1/1     Running   0          53m
    kube-system   calico-node-95ltb                         1/1     Running   0          53m
    kube-system   calico-node-9s7jk                         1/1     Running   0          53m
    kube-system   calico-node-ntbnw                         1/1     Running   0          52m
    kube-system   calico-node-t2tfx                         1/1     Running   0          53m
    kube-system   coredns-677dfcd7d8-qcqfw                  1/1     Running   0          2m42s
    kube-system   kube-proxy-52hkz                          1/1     Running   0          10h
    kube-system   kube-proxy-54f8d                          1/1     Running   0          10h
    kube-system   kube-proxy-757j7                          1/1     Running   0          10h
    kube-system   kube-proxy-c2ht9                          1/1     Running   0          10h
    kube-system   kube-proxy-fdx48                          1/1     Running   0          10h
    ```

5. 测试 coredns 是否正常工作
    新建一个 dnstool 的 pod，执行如下命令

    ```shell
    cat<<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: busybox
      namespace: default
    spec:
      containers:
      - name: busybox
        image: busybox:1.28
        command:
          - sleep
          - "3600"
        imagePullPolicy: IfNotPresent
      restartPolicy: Always
    EOF
    ```

    该 pod 启动成功后，执行如下命令，查看是否能解析到对应的 ip 地址

    ```shell
    [root@k8s-m1 coredns]# kubectl get pods
    NAME      READY   STATUS    RESTARTS   AGE
    busybox   1/1     Running   0          2m7s
    [root@k8s-m1 coredns]# kubectl exec -ti busybox -- nslookup kubernetes
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      kubernetes
    Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
    ```

### dns-horizontal-autoscaler

部署 coredns 时未指定副本数，默认 1 个 coredns 的 pod 运行，可以安装 dns-horizontal-autoscaler 实现 dns 横向自动扩容。

1. Get the name of your DNS Deployment
    List the DNS deployments in your cluster in the kube-system namespace:

    ```shell
    kubectl get deployment -l k8s-app=kube-dns --namespace=kube-system
    ```

    The output is similar to this:

    ```shell
    NAME      READY   UP-TO-DATE   AVAILABLE   AGE
    coredns   1/1     1            1           18m
    ```

    If you don’t see a Deployment for DNS services, you can also look for it by name:

    ```shell
    kubectl get deployment --namespace=kube-system
    ```

    and look for a deployment named coredns or kube-dns.

    Your scale target is

    ```shell
    Deployment/<your-deployment-name>
    ```

    where `<your-deployment-name>` is the name of your DNS Deployment. For example, if the name of your Deployment for DNS is coredns, your scale target is Deployment/coredns.

    > Note: CoreDNS is the default DNS service for Kubernetes. CoreDNS sets the label `k8s-app=kube-dns` so that it can work in clusters that originally used kube-dns.

    本文的 **target** 为 Deployment/coredns

2. Enable DNS horizontal autoscaling
    In this section, you create a new Deployment. The Pods in the Deployment run a container based on the `cluster-proportional-autoscaler-amd64` image.

    执行如下命令，将上一步获取的 **target** 更新到 dns 横向自动扩容的清单文件中，默认的镜像为 `k8s.gcr.io/cluster-proportional-autoscaler-amd64:1.6.0`，该镜像需要国内无法下载，改为`hekai/k8s.gcr.io_cluster-proportional-autoscaler-amd64_1.6.0`

    ```shell
    sed -ri "s#\{\{.Target\}\}#Deployment\/coredns#" /home/k8s/k8s-manual-files/addons/dns-horizontal-autoscaler/dns-horizontal-autoscaler.yaml
    sed -ri "s#k8s.gcr.io/cluster-proportional-autoscaler-amd64:1.6.0#hekai/k8s.gcr.io_cluster-proportional-autoscaler-amd64_1.6.0#" /home/k8s/k8s-manual-files/addons/dns-horizontal-autoscaler/dns-horizontal-autoscaler.yaml
    ```

    使用 hostpath 将宿主机时区文件 /etc/localetime 挂载到容器中，使容器内的时间和宿主机时间保持同步（可选），修改后与原文件对比差异如下：

    ```shell
    [root@k8s-m1 dns-horizontal-autoscaler]# diff dns-horizontal-autoscaler.yaml dns-horizontal-autoscaler.yaml.orig
    88c88
    <         image: hekai/k8s.gcr.io_cluster-proportional-autoscaler-amd64_1.6.0
    ---
    >         image: k8s.gcr.io/cluster-proportional-autoscaler-amd64:1.6.0
    98c98
    <           - --target=Deployment/coredns
    ---
    >           - --target={{.Target}}
    104,107d103
    <         volumeMounts:
    <         - name: host-time
    <           mountPath: /etc/localtime
    <           readOnly: true
    112,115d107
    <       volumes:
    <       - name: host-time
    <         hostPath:
    <           path: /etc/localtime
    ```

3. 执行如下命令新建 dns 横向自动扩容的 Deployment

    ```shell
    kubectl apply -f /home/k8s/k8s-manual-files/addons/dns-horizontal-autoscaler/dns-horizontal-autoscaler.yaml
    ```

> 更多操作请参考官方文档：<https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/>

### metrics-server

[Metrics Server](https://github.com/kubernetes-incubator/metrics-server) 是实现了 Metrics API 的元件,其目标是取代 Heapster 作为 Pod 与 Node 提供资源的 Usage metrics，该元件会从每个 Kubernetes 节点上的 Kubelet 所公开的 Summary API 中收集 Metrics

- Horizontal Pod Autoscaler（HPA）控制器用于实现基于CPU使用率进行自动Pod伸缩的功能。
- HPA 控制器基于 Master 的 kube-controller-manager 服务启动参数 –horizontal-pod-autoscaler-sync-period 定义是时长（默认30秒），周期性监控目标 Pod 的 CPU 使用率，并在满足条件时对 ReplicationController 或 Deployment 中的 Pod 副本数进行调整,以符合用户定义的平均 Pod CPU 使用率。
- 在新版本的 kubernetes 中 Pod CPU 使用率不在来源于 heapster，而是来自于 metrics-server，官网原话是 The –horizontal-pod-autoscaler-use-rest-clients is true or unset. Setting this to false switches to Heapster-based autoscaling, which is deprecated.
- yml 文件来自于github <https://github.com/kubernetes-incubator/metrics-server/tree/master/deploy/1.8+>

若启用 metrics-server，则 kube-apiserver 的 systemed unit 文件（kube-apiserver.service）需要配置如下参数（本文默认已配置）：

```shell
  --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
  --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
  --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
  --requestheader-allowed-names=aggregator  \
  --requestheader-group-headers=X-Remote-Group  \
  --requestheader-extra-headers-prefix=X-Remote-Extra-  \
  --requestheader-username-headers=X-Remote-User  \
```

1. 官方 metric-server 使用的镜像为 `k8s.gcr.io/metrics-server-amd64:v0.3.5`，该镜像在国内无法下载，替换为 `hekai/k8s.gcr.io_metrics-server-amd64_v0.3.5`

    ```shell
    sed -ri "s#k8s.gcr.io/metrics-server-amd64:v0.3.5#hekai/k8s.gcr.io_metrics-server-amd64_v0.3.5#" /home/k8s/k8s-manual-files/addons/metric-server/metrics-server-deployment.yaml
    ```

2. 使用 hostpath 将宿主机时区文件 /etc/localetime  挂载到容器中，使容器内的时间和宿主机时间保持同步（可选），修改后与原文件对比差异如下：

    ```shell
    [root@k8s-m1 metric-server]# diff metrics-server-deployment.yaml metrics-server-deployment.yaml.orig
    30,32d29
    <       - name: host-time
    <         hostPath:
    <           path: /etc/localtime
    35c32
    <         image: hekai/k8s.gcr.io_metrics-server-amd64_v0.3.5
    ---
    >         image: k8s.gcr.io/metrics-server-amd64:v0.3.5
    40,42d36
    <         - name: host-time
    <           mountPath: /etc/localtime
    <           readOnly: true
    ```

3. 执行如下命令，部署 metric-server

    ```shell
    kubectl apply -f /home/k8s/k8s-manual-files/addons/metric-server/
    ```

4. 查看 pod 的状态

    ```shell
    kubectl -n kube-system get po -l k8s-app=metrics-server
    ```

5. 过一段时间就可以查看 node 的 top 信息

    ```shell
    kubectl top nodes
    ```

## Kubernetes Extra Addons

### kubernetes-dashboard

#### 阅读 Dashboard 官方文档

> 参考：<https://github.com/kubernetes/dashboard>

#### 安装 Dashboard

> 注意： k8s 1.15.0 server 包中含有 dashboard 的清单文件，位置为 `kubernetes/cluster/addons/dashboard` ，其中，使用的是 dashboard 1.10.1 版本，因为 dashboard 发布页面上显示 dashboard 1.10.1 版本不支持 k8s 1.15.0，所以此次安装 dashboard 使用的是 dashboard v2.0.0-beta4

1. 在安装新Beta之前，请通过删除其名称空间来删除以前的版本：

   ```console
   kubectl delete ns kubernetes-dashboard
   ```

2. 执行以下命令进行安装

    ```console
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
    ```

3. 访问 dashboard

    访问方式：
    1. 使用 ingress 访问（看下文）
    2. 使用 nodePort 访问
        1. 查看 dashboard 的 Service

            ```console
            [root@k8s-m1 ~]# kubectl get svc -n kubernetes-dashboard
            NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
            dashboard-metrics-scraper   ClusterIP   10.99.202.100   <none>        8000/TCP   30m
            kubernetes-dashboard        ClusterIP   10.103.89.91    <none>        443/TCP    30m
            ```

        2. 修改 dashboard 的 Service 的 服务类型（type）

            ```console
            [root@k8s-m1 ~]# kubectl edit svc -n kubernetes-dashboard kubernetes-dashboard
            ```

            将 `type: ClusterIP` 修改为 `type: NodePort`

        3. 查看该 Service 映射的端口

            ```console
            [root@k8s-m1 ~]# kubectl get svc -n kubernetes-dashboard
            NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
            dashboard-metrics-scraper   ClusterIP   10.99.202.100   <none>        8000/TCP        35m
            kubernetes-dashboard        NodePort    10.103.89.91    <none>        443:30732/TCP   35m
            ```

            可以看到，映射到了节点的30732端口。

        4. 访问 dashboard
            <https://192.168.15.5:30732>

            ![IMAGES](../../images/dashboard-nodeport-20191023.png)

            chrome 浏览器无法访问，因为 dashboard 使用的根证书在本地浏览器不存在，所以不被信任。使用 firefox 浏览器可以访问。配置 https 请查看 ingress-nginx 一节。

4. 登录 dashboard
   1. 使用 token 登录 dashboard
        1. 在 dashboard 的命名空间（本文为 kubernetes-dashboard）内新建一个 sa （Service Account）

            ```console
            kubectl create sa admin-user -n kubernetes-dashboard
            ```

        2. 将新建的名为 dashboard-admin 的 sa 与 名为 cluster-admin 的 clusterrole 绑定

            ```console
            kubectl create clusterrolebinding admin-user --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:admin-user
            ```

        3. 输出可以用来登录的 token

            ```console
            DASHBOARD_LOGIN_TOKEN=$(kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')  | grep -E '^token' | awk '{print $2}')
            echo ${DASHBOARD_LOGIN_TOKEN}
            ```

   2. 使用 kubeconfig 登录 dashboard，需要使用上面的 token

        ```console
        source /home/k8s/env.sh
        # 设置集群参数
        kubectl config set-cluster kubernetes \
        --certificate-authority=/etc/kubernetes/pki/ca.crt \
        --embed-certs=true \
        --server=${KUBE_APISERVER} \
        --kubeconfig=dashboard.kubeconfig

        # 设置客户端认证参数，使用上面创建的 Token
        kubectl config set-credentials admin_user \
        --token=${DASHBOARD_LOGIN_TOKEN} \
        --kubeconfig=dashboard.kubeconfig

        # 设置上下文参数
        kubectl config set-context default \
        --cluster=kubernetes \
        --user=admin_user \
        --kubeconfig=dashboard.kubeconfig

        # 设置默认上下文
        kubectl config use-context default --kubeconfig=dashboard.kubeconfig
        ```

> https证书是 dashboard 自动生成的，因为清单文件（recommended.yaml）中配置了 `--auto-generate-certificates`

### Helm

#### 阅读 Helm 官方文档

[Helm v2.14.3 官方文档阅读笔记](../helm-v2.14.3-官方文档阅读笔记/index.html)

#### Helm 介绍

helm 是 kubernetes 的包管理器。  
helm 通过 kubernetes 的配置文件（通常是 $HOME/.kube/config ）来断定将 tiller （helm 服务端）安装到哪个 context 中。  
如果想弄清楚 tiller 将要安装到哪个 context 中，可以执行 `kubectl config current-context` 或 `kubectl cluster-info` 查看。

```console
[root@k8s-m1 ~]# kubectl config current-context
kubernetes-admin@kubernetes
[root@k8s-m1 ~]# kubectl cluster-info
Kubernetes master is running at https://127.0.0.1:8443
CoreDNS is running at https://127.0.0.1:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:8443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

本文是在拥有全部权限，完全控制的 k8s 集群中使用 helm，即在私有网络中使用，不用考虑额外的安全配置。如果 k8s 集群暴露在一个更大的网络中，或和其他人共享集群（正式环境属于这一类），必须采取额外的步骤保护已安装的 helm ，避免其他用户无意或恶意的操作损坏 helm 或它的数据，在正式环境或多租户环境中保护 helm 的部署，参考：<https://helm.sh/docs/using_helm/#securing-your-helm-installation>

如果 k8s 集群中启用了 RBAC ，则需要在安装 helm 前配置 service account 。

本次部署默认开启了 RBAC （见 kube-apiserver.service），安装之前，需要配置 service account

#### 安装 Helm

##### 安装 helm 客户端

1. 下载期望安装的 helm 版本，下载地址：<https://github.com/helm/helm/releases>
2. 将下载得到的 tar 包解压

    ```console
    tar -zxvf helm-v2.14.1-linux-amd64.tar.gz
    ```

3. 将 helm 二进制文件放到可执行目录中。

   ```console
   mv linux-amd64/helm /usr/local/bin/helm
   ```

##### 安装 helm 服务端（tiller）

1. 在 namespace 为 kube-system 中新建一个 sa （Service Account），名为 tiller

    ```console
    kubectl -n kube-system create sa tiller
    ```

2. 将上一步新建的 sa 与名为 cluster-admin 的clusterrole 进行绑定（clusterrolebinding），该绑定（clusterrolebinding）名为 tiller

    ```console
    kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
    ```

    > Note: 名为 cluster-admin 的 clusterrole 是 kubernetes 集群默认创建的，不需要手动创建。
3. 使用 新建的 sa 安装 helm 服务端（tiller）

    ```console
    helm init --service-account tiller --history-max 200 --tiller-image hekai/gcr.io_kubernetes-helm_tiller_v2.14.1
    ```

    > Note: `tiller` 默认会被安装到 `kube-system` 命名空间内。默认情况下，`tiller` 会使用 `gcr.io/kubernetes-helm/tiller:v2.14.1` 镜像，由于国内墙的原因，该镜像无法下载，可以使用 `--tiller-image` 参数指定 `tiller` 的镜像，此处使用 `hekai/gcr.io_kubernetes-helm_tiller_v2.14.1`
    > 官网推荐初始化 `helm` 时使用 `--history-max` 参数，因为在 helm 的历史记录中，configmaps 和其他对象如果在初始化时不指定 `--history-max` 就会无限地增大，不方便维护。

4. 查看 tiller 是否安装成功

    ```console
    $ kubectl get pods -n kube-system -l app=helm
    NAME                            READY   STATUS    RESTARTS   AGE
    tiller-deploy-799f6d8f7-dsv4r   1/1     Running   0          40s
    $ helm version
    Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
    ```

### MetalLB

#### 阅读 MetalLB官方文档

[MetalLB-v0.8.1-官方文档阅读笔记](../metallb-v0.8.1-官方文档阅读笔记/index.html)

#### MetalLB 介绍

MetalLB 是为裸机部署的 k8s 集群提供的使用标准路由协议的负载均衡器（load-balancer）的实现。

Kubernetes 没有为 k8s 的裸机集群提供网络负载均衡器（对应的 Service 的类型为 LoadBalancer）的实现，Kubernetes 实现的网络负载均衡器都是调用各种 Iaas 平台的粘合代码（代码耦合性很高），如果你没有运行在支持 load-balancer 的 Iaas 平台上，LoadBalancers 被创建后将会一直处于 pending 状态。

裸机部署的 k8s 集群操作者只剩下两种方式将用户的流量引入集群内部， NodePort 和 externalIPs 服务（还有 ingress 呢），这两种方式如果在生产环境使用都有很大的缺点，从而使裸机部署的 k8s 集群成为 k8s 生态系统的二等公民。

MetalLB 通过提供一个整合标准网络设备的网络负载均衡器的实现纠正了这种不平衡，因此在裸机部署的集群外的服务也能够像实现了网络负载均衡器的 Iaas 平台一样工作。

一旦 MetalLB 为服务分配了外部 IP 地址，它就需要使群集之外的网络意识到该 IP 在群集中“存在”。 MetalLB 使用标准路由协议来实现此目的：ARP，NDP或BGP。

2层协议模式（ARP/NDP）: 在 2 层协议模式中，集群中的一台机器拥有服务的所有权，并且使用标准地址发现协议（ipv4 使用 ARP，ipv6 使用 NDP）来让那些 ip 能够在本地网络中可达。从局域网的角度看，该机器上有多个 IP 地址。

优点：2层协议模式的优势是它的通用性，它可以在任何以太网网络上运行，不需要特殊的硬件，甚至不需要特殊的路由器。在2层协议模式中，所有外部服务的流量都会流向一个节点（机器），在该节点上，k8s 的 kube-proxy 组件将流量转发到对应服务的 pods 中。这就意味着，2层协议模式没有实现负载均衡器。相反地，它实现了一个故障转移的机制，以便当前节点如果因为某些原因发生故障时，其他节点可以接替工作。如果 leader 节点因为某些原因发生故障，故障转移会自动进行：旧 leader 的租约超过 10 秒的超时时间后，另一个节点会变成 leader 并且接管服务 ip 的所有权（和 keepalived 比较相似）。

缺点：2层协议模式有两个主要的限制需要被注意，单节点瓶颈和潜在的缓慢故障转移。就像上面描述的一样，2层协议模式中，被选举出的单节点的 leader 会接收该服务ip的所有的流量，这意味着服务的带宽被单节点的带宽限制，这个是使用 ARP 和 NDP 引导流量的基本限制。在当前的 MetalLB 实现中，节点间的故障转移依靠客户端的协作。当一个故障转移发生时， MetalLB 会发送一些2层协议的 gratuitou 数据包通知客户端，让客户端知道和服务 ip 关联的 mac 地址已经发生改变。大多数操作系统知道如何正确的处理gratuitou数据包并且正确地更新缓存，在这种情况下，故障转移可以在几秒内完成。然而，一些系统没有实现gratuitou数据包的处理机制，或者存在错误的操作导致更新缓存缓慢。所有的主流的操作系统(Windows, Mac, Linux）都正确地实现了二层故障转移，所以那些版本较老或使用较少的操作系统可能会存在问题。

BGP 模式：在 BGP 模式中，机器中的所有机器都会与您控制的附近的路由器建立 BGP 对等会话，并且告诉这些路由器如何转发服务的 ip。使用 BGP 模式能够实现真正的不同节点的负载均衡，并实现细粒度的流量控制。

优点：假设您的路由器配置为支持多路径，这将实现真正的负载平衡：MetalLB发布的路由彼此等效，除了它们的下一跳。 这意味着路由器将一起使用所有下一跳，并在它们之间进行负载平衡。数据包到达节点后，kube-proxy负责流量路由的最后一跳，以将数据包到达服务中的一个特定容器。负载平衡的确切实现取决于您的特定路由器模式和配置，但是常见的行为是基于数据包哈希值来平衡每个连接的流量。每个连接意味着一个单独的 tcp 或 udp 会话的所有的数据包都会定向到机器中的某一个单独机器上。流量分配仅在不同的连接进行，而不是在同一个连接的不同包。这个是一个很好的设计，因为在不同的集群节点中分发数据包将会导致在多个级别上产生不良的行为：1. 将单个连接分布在多个路径上会导致数据包在网络上重新排序，这将严重影响最终主机的性能。2. Kubernetes中单节点流量路由在各个节点之间不保证是一致的。 这意味着两个不同的节点可能决定将同一连接的数据包路由到不同的Pod，这将导致连接失败

缺点：将BGP用作负载平衡机制的优势在于，您可以使用标准路由器硬件，而不是定制的负载平衡器。 但是，这也有缺点。最大的问题是，基于BGP的负载平衡无法对地址后端集的更改做出优雅的响应。 这意味着，当群集节点出现故障时，您应该想到与你的服务的所有活动连接被断开（用户将看到“对等方重置连接”）。基于BGP的路由器实现无状态负载平衡。 他们通过散列数据包头中的某些字段，并使用该散列作为可用后端数组的索引，将给定的数据包分配给特定的下一跳。问题在于路由器中使用的哈希值通常不稳定，因此，只要后端集的大小发生变化（例如，当节点的BGP会话断开时），现有连接就会随机有效地进行哈希刷新，这意味着大多数现有连接会突然结束被转发到另一后端，该后端不知道所讨论的连接。这样做的结果是，每当服务的IP→节点映射发生更改时，您都应该会看到连接到服务的大多数活动的连接断开。 没有持续的数据包丢失或黑洞，只有一次干净的清理。BGP 模式无法与其他的 BGP 兼容（如 calico 的 bgp 模式）

#### 安装 MetalLB

1. 安装前检查是否满足要求
    - 一个版本最低为1.13.0的没有网络负载均衡器的 kubernetes 集群
    - 一个能与 MetalLB 共存的[集群网络配置](https://metallb.universe.tf/installation/network-addons/)
    - 一些 ipv4 地址供 MetalLB 使用
    - 根据操作模式，您可能需要一个或多个能够使用BGP的路由器。

    > 本文中 calico 使用的是默认的 IPIP 模式

2. 执行清单文件，在 metallb-system 的命名空间内安装 MetalLB

    ```console
    kubectl apply -f https://raw.githubusercontent.com/google/metallb/master/manifests/metallb.yaml
    ```

    安装清单不包括配置文件。 MetalLB的组件仍将启动，但在定义和部署配置映射之前将保持 idle 状态。

3. 定义配置文件，在配置文件中指定使用二层协议模式或者 BGP 模式，配置文件示例：<https://raw.githubusercontent.com/google/metallb/master/manifests/example-config.yaml>

   **本文使用二层协议模式**

   - 二层协议模式是最简单的，适合大部分情况，不需要任何协议对应的配置，只需要 IP 即可。第2层模式不需要将IP绑定到工作节点的网络接口。 它的工作原理是直接响应本地网络上的ARP请求，从而将计算机的MAC地址提供给客户端。例如，以下配置使 MetalLB 控制从 192.168.15.144 到 192.168.15.148 的 IP ，并配置第2层模式：

        ```yaml
       apiVersion: v1
       kind: ConfigMap
       metadata:
         namespace: metallb-system
         name: config
       data:
         config: |
           address-pools:
           - name: default
             protocol: layer2
             addresses:
             - 192.168.15.144-192.168.15.148
        ```

   - 对于具有一个BGP路由器和一个IP地址范围的基本配置，您需要4条信息：

      - MetalLB应该连接的路由器IP地址，
      - 路由器的AS号，
      - MetalLB应该使用的AS号，
      - 以CIDR前缀表示的IP地址范围。
    例如，如果要给MetalLB提供192.168.10.0/24的范围和AS编号64500，并将其连接到AS编号为64501的地址为10.0.0.1的路由器，则配置如下所示：

       ```yaml
       apiVersion: v1
       kind: ConfigMap
       metadata:
         namespace: metallb-system
         name: config
       data:
         config: |
           peers:
           - peer-address: 10.0.0.1
             peer-asn: 64501
             my-asn: 64500
           address-pools:
           - name: default
             protocol: bgp
             addresses:
             - 192.168.10.0/24
       ```

4. 执行配置文件

```console
kubectl apply -f config.yaml
```

#### 测试 LoadBalancer

新建清单文件，`nginx-test.yaml`，内容如下

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - name: http
          containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

执行部署该清单文件

```console
kubectl apply -f nginx-test.yaml
```

查看是否分配了 EXTERNAL-IP

```console
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>           443/TCP        11d
nginx        LoadBalancer   10.109.245.49   192.168.15.144   80:32024/TCP   8m20s
```

可以看到 Service 类型为 LoadBalancer ，也映射了宿主机的端口

集群内访问该地址

```console
[root@c48-1 ~]# curl 192.168.15.144
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@c48-1 ~]#
```

集群外也可以直接通过 EXTERNAL-IP 访问。

![images](../../images/metallb-test-20191022.png)

因为 Service 也映射了端口，所以也可以通过 NodePort 访问

![images](../../images/metallb-test2-20191022.png)

删除测试的部署

```console
kubectl delete -f nginx-test.yaml
```

### ingress-nginx

#### 阅读 ingress-nginx 官方文档

[Ingress-Nginx-20191015-官方文档阅读笔记](../ingress-nginx-20191015-官方文档阅读笔记/index.html)

#### ingress-nginx 介绍

Ingress 将 k8s 集群内部的 Service 通过 HTTP 和 HTTPS 路由暴露到集群外部。 可以将Ingress配置为提供服务外部可访问的URL，负载平衡流量，终止SSL / TLS并提供基于名称的虚拟主机。 Ingress Controller 通常结合负载平衡器实现 Ingress，尽管它也可以配置边缘路由器或其他前端以帮助处理流量。

Ingress 不会公开任意端口或协议。 若将除 HTTP 和 HTTPS 以外的服务暴露到 Internet，通常使用 Service.Type = NodePort 或 Service.Type = LoadBalancer 类型的服务。

为了使 Ingress 资源正常工作，集群必须运行一个 Ingress Controller 。

本文使用 ingress-nginx 作为 k8s 集群的 Ingress Controller 。

ingress-nginx 是使用 nginx 实现的 Ingress Controller ，使用 nginx 的重点是 nginx 的配置文件（nginx.conf），若 nginx 的配置文件发生变化必然会涉及 nginx configuration reload，导致 ingress controller 短暂的不可用。需要注意的是，nginx 配置文件中，上游（upstream）的变化不会导致 nginx reload（如部署的服务的 Endpoints 发生变化，不会导致 reload）

以下情况会导致 nginx configuration reload

- 创建新的 Ingress
- 现有 Ingress 中添加 TLS 部分
- Ingress annotations 的更改影响的不仅仅是上游（upstream）配置。 例如，load-balance annotation不需要重新加载。
- 从 Ingress 中添加/删除路径。
- 删除 Ingress、Service 或 Secret
- Ingress 中缺少的一些的引用对象变为可用，例如 Service 或 Secret。
- Secret 被更新。

#### 部署 Ingress Controller

##### load-balancer 方式部署

使用 metalLB 提供的 load-balancer 部署 ingress-nginx

***本文使用的是该方式部署的 Ingress Controller***

1. 安装ingress-nginx 清单文件

    ```console
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
    ```

2. 下载 Service 文件，将文件中的 `type: LoadBalancer` 修改为 `type: NodePort`，然后执行安装

    ```console
    wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
    ```

    修改 type

    ```console
    kubectl create -f service-nodeport.yaml
    ```

3. 检查是否分配了 EXTERNAL-IP

    ```console
    $ kubectl get svc -n ingress-nginx
    NAME            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
    ingress-nginx   LoadBalancer   10.107.168.72   192.168.15.144   80:32435/TCP,443:31885/TCP   4m41s
    ```

4. 新建 Ingress 文件测试

    > 注意 Ingress 文件的 namespace 必须与对应 service 的 namespace 保持一致，否则会报错503

##### hostNetwork 方式部署

> 注意： 默认配置从所有名称空间监视Ingress对象。要更改此行为，请使用标志 --watch-namespace 将范围限制为特定的名称空间。
> 如果多个 Ingress 为同一主机定义了不同的路径，则 Ingress Controller 将合并这些定义。

使用 hostnetwork 的方式进行部署，这种方法的好处是，NGINX Ingress控制器可以将端口80和443直接绑定到Kubernetes节点的网络接口，而无需NodePort Services施加额外的网络转换。此部署方法的一个主要限制是，在每个群集节点上只能调度单个NGINX Ingress控制器Pod，因为从技术上讲，在同一网络接口上多次绑定同一端口是不可能的。

缺点是只能部署一个 ingress controller，并且存在单点问题，如新建 ingress ，域名为 example.com，在 dns中配置域名与ip 的映射后，如配置 192.168.15.5 example.com ，若对应的ip（192.168.15.5）宕机了，会导致该ingress 不可用。

1. 下载 ingress-nginx 清单文件

    ```console
    wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
    ```

2. 修改清单文件
   - 将 Deployment 改为使用 DaemonSet 进行部署（保证一个节点上运行一个 ingress controller 容器）
   - 去除清单文件中的副本数限制（replicas: 1）
   - 添加 `hostNetwork: true` ，启用 hostNetwork
   - 添加 `dnsPolicy: ClusterFirstWithHostNet` ，允许 pod 使用 k8s 集群内的 dns 服务
   - 去除参数 `- --publish-service=$(POD_NAMESPACE)/ingress-nginx` ，因为使用 hostNetwork 模式不需要新建 Service，hostNetwork模式下此参数会导致 Ingress 对象的 status 为空
   - 添加参数 `- --report-node-internal-ip-address` ，将所有Ingress对象的状态设置为运行NGINX Ingress控制器的所有节点的内部IP地址。

    修改后和原清单内容对比如下：

    ```shell
    [root@k8s-m1 ingress-nginx]# diff mandatory.yaml mandatory.yaml.orig 
    191c191
    < kind: DaemonSet
    ---
    > kind: Deployment
    199c199
    <   #replicas: 1
    ---
    >   replicas: 1
    216,217d215
    <       hostNetwork: true
    <       dnsPolicy: ClusterFirstWithHostNet
    228c226
    <             #- --publish-service=$(POD_NAMESPACE)/ingress-nginx
    ---
    >             - --publish-service=$(POD_NAMESPACE)/ingress-nginx
    230d227
    <             - --report-node-internal-ip-address
    ```

3. 执行修改后的清单文件，完成 nginx-ingress 的部署

    ```console
    kubectl apply -f mandatory.yaml
    ```

#### 测试 Ingress

将上文中的 nginx-test.yaml 修改 Service 的 tpye ，改为 ClusterIP， ClusterIP 模式只能在集群内访问，集群外无法访问。修改后如下所示：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: ClusterIP
```

部署该清单文件

```console
kubectl create -f nginx-test.yaml
```

查看部署是否成功

```console
[root@k8s-m1 test-nginx]# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6877889877-jcsrj   1/1     Running   0          56s
[root@k8s-m1 test-nginx]# kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   12d
nginx        ClusterIP   10.96.121.19   <none>        80/TCP    58s
```

新建 Ingress ，将 Service 以 http 方式暴露到集群外部，使集群外部可以访问 nginx

```yaml
apiVersion: extensions/v1
kind: Ingress
metadata:
  name: nginx-ingress
 # namespace: kube-system
spec:
  rules:
  - host: nginx.k8s.abc
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
```

执行如下命令，部署该 Ingress

> 注意： Ingress 的命名空间需要和 Deployment 、 Service 的命名空间保持一致。

```console
kubectl create -f test-nginx-ing.yaml
```

过一段时间，查看 ingress 是否新建成功

> 若一段时间后，该 ingress 的 ADDRESS 还是显示为空，则查看 ingress-nginx 的日志排查问题，
> `kubectl logs -n ingress-nginx -f nginx-ingress-controller-799dbf6fbd-f5fkb`，nginx-ingress-controller-799dbf6fbd-f5fkb 替换为自己的 nginx-ingress-controller 对应 pod 的名字

```console
$ kubectl get ing
NAME            HOSTS              ADDRESS          PORTS   AGE
nginx-ingress   nginx.k8s.abc   192.168.15.144   80      2m
```

> 因为本文 ingress-nginx 使用的是 metallb 实现的 loadbalancer 部署的，所以 ADDRESS 显示的是 metallb 为 ingress-nginx 分配的 IP。

在集群外的主机上配置 dns 映射，如 win10 系统，在 `C:\Windows\System32\drivers\etc\hosts` 文件中添加 `192.168.15.144  nginx.k8s.abc`，然后就可以在 win10 的浏览器中直接通过域名访问集群内的 nginx 服务了。

在浏览器中输入 <http://nginx.k8s.abc> ，访问成功。

#### Ingress 配置 https

> 参考：<https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/tls-termination>

本文使用 openssl 生成自签名证书演示。

1. 新增 openssl.cnf 配置文件，文件内容如下：

    ```txt
    $ cat openssl.cnf
    [ req ]
    default_bits = 2048
    default_md = sha256
    distinguished_name = req_distinguished_name

    [req_distinguished_name]

    [ v3_ca ]
    basicConstraints = critical, CA:TRUE
    keyUsage = critical, digitalSignature, keyEncipherment, keyCertSign

    [ v3_req_ingress ]
    basicConstraints = CA:FALSE
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth, clientAuth
    subjectAltName = @alt_names_ingress

    [ alt_names_ingress ]
    DNS.1 = *.k8s.abc
    DNS.2 = localhost
    IP.1 = 192.168.15.5
    IP.2 = 192.168.15.6
    IP.3 = 192.168.15.7
    IP.4 = 192.168.15.143
    IP.5 = 192.168.15.161
    IP.6 = 127.0.0.1
    ```

    > 其中 alt_names_ingress 中指定了使用该证书的域名和 ip，若不加的话，通过该 域名或 IP 访问的时候浏览器中证书没有问题，但是还会出现警告。
    > 域名使用的是通配符指定的，所以生成的证书适合格式为 *.k8s.abc 的域名，且能使用 ip 进行访问。

2. 生成CA私钥

    ```console
    openssl genrsa -out ca.key 2048
    ```

3. 使用CA私钥生成CA自签名证书

    ```console
    openssl req -x509 -new -nodes -key ca.key -config openssl.cnf -subj "/CN=Ingress-CA" -extensions v3_ca -out ca.crt -days 10000
    ```

4. 生成 ingress 私钥

    ```console
    openssl genrsa -out ingress.key 2048
    ```

5. 使用 ingress 私钥生成证书签名请求

    ```console
    openssl req -new -key ingress.key -subj "/CN=ingress" -out ingress.csr
    ```

6. 使用 CA 证书根据证书签名请求签发 ingress 证书

    ```console
    openssl x509 -in ingress.csr -req -CA ca.crt -CAkey ca.key -CAcreateserial -extensions v3_req_ingress -extfile openssl.cnf -out ingress.crt -days 10000
    ```

至此， 适合 *.k8s.abc 的 ingress 证书生成好了，下面来试用下

将上文中以 http 方式暴露的 nginx 服务的 ingress 修改下，使其以 https 方式暴露 nginx 服务。

删除上文中新建的 nginx ingress 对象

```console
kubectl delete -f test-nginx-ing.yaml
```

修改 test-nginx-ing.yaml 文件，修改后的内容如下：

```console
$ cat test-nginx-ing.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
 # namespace: kube-system
spec:
  rules:
  - host: nginx.k8s.abc
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
  tls:
  - hosts:
    - nginx.k8s.abc
    secretName: nginx-ingress-secret
```

创建 ingress 使用的 secret

```console
kubectl create secret generic nginx-ingress-secret --from-file=tls.crt=ingress.crt --from-file=tls.key=ingress.key --from-file=ca.crt=ca.crt
```

> Note: The CA Certificate must contain the trusted certificate authority chain to verify client certificates.
> 该 secret 的 namespace （命名空间）必须和 ingress 一致。

创建修改后 ingress 对象

```console
kubectl create -f test-nginx-ing.yaml
```

查看生成的 ingress

```console
$ kubectl get ing
NAME            HOSTS              ADDRESS          PORTS     AGE
nginx-ingress   nginx.k8s.abc   192.168.15.144   80, 443   5m42s
```

在集群外的主机上通过 <https://nginx.k8s.abc> 访问集群内的 nginx 服务。

浏览器显示不安全，如下图

![IMAGE](../../images/ingress-nginx-https-1-20191023.png)

查看该网页的证书，如下图

![IMAGE](../../images/ingress-nginx-https-2-20191023.png)

原因是本文使用的是自签名证书配置的 https，CA证书不被浏览器信任。解决办法为将生成的CA证书（ca.crt）导入到 win10 的`受信任的根证书办法机构`中。双击 ca.crt 导入即可。

导入ca.crt成功后，重新打开浏览器，访问 <https://nginx.k8s.abc> ，就可以看到 https 证书被浏览器信任了。

![IMAGE](../../images/ingress-nginx-https-3-20191023.png)

> 注意：如果生成证书时，对应的域名或 ip 不在 openssl.cnf 中配置，会出现浏览器提示 https 网站不安全，但是证书没问题的情况。

下面介绍下 win10 系统如何删除导入的证书

在win10左下角的搜索框中搜索并打开 `mmc`，依次点击 `文件` -> `添加/删除管理单元`，在可用的管理单元中双击证书，添加到`我的用户账户`中，点击`确定`，之后就可以对证书进行删除等操作了。

#### 使用 ingress 为 kubernetes dashboard 暴露服务

上文中，kubernetes dashboard 是使用 nodeport 向集群外暴露的服务。此处介绍下使用 ingress 向集群外暴露 dashboard

dashboard 默认使用 https 暴露服务，它的清单文件中包含了启动参数`auto-generate-certificates`，表示使用 https 提供服务，并且自动生成证书，生成的证书存放在名为 `kubernetes-dashboard-certs` 的 secret 中。2.0版本的 dashboard 的命名空间为 kubernetes-dashboard，低版本的命名空间为 kube-system。

下面我们将 dashboard 自动生成的证书替换为自己的证书，简单起见，使用上文生成的 ingress 的证书。

1. 将 `kubernetes-dashboard` 命名空间中的名为`kubernetes-dashboard-certs` 的 `secret` 删掉

    ```console
    kubectl delete secret -n kubernetes-dashboard kubernetes-dashboard-certs
    ```

2. 生成新的 `kubernetes-dashboard-certs`

    ```console
    kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.crt=ingress.crt --from-file=dashboard.key=ingress.key -n kubernetes-dashboard
    ```

3. 重启 dashboard 的 pod ，是 dashboard 挂载新的证书

    ```console
    kubectl delete pods -n kubernetes-dashboard kubernetes-dashboard-d9fdb86c9-2w2zz
    ```

    > `kubernetes-dashboard-d9fdb86c9-2w2zz` 替换为自己的 `dashboard` 的 `pod` 名。

4. 通过 nodeport 访问 dashboard ，就是受浏览器信任的了。
   > 上文中已经将 dashboard 使用的证书的 ca证书导入到win10中，所以 https 可以被信任。

    ![IMAGE](../../images/dashboard-nodeport-https-20191023.png)

5. 使用 ingress 暴露 dashboard 服务，首先在 kubernetes-dashboard 命名空间内生成 ingress 使用的证书

    ```console
    kubectl create secret generic dashboard-ingress-secret --from-file=tls.crt=ingress.crt --from-file=tls.key=ingress.key --from-file=ca.crt=ca.crt -n kubernetes-dashboard
    ```

    >注意： 本文中 dashboard 的 ingress 中使用的证书（在 secret 中）和 dashboard 的容器中使用的证书是同一个，如果不想使用同一个的话，需要使用同一个 CA 签发的证书。

6. 新建 dashboard 的 ingress 文件，内容如下：

    ```console
    $ cat dashboard-ingress.yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: dashboard-ingress
      namespace: kubernetes-dashboard
      annotations:
        nginx.ingress.kubernetes.io/ingress.class: "nginx"
        nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    spec:
      rules:
      - host: dashboard.k8s.abc
        http:
          paths:
          - backend:
              serviceName: kubernetes-dashboard
              servicePort: 443
      tls:
      - hosts:
        - dashboard.k8s.abc
        secretName: dashboard-ingress-secret
    ```

    > 注意：  
    > 必须要加上 `annotations: nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`  
    > Using `backend-protocol` annotations is possible to indicate how NGINX should communicate with the backend service. (Replaces `secure-backends` in older versions), Valid Values: HTTP, HTTPS, GRPC, GRPCS and AJP, By default NGINX uses `HTTP`.

7. 部署 ingress 文件

    ```console
    kubectl apply -f dashboard-ingress.yaml
    ```

    查看 ingress

    ```console
    # kubectl get ing -n kubernetes-dashboard 
    NAME                HOSTS                  ADDRESS          PORTS     AGE
    dashboard-ingress   dashboard.k8s.abc   192.168.15.144   80, 443   25m
    ```

8. 在 win10 的 hosts 文件中配置 域名与ip 的映射，然后通过域名访问dashboard

    ![IMAGES](../../images/dashboard-ingress-20191023.png)

    这样手动配置 ingress 的证书还是比较麻烦的，每新增一个 ingress，都要手动在 ingress 的命名空间内新建一个包含证书的 secret，后面了解了 cert-manager 后，可以让 cert-manager 自动为我们生成证书，续约证书等。

### cert-manager

#### 阅读 cert-manager官方文档

[Cert-Manager-v0.11-官方文档阅读笔记](../cert-manager-v0.11-官方文档阅读笔记/index.html)

#### cert-manager 介绍

cert-manager 是一个基于 kubernetes 的证书管理控制器。它可以帮助你从不同的来源签发证书，如 Let's Encrypt ， HashiCorp Vault ， Venafi ，简单的签名密钥对或自签名。

它将确保证书有效并且是最新的，并在到期前尝试在配置的时间续订证书。

cert-manager 支持运行在 kubernetes 或 openshif 上，本文只介绍基于 kubernetes 的安装。

cert-manager 可以使用 清单文件安装 或使用 helm 安装。它利用自定义资源对象（CustomResourceDefinitions）来配置证书颁发机构和请求证书。cert-manager 部署完成后，需要你配置代表证书颁发机构的 Issuer 或 ClusterIssuer 。

在你可以使用 cert-manager 签发证书之前，必须配置至少一个 Issuer 或 ClusterIssuer。 Issuer 与 ClusterIssuer 的区别是 Issuer 的作用域为单个命名空间，并且只能在其命名空间内签发证书，这在多租户环境中非常有用。而 ClusterIssuer 是 Issuer 的集群范围版本，证书资源可以在任何命名空间中引用它。

cert-manager 支持的 Issuer 类型有：

- CA-颁发由X509签名密钥对签名的证书，该证书存储在Kubernetes API server 的 secret 中。
- 自签名-颁发自签名证书。
- ACME-颁发通过对诸如 Let's Encrypt 之类的 ACME 服务器执行挑战验证而获得的证书。
- Vault-颁发从配置了 Vault PKI 后端的 Vault 实例中的获得的证书。
- Venafi-颁发从 Venafi Cloud 或 Trust Protection Platform 实例获得的证书。

概念理解：

- [Issuer](https://cert-manager.readthedocs.io/en/latest/reference/issuers.html)（证书颁发者）
- [ClusterIssuers](https://cert-manager.readthedocs.io/en/latest/reference/clusterissuers.html)（集群范围内的证书颁发者）
- [Certificates](https://cert-manager.readthedocs.io/en/latest/reference/certificates.html)（证书）


本文只介绍 CA 和 自签名 两种 Issuer。

##### 配置 CA Issuers

> 参考：<https://cert-manager.readthedocs.io/en/latest/tasks/issuers/setup-ca.html>

cert-manager 可用于使用存储在 Kubernetes Secret 资源中的任意签名密钥对来获取证书。

本指南将向您展示如何基于存储在 secret 资源中的签名密钥对来配置和创建基于 CA 的 issuer。

1. 生成一个签名密钥对（如果你已有签名密钥对，可不执行此步骤）

    CA Issuer不会自动为您创建和管理签名密钥对。 所以，您将需要提供自己的 CA 或使用诸如 openssl 或 cfssl 之类的工具生成自签名的 CA 。

    本步骤将说明如何生成新的签名密钥对，但是如果你的签名密钥对是 CA ，你就可以使用自己的代替，跳过这一步。

    ```console
    # Generate a CA private key
    $ openssl genrsa -out ca.key 2048

    # Create a self signed Certificate, valid for 10yrs with the 'signing' option set
    $ openssl req -x509 -new -nodes -key ca.key -subj "/CN=${COMMON_NAME}" -days 3650 -reqexts v3_req -extensions v3_ca -out ca.crt
    ```

    上述命令会生成两个文件， ca.key 和 ca.crt ，分别为你的签名证书对的密钥和证书。如果你已经有了自己的签名密钥对，你应该分别命名私钥和证书为 ca.key 和 ca.crt 。

2. 将签名密钥对以 secret 形式保存

    我们将创建一个 Issuer ，它将使用此密钥对生成签名证书。 您可以在[Issuer参考文档](https://cert-manager.readthedocs.io/en/latest/reference/issuers.html)中阅读有关 Issuer 资源的更多信息。 为了允许 Issuer 引用签名密钥对，我们将其存储在Kubernetes Secret资源中。

    Issuer 是命名空间范围的资源，因此它们只能在自己的命名空间中引用 Secrets。 因此，我们将密钥对放入与 Issuer 相同的名称空间中。 我们也可以创建一个ClusterIssuer，它是Issuer的群集范围版本。 有关ClusterIssuers的更多信息，请阅读[ClusterIssuer参考文档](https://cert-manager.readthedocs.io/en/latest/reference/clusterissuers.html)。

    以下命令将在默认名称空间中创建一个包含签名密钥对的Secret：

    ```console
    kubectl create secret tls ca-key-pair \
       --cert=ca.crt \
       --key=ca.key \
       --namespace=default
    ```

3. 引用 secret 创建一个 Issuer

    现在，我们可以创建一个引用刚刚创建的 Secret 资源的 Issuer ：

    ```yaml
    apiVersion: cert-manager.io/v1alpha2
    kind: Issuer
    metadata:
      name: ca-issuer
      namespace: default
    spec:
      ca:
        secretName: ca-key-pair
    ```

    我们现在准备获得证书！

4. 获取签名证书

    现在，我们可以创建以下证书资源，以指定所需的证书。 您可以在[参考文档](https://cert-manager.readthedocs.io/en/latest/reference/certificates.html)中阅读有关证书资源的更多信息。

    ```yaml
    apiVersion: cert-manager.io/v1alpha2
    kind: Certificate
    metadata:
      name: example-com
      namespace: default
    spec:
      secretName: example-com-tls
      issuerRef:
        name: ca-issuer
        # We can reference ClusterIssuers by changing the kind here.
        # The default value is Issuer (i.e. a locally namespaced Issuer)
        kind: Issuer
      commonName: example.com
      organization:
      - Example CA
      dnsNames:
      - example.com
      - www.example.com
    ```

    为了使用 Issuer 获得证书，我们必须在与 Issuer 相同的名称空间中创建证书资源，因为 Issuer 是命名空间范围的资源。 如果我们想在多个命名空间之间重用签名密钥对，则可以选择创建 ClusterIssuer 。

    一旦我们创建了 Certificate 资源，cert-manager 将尝试使用 Issuer `ca-issuer`获取证书。 如果成功，则证书将存储在名为`example-com-tls` 的 Secret 资源中，位于与 Certificate 资源相同的名称空间中（default）。

    上面的示例将 commonName 字段显式设置为 `example.com` 。 如果 dnsNames 字段中尚未包含 commonName 字段，则 cert-manager 会自动将 commonName 字段添加为[DNS SAN](https://en.wikipedia.org/wiki/Subject_Alternative_Name)。

    如果我们未指定 commonName 字段，则将使用第一个指定的DNS SAN（在 dnsNames 下）作为证书的通用名称。

    创建上述证书后，我们可以检查是否已成功获得证书，如下所示：

    ```console
    $ kubectl describe certificate example-com
    Events:
      Type     Reason                 Age              From                     Message
      ----     ------                 ----             ----                     -------
      Warning  ErrorCheckCertificate  26s              cert-manager-controller  Error checking existing TLS certificate: secret "example-com-tls" not found
      Normal   PrepareCertificate     26s              cert-manager-controller  Preparing certificate with issuer
      Normal   IssueCertificate       26s              cert-manager-controller  Issuing certificate...
      Normal   CertificateIssued      25s              cert-manager-controller  Certificate issued successfully
    ```

    你还可以通过执行 `kubectl get secret example-com-tls -o yaml` 检查 issuer 生成的证书是否成功，你应该看到一个base64编码的签名TLS密钥对。

    一旦获得证书，cert-manager 将继续检查其有效性，并在证书即将到期时尝试对其进行更新。 当证书上的 “Not After” 字段小于当前时间加上30天时，cert-manager 认为证书即将到期。 对于基于 CA 的颁发者，cert-manager 将颁发 “Not After” 字段设置为当前时间加上365天的证书。

##### 配置自签名 Issuer

自签名 Issuers 将发行自签名证书。

在 Kubernetes 中构建 PKI 时，或生成与 [CA Issuer](https://cert-manager.readthedocs.io/en/latest/tasks/issuers/setup-ca.html) 一起使用的根 CA 的方式时，这很有用。

自签名颁发者不包含其他配置字段，并且可以使用如下资源创建：

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: selfsigning-issuer
spec:
  selfSigned: {}
```

> `selfSigned: {}` 一行的存在足以表明该 Issuer 的类型为 “selfsigned”。

创建后，你可以像平常一样通过在 `issuerRef` 中引用上面新建的 Issuer 来颁发证书：

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: example-crt
spec:
  secretName: my-selfsigned-cert
  commonName: "my-selfsigned-root-ca"
  isCA: true
  issuerRef:
    name: selfsigning-issuer
    kind: ClusterIssuer
```

##### 为 ingress 资源自动创建证书

cert-manager 可以配置为通过 Ingress 上的 annotations 自动为 Ingress 资源设置 TLS 证书。

cert-manager的一个子组件 ingress-shim 负责这项工作。

ingress-shim 监视整个集群中的 Ingress 资源。如果它观察到具有特定 annotations 的Ingress，它将确保和 ingress 中secret相同名称的并且 配置和 ingress 中描述一致的证书资源存在。例如：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    # add an annotation indicating the issuer to use.
    cert-manager.io/cluster-issuer: nameOfClusterIssuer
  name: myIngress
  namespace: myIngress
spec:
  rules:
  - host: myingress.com
    http:
      paths:
      - backend:
          serviceName: myservice
          servicePort: 80
        path: /
  tls: # < placing a host in the TLS config will indicate a cert should be created
  - hosts:
    - myingress.com
    secretName: myingress-cert # < cert-manager will store the created certificate in this secret.
```

***Configuration***
Since cert-manager v0.2.2, ingress-shim is deployed automatically as part of a Helm chart installation.

If you would also like to use the old [kube-lego](https://github.com/jetstack/kube-lego) `kubernetes.io/tls-acme: "true"` annotation for fully automated TLS, you will need to configure a default Issuer when deploying cert-manager. This can be done by adding the following `--set` when deploying using Helm:

```console
--set ingressShim.defaultIssuerName=letsencrypt-prod \
--set ingressShim.defaultIssuerKind=ClusterIssuer
```

In the above example, cert-manager will create Certificate resources that reference the ClusterIssuer letsencrypt-prod for all Ingresses that have a `kubernetes.io/tls-acme: "true"` annotation.

For more information on deploying cert-manager, read the [deployment guide](https://cert-manager.readthedocs.io/en/latest/getting-started/index.html).

kube-lego 是 cert-manager 的上一个版本，现在已经停止更新了。

***支持的 annotations***
你可以在 ingress 中执行如下的 annotations 从而能够触发证书（Certificate ）资源能够被自动创建。

- `cert-manager.io/issuer` - the name of an Issuer to acquire the certificate required for this ingress from. The Issuer **must** be in the same namespace as the Ingress resource.
- `cert-manager.io/cluster-issuer` - the name of a ClusterIssuer to acquire the certificate required for this ingress from. It does not matter which namespace your Ingress resides, as ClusterIssuers are non-namespaced resources.
- `kubernetes.io/tls-acme: "true"` - this annotation requires additional configuration of the ingress-shim (see above). Namely, a default issuer must be specified as arguments to the ingress-shim container.
- `acme.cert-manager.io/http01-ingress-class` - this annotation allows you to configure ingress class that will be used to solve challenges for this ingress. Customising this is useful when you are trying to secure internal services, and need to solve challenges using different ingress class to that of the ingress. If not specified and the ‘acme-http01-edit-in-place’ annotation is not set, this defaults to the ingress class of the ingress resource.
- `acme.cert-manager.io/http01-edit-in-place`: "true" - this controls whether the ingress is modified ‘in-place’, or a new one created specifically for the http01 challenge. If present, and set to “true” the existing ingress will be modified. Any other value, or the absence of the annotation assumes “false”.

#### 安装 cert-manager

本文使用清单文件安装，若想使用 helm 安装，请参考[官网安装](https://cert-manager.readthedocs.io/en/latest/getting-started/install/kubernetes.html)

1. 新建 cert-manager 命名空间，cert-manager 默认安装在名为 cert-manager 的命名空间内

    ```console
    kubectl create namespace cert-manager
    ```

2. 执行清单文件，所有关于 cert-manager 的资源都定义在了清单文件中

    ```console
    kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.11.0/cert-manager.yaml
    ```

    > 注意：如果你运行的 k8s 版本小于或等于 v1.15，你需要在执行命令时添加 `--validate=false` ，因为本文的 k8s 版本为 v1.15.0，所以需要添加该参数  
    > cert-manager 会部署一个 webhook 作为 apiservice ，当 webhook 不在运行时但这个 apiservice 还存在的时候进行卸载 cert-manager 就会出现问题，所以卸载 cert-manager 的时候确保按照[卸载文档](https://cert-manager.readthedocs.io/en/latest/tasks/uninstall/kubernetes.html)进行。

##### 配置 CA Issuers

本文使用上面[Ingress 配置 https][#Ingress 配置 https] 创建的 CA 证书来配置 ClusterIssuer 。

1. 生成签名密钥对

    跳过，使用 [Ingress 配置 https] 中 创建的 ca.key 和 ca.crt 。

2. 将签名密钥对保存为 secret

    ```console
    kubectl create secret tls ca-key-pair \
       --cert=ca.crt \
       --key=ca.key \
       --namespace=cert-manager
    ```

    > 注意： 因为本次使用的是 ClusterIssuer，所以这个 secret 需要放在 cert-manager 命名空间内。若使用 Issuer，则 secret 需要放在与 Issuer 相同的命名空间内。

3. 新建一个 ClusterIssuer 引用上面的 secret

    ```console
    $ cat cluster-issuer.yaml
    apiVersion: cert-manager.io/v1alpha2
    kind: ClusterIssuer
    metadata:
      name: cluster-issuer
    spec:
      ca:
        secretName: ca-key-pair
    ```

    > ClusterIssuer 不用指定命名空间，Issuer 需要指定命名空间。ClusterIssuer 引用的 secret 需要在 cert-manager 命名空间内，如果 secret 在 default 命名空间内，则 ClusterIssuer 找不到 secret : `secret "ca-key-pair" not found`

    通过如下命令查看 ClusterIssuer 的状态

    ```console
    kubectl describe ClusterIssuer cluster-issuer
    ```

4. 利用 annotations ，让 cert-manager 的 ingress-shim 组件自动生成 ingress 的证书。

    使用 dashboard 的 ingress 进行测试

    1. 首先删除 dashboard 的 ingress

        ```console
        kubectl delete ing -n kubernetes-dashboard dashboard-ingress
        ```

        > dashboard-ingress 为 dashboard 的 ingress 中的 name  
        > 也可以通过 dashboard 的 ingress 的 yaml 文件删除 ingress，命令为：`kubectl delete -f dashboard-ingress.yaml`

    2. 删除 dashboard ingress 的 secret

        ```console
        kubectl delete secret -n kubernetes-dashboard dashboard-ingress-secret
        ```

    3. 添加 annotations
        修改 dashboard 的 ingress 文件，dashboard-ingress.yaml ， 添加 annotations cert-manager.io/cluster-issuer，修改后内容如下：

        ```console
        $ cat dashboard-ingress.yaml
        apiVersion: extensions/v1beta1
        kind: Ingress
        metadata:
          name: dashboard-ingress
          namespace: kubernetes-dashboard
          annotations:
            nginx.ingress.kubernetes.io/ingress.class: "nginx"
            nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
            cert-manager.io/cluster-issuer: "cluster-issuer"
        spec:
          rules:
          - host: dashboard.k8s.abc
            http:
              paths:
              - backend:
                  serviceName: kubernetes-dashboard
                  servicePort: 443
          tls:
          - hosts:
            - dashboard.k8s.abc
            secretName: dashboard-ingress-secret
        ```

    4. 安装 ingress

        ```console
        kubectl create -f dashboard-ingress.yaml
        ```

        可以看到执行完之后， kubernetes-dashboard 中新建了一个名为`dashboard-ingress-secret`的secret，通过浏览器也能够正常访问。

使用 cert-manager 自动生成 ingress 证书的好处还有，可以使用任意的域名，如果手动配置 ingress 证书，需要在证书上配置域名或 IP，如果域名不对的话，访问还是会提示不安全。使用 cert-manager 解决了这个问题，它只使用 CA 根证书，自动生成 https 的证书，可用在 ingress 里配置任意的域名。

以后在创建 ingress 的时候，就可以加上 annotations: `cert-manager.io/cluster-issuer: "cluster-issuer"`，这样 cert-manager 就能自动为我们创建 https 证书了。

### nfs-storageclass

#### 安装 NFS 服务器

略

#### 安装 NFS 客户端

1. 获取 NFS 服务器的连接信息
2. 在[发布版本页面](https://github.com/kubernetes-incubator/external-storage/releases)下载最新的发布包

    ```console
    wget https://github.com/kubernetes-incubator/external-storage/archive/v5.5.0.tar.gz
    ```

    下载后解压

    ```console
    tar -zxvf external-storage-5.5.0.tar.gz
    ```

    进入 nfs-client 目录

    ```console
    cd external-storage-5.5.0/nfs-client/
    ```

3. 设置权限和修改命名空间

    新建 nfs-storageclass 命名空间

    ```console
    kubectl create ns nfs-storageclass
    ```

    修改 deploy/rbac.yaml 中的ServiceAccount，Role 等的命名空间，修改后内容如下

    ```yaml
    kind: ServiceAccount
    apiVersion: v1
    metadata:
      name: nfs-client-provisioner
      namespace: nfs-storageclass
    ---
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: nfs-client-provisioner-runner
    rules:
      - apiGroups: [""]
        resources: ["persistentvolumes"]
        verbs: ["get", "list", "watch", "create", "delete"]
      - apiGroups: [""]
        resources: ["persistentvolumeclaims"]
        verbs: ["get", "list", "watch", "update"]
      - apiGroups: ["storage.k8s.io"]
        resources: ["storageclasses"]
        verbs: ["get", "list", "watch"]
      - apiGroups: [""]
        resources: ["events"]
        verbs: ["create", "update", "patch"]
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: run-nfs-client-provisioner
    subjects:
      - kind: ServiceAccount
        name: nfs-client-provisioner
        namespace: nfs-storageclass
    roleRef:
      kind: ClusterRole
      name: nfs-client-provisioner-runner
      apiGroup: rbac.authorization.k8s.io
    ---
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: leader-locking-nfs-client-provisioner
      namespace: nfs-storageclass
    rules:
      - apiGroups: [""]
        resources: ["endpoints"]
        verbs: ["get", "list", "watch", "create", "update", "patch"]
    ---
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: leader-locking-nfs-client-provisioner
      namespace: nfs-storageclass
    subjects:
      - kind: ServiceAccount
        name: nfs-client-provisioner
        # replace with namespace where provisioner is deployed
        namespace: nfs-storageclass
    roleRef:
      kind: Role
      name: leader-locking-nfs-client-provisioner
      apiGroup: rbac.authorization.k8s.io

    ```

    执行如下命令部署 rbac.yaml

    ```console
    kubectl create -f deploy/rbac.yaml
    ```

4. 修改 deployment.yaml 配置 NFS server 连接信息
    将 deploy/deployment.yaml 中的 `<YOUR NFS SERVER HOSTNAME>` 修改为 nfs server 的 hostname 或 ip，同时也修改下`NFS_PATH` 和 `PROVISIONER_NAME` 对应的值。若修改 `deploy/deployment.yaml` 中 `PROVISIONER_NAME` 对应的值，必须同时修改下 `deploy/class.yaml` 中 `provisioner` 的值，使两个值保持一致。

    因为修改了默认的命名空间，默认命名空间为 default，改为了 nfs-storageclass，所以，也需要修改下 deploy/deployment.yaml 中 Deployment 和 ServiceAccount 的命名空间。

    deploy/class.yaml 这个 StorageClass 定义文件中，添加注解 `storageclass.kubernetes.io/is-default-class: "true"`， 该注解将该 storageclass 配置为默认的 StorageClass。参考：<https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/>

    nfs server 地址为 `192.168.15.143`， 路径为 `/home/data/nfs/data`

    修改后的 deploy/deployment.yaml 内容如下

    ```yaml
    #apiVersion: v1
    #kind: ServiceAccount
    #metadata:
    #  name: nfs-client-provisioner
    #  namespace: nfs-storageclass
    ---
    kind: Deployment
    apiVersion: extensions/v1beta1
    metadata:
      name: nfs-client-provisioner
      namespace: nfs-storageclass
    spec:
      replicas: 1
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: nfs-client-provisioner
        spec:
          serviceAccountName: nfs-client-provisioner
          containers:
            - name: nfs-client-provisioner
              image: quay.io/external_storage/nfs-client-provisioner:latest
              volumeMounts:
                - name: nfs-client-root
                  mountPath: /persistentvolumes
              env:
                - name: PROVISIONER_NAME
                  value: nfs-storage
                - name: NFS_SERVER
                  value: 192.168.15.143
                - name: NFS_PATH
                  value: /home/data/nfs/data
          volumes:
            - name: nfs-client-root
              nfs:
                server: 192.168.15.143
                path: /home/data/nfs/data
    ```

    注释掉了 ServiceAccount，因为在 deploy/rbac.yaml 中已经定义了该 ServiceAccount

    修改后的 deploy/class.yaml 内容如下

    ```yaml
    kind: StorageClass
    metadata:
      name: managed-nfs-storage
      annotations:
        storageclass.kubernetes.io/is-default-class: "true" # 将该sc配置为默认的sc
    provisioner: nfs-storage # or choose another name, must match deployment's env PROVISIONER_NAME'
    parameters:
      archiveOnDelete: "false"
    ```

5. 部署

    ```console
    kubectl create -f deploy/deployment.yaml -f deploy/class.yaml
    ```

    查看安装是否成功

    ```console
    $ kubectl get pods -n nfs-storageclass
    NAME                                      READY   STATUS    RESTARTS   AGE
    nfs-client-provisioner-699948c999-45p56   1/1     Running   0          108s
    $ kubectl get sc
    NAME                            PROVISIONER   AGE
    managed-nfs-storage (default)   nfs-storage   2m37s
    ```

6. 测试

    ```console
    kubectl create -f deploy/test-claim.yaml -f deploy/test-pod.yaml
    ```

    查看 pvc 是否挂载成功

    ```console
    [root@k8s-m1 nfs-client]# kubectl get pvc
    NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
    test-claim   Bound    pvc-bf3135ac-a04d-418a-9a87-675bef49c022   1Mi        RWX            managed-nfs-storage   4m53s
    ```

    挂载成功，删除测试部署

    ```console
    kubectl delete -f deploy/test-claim.yaml -f deploy/test-pod.yaml
    ```

### Harbor

#### 阅读 Harbor 官方文档

[Harbor-v1.9.1-官方文档阅读笔记](../harbor-v1.9.1-官方文档阅读笔记/index.html)

#### 安装 Harbor

本文使用 helm 安装 Harbor，使用 nfs-storageclass 作为 harbor 的后端存储。

参考：<https://github.com/goharbor/harbor-helm/tree/1.2.0>

1. 在 [harbor-helm 发布页面](https://github.com/goharbor/harbor-helm/releases)下载发布包

    ```console
    wget https://github.com/goharbor/harbor-helm/archive/v1.2.1.tar.gz
    ```

    解压发布包

    ```console
    tar -zxvf v1.2.1.tar.gz
    ```

2. 编辑 `values.yaml` ，修改 harbor-helm 的配置

    - 配置这么暴露 harboar 的服务，默认是 Ingress，本文使用 Ingress 暴漏服务，所以不用修改
    - tls 默认是开启的。如果使用 ingress 暴漏服务的时候，tls是关闭的，则需要在执行 pull/push 命令的时候加上端口号。[见issues](Refer to <https://github.com/goharbor/harbor/issues/5291>)
    - 配置 tls secret，对应 values.yaml 中的secretName 和 notarySecretName，若 notarySecretName不指定的时候，会使用secretName 的证书和key，secretName 指定 harbor 管理界面域名使用的的https证书，notarySecretName 中指定的 notary 页面使用的 https 证书。secretName 中指定的 secret 需要包含 tls.crt 、tls.key 和 ca.crt， secret 中证书的内容和 ingress secret 的内容相同，所以可以使用 ingress 的证书的手动生成方式或使用 cert-manager 进行生成。secretName和notarySecretName不要配置成一样的，否则其中一个页面无法 https。
    - 配置 expose.ingress.hosts，将 core 和 notary 分别配置为期望的域名。
    - ingress annotatiaons 中添加 cert-manager.io/cluster-issuer: "cluster-issuer"，使用 cert-manager 自动生成包含证书的 secret。
    - 配置 externalURL，因为本文使用的是 ingress 暴漏服务，所以此处的域名应该和 `expose.ingress.hosts.core` 配置的域名相同
    - 配置如何持久化数据，默认使用 storageclass 持久化，上面安装了 nfs-storage ，并设为了默认的 sc ，可以使用它来动态提供 pv，满足 pvc 的要求，所以不用修改

    > helm install 的方式
    > - 从 chart 仓库中安装 `helm install stable/mysql`
    > - 从本地的 chart 包安装 `helm install foo-0.1.1.tgz`
    > - 从 chart 包解压的目录进行安装 `helm install path/to/foo`
    > - 通过完整 URL 安装 `helm install https://example.com/charts/foo-1.2.3.tgz`

    修改后的 values.yaml 和原文对比如下：

    ```console
    [root@k8s-m1 harbor-helm-1.2.1]# diff values.yaml values.yaml.orig
    19c19
    <     secretName: "core.harbor.ingress.secret"
    ---
    >     secretName: ""
    23c23
    <     notarySecretName: "notary.harbor.ingress.secret"
    ---
    >     notarySecretName: ""
    29,30c29,30
    <       core: core.harbor.k8s.abc
    <       notary: notary.harbor.k8s.abc
    ---
    >       core: core.harbor.domain
    >       notary: notary.harbor.domain
    41d40
    <       cert-manager.io/cluster-issuer: "cluster-issuer"
    102c101
    < externalURL: https://core.harbor.k8s.abc
    ---
    > externalURL: https://core.harbor.domain
    ```

    annotations 添加 `cert-manager.io/cluster-issuer: cluster-issuer`，cert-manager 监听到后会新建 ingress tls 的 secret。

3. 安装 harbor-helm

    新建 harbor-helm 安装的命名空间

    ```console
    kubectl create ns harbor
    ```

    在 `values.yaml` 所在的目录进行安装，所以 install 后面是一个点，表示当前目录。`--namespace` 指定安装的命名空间，`--name` 指定安装后的名字

    ```console
    [root@k8s-m1 harbor-helm-1.2.1]# helm install . --namespace harbor --name harbor-helm --values values.yaml
    ```

    helm v3 使用如下命令安装

    ```console
    [root@k8s-m1 harbor-helm-1.2.2]# helm install . --namespace harbor --name-template harbor-helm --values values.yaml
    ```

4. 登录管理界面

    查看生成的 ingress

    ```console
    $ kubectl get ing -n harbor
    NAME                         HOSTS                                             ADDRESS          PORTS     AGE
    harbor-helm-harbor-ingress   core.harbor.k8s.abc,notary.harbor.k8s.abc   192.168.15.144   80, 443   7m51s
    ```

    在 win10 hosts文件中，添加 `192.168.15.144  *.harbor.k8s.abc`，然后通过浏览器访问 `https://core.harbor.k8s.abc` 和 `https://notary.harbor.k8s.abc` ，https 是安全的。

    harbor 管理界面 `https://core.harbor.k8s.abc` 的默认用户名密码为 `admin/Harbor12345`

5. 通过 docker client 推送或拉取镜像

    docker client 默认使用 https 连接镜像仓库，可以通过[这个文档](https://docs.docker.com/registry/insecure/)了解如何配置使用 http 连接镜像仓库。

    如果一个私有镜像仓库使用http 或使用未知ca的https 提供服务，需要将 `--insecure-registry myregistrydomain.com` 添加到 docker client 的守护进程启动参数中。

    对于使用HTTPS的镜像仓库，如果您有权访问镜像仓库的CA证书，只需将CA证书放在/etc/docker/certs.d/myregistrydomain.com/ca.crt。

    ***下载镜像***
    如果镜像属于私有的项目，你首先需要登录

    ```console
    docker login core.harbor.k8s.abc
    ```

    然后才可以下载镜像

    ```console
    docker pull core.harbor.k8s.abc/library/centos:latest
    ```

    ***推送镜像***  
    推送镜像之前，必须通过 harbor UI 新建一个对应的项目，如新建一个 demo 项目

    然后通过 docker client 登录 harbor

    ```console
    docker login core.harbor.k8s.abc
    ```

    为 镜像 打标签

    ```console
    docker tag centos:latest core.harbor.k8s.abc/demo/centos:latest
    ```

    推送镜像到 harbor 仓库

    ```console
    docker push core.harbor.k8s.abc/demo/centos:latest
    ```

### tidb

#### 官方文档

<https://pingcap.com/docs-cn/v3.0/tidb-in-kubernetes/tidb-operator-overview/>

#### 安装

[TiDB-v3.0-安装记录](../tidb-v3.0-官方文档阅读笔记/index.html)

### jenkins

#### jenkins master

jenkins master 镜像基于 `jenkins/jenkins:lts` 进行构建，`jinkins/jenkins:lts` 的介绍详见： <https://github.com/jenkinsci/docker>

因为 oracle jdk 无法直接下载（官网下载方式需要登录），所以无法直接在 `Dockerfile` 里指定下载地址，此处通过在 build 时使用 `--build-arg` 参数指定 jdk 的下载地址，jdk 的下载地址可以通过 onedriver 生成，生成的地址虽然只有一个小时有效期，但是足够进行构建镜像了。为了加速构建镜像，此处使用 dockerhub 进行镜像的构建，所以需要在 Dockerfile 所在的目录新建一个 hooks 目录，里面放入构建文件 build。

jenkins master 的Dockerfile 等构建资料详见：<https://github.com/msdemt/dockerfiles/tree/master/jenkins/jenkins-master>

关于为什么不使用 COPY 命令将 jdk 包传入镜像进行构建的原因，是因为 jdk 的 tar 包有 185mb，如果使用 COPY 命令，则生成的镜像会多出一层，这层有185mb ，即使在后面的 RUN 命令中删除了 tar 包，这层还是存在的，从而导致生成的镜像比较大，所以将下载 jdk，删除多余文件都放在了一个 RUN 命令中执行。

关于使用 `dockerhub` 构建镜像如何传入 `ARG` 参数的方法详见：<https://github.com/docker/hub-feedback/issues/508>

使用 dockerhub 生成镜像，名为 hekai/jenkins-slaver-oraclejdk8-debian:20191029

下载镜像后，为镜像重新打 tag ，使镜像名符合 harbor 仓库的要求

```console
docker tag hekai/jenkins-slaver-oraclejdk8-debian:20191029 core.harbor.k8s.abc/library/jenkins-slaver-oraclejdk8-debian:20191029
```

将生成后的镜像推送到 harbor 仓库

```console
docker push core.harbor.k8s.abc/library/jenkins-slaver-oraclejdk8-debian:20191029
```

修改 `jenkins.yaml` 中的镜像名为 `core.harbor.k8s.abc/library/jenkins-slaver-oraclejdk8-debian:20191029`，执行清单文件，生成 jenkins 服务

```console
kubectl apply -f jenkins.yaml
kubectl apply -f jenkins-ing.yml
```

查看 jenkins master 的 pod，可以看到 jenkins 服务第一次启动生成的默认密码

```console
$ kubectl get pods -n jenkins-ns
NAME                              READY   STATUS    RESTARTS   AGE
jenkins-deploy-7dbcdd9848-d8b27   0/1     Running   0          42s
$ kubectl logs -n jenkins-ns jenkins-deploy-7dbcdd9848-d8b27
... ...
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

a9ad8fbef3e448c0b208c54fd5313a24

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
... ...
2019-10-25 07:18:54.943+0000 [id=39]  INFO  hudson.util.Retrier#start: The attempt #1 to do the action check updates server failed with an allowed exception:
java.net.SocketTimeoutException: connect timed out
  at java.net.PlainSocketImpl.socketConnect(Native Method)
... ...
```

jenkins pod 启动后，会访问 `https://updates.jenkins.io/update-center.json` ，下载 updates 资源，生成 updates 目录，如果访问不通，则 jenkins 的 pod 会处于 0/1 的状态。

如下

```console
$ kubectl get pods -n jenkins-ns
NAME                              READY   STATUS    RESTARTS   AGE
jenkins-deploy-7dbcdd9848-rrgct   0/1     Running   0          77s
```

查看该 pod 的日志，会发现有个 `SocketTimeoutException` 的错误。

需要修改 /var/jenkins_home/ 中的 `hudson.model.UpdateCenter.xml` 文件中指定的更新地址。

因为本文使用的使 nfs 挂载的 jenkins，在 jenkins.yaml 中找到jenkins 容器的 `/var/jenkins_home` 对应的 pvc 的名字，本文为 `jenkins-home-claim`，然后在 nfs server 挂载目录中找到对应的挂载文件夹，本文为 `/home/data/nfs/data/jenkins-ns-jenkins-home-claim-pvc-4662aeda-4628-4425-a9f1-5a6461425ca5`

```console
[root@c48-2 jenkins-ns-jenkins-home-claim-pvc-4662aeda-4628-4425-a9f1-5a6461425ca5]# ls
config.xml               hudson.model.UpdateCenter.xml  jenkins.install.UpgradeWizard.state  jobs  nodeMonitors.xml  plugins     secret.key.not-so-secret  userContent  war
copy_reference_file.log  identity.key.enc               jenkins.telemetry.Correlator.xml     logs  nodes             secret.key  secrets                   users
[root@c48-2 jenkins-ns-jenkins-home-claim-pvc-4662aeda-4628-4425-a9f1-5a6461425ca5]# cat hudson.model.UpdateCenter.xml
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://updates.jenkins.io/update-center.json</url>
  </site>
</sites>
```

jenkins 第一次启动会访问 `hudson.model.UpdateCenter.xml` 中指定的地址，生成 `updates` 目录及其内容，因为默认的地址为 `https://updates.jenkins.io/update-center.json` ，该地址国内可能无法访问，所以 jenkins 没有启动成功。 将该地址修改为清华大学 jenkins 镜像源的地址 `https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`，手工修改或执行如下命令修改

```console
sed -i 's/updates.jenkins.io/mirrors.tuna.tsinghua.edu.cn\/jenkins\/updates/g' hudson.model.UpdateCenter.xml
```

修改后，重启 jenkins 对应的 pod

```console
kubectl delete pods -n jenkins-ns jenkins-deploy-7dbcdd9848-d8b27
```

jenkins 服务启动成功，生成了 updates 文件夹

```console
[root@c48-2 jenkins-ns-jenkins-home-claim-pvc-4662aeda-4628-4425-a9f1-5a6461425ca5]# ls
config.xml               hudson.model.UpdateCenter.xml  jenkins.install.UpgradeWizard.state  jobs  nodeMonitors.xml  plugins        secret.key                secrets  userContent  war
copy_reference_file.log  identity.key.enc               jenkins.telemetry.Correlator.xml     logs  nodes             queue.xml.bak  secret.key.not-so-secret  updates  users
```

此时若进行插件下载，会出现很多失败的情况，因为 jenkins 通过 updates/default.json 中的地址进行对应插件的下载，默认的地址为 `http://updates.jenkins-ci.org/download/plugins` ，这个地址国内也是无法访问或访问特别慢，将该地址修改为清华大学的 jenkins 镜像对应的地址，同时修改测试连接的地址为`www.baidu.com` ，使用如下命令修改

```console
sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' updates/default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' updates/default.json
```

再次重启 jenkins 对应的 pod

```console
kubectl delete pods -n jenkins-ns jenkins-deploy-7dbcdd9848-9brbx
```

jenkins 服务启动成功

```console
$ kubectl get pods -n jenkins-ns
NAME                              READY   STATUS    RESTARTS   AGE
jenkins-deploy-7dbcdd9848-rrgct   1/1     Running   0          77s
```

查看 jenkins 的初始密码，记录初始密码的文件位于 `secrets/initialAdminPassword`

```console
[root@c48-2 jenkins-ns-jenkins-home-claim-pvc-4662aeda-4628-4425-a9f1-5a6461425ca5]# cat secrets/initialAdminPassword
a9ad8fbef3e448c0b208c54fd5313a24
```

访问 jenkins 网页 `https://jenkins.k8s.abc`，输入初始密码，安装推荐的插件，配置管理员用户，若首次登录页面空白。再次重启下 jenkins 的 pod 。

```console
kubectl delete pods -n jenkins-ns jenkins-deploy-7dbcdd9848-rrgct
```

万能的重启，如果 jenkins 有个启动参数能够指定插件源就不用这么麻烦了。。。

jenkins master 安装成功。

登录后，修改 admin 用户默认密码。

#### jenkins slaver

jenkins slaver 的镜像基于 `jenkins/jnlp-slave` ， 详见：<https://github.com/jenkinsci/docker-jnlp-slave>

jenkins slaver 的 Dockerfile 等构建资料详见：<https://github.com/msdemt/dockerfiles/tree/master/jenkins/jenkins-slaver>

#### jdk images

由于要使用 jenkins 运行 Java 项目，所以需要包含 oracle jdk 和 tomcat 的镜像。

基于 debian 的 oracle jdk 镜像的Dockerfile等构建资料详见：<https://github.com/msdemt/dockerfiles/tree/master/jdk/oraclejdk-with-debian>

基于 debian 和 oracle jdk 的 tomcat 镜像详见：<https://github.com/msdemt/dockerfiles/tree/master/tomcat/tomcat9-oraclejdk8-debian>

#### jenkins pipeline

jenkins 安装插件：Kubernetes Cli 、Kubernetes、Kubernetes Continuous Deploy

kubernetes 插件介绍：<https://github.com/jenkinsci/kubernetes-plugin>

1. 配置 kubernetes

    依次选择 Manage Jenkins - Configure System - 云 - 新增一个云

    名称为 `kubernetes` ， kubernetes 地址为 `https://kubernetes.default` ，点击连接测试，测试成功后，保存。

2. 生成 access token
   点击用户列表，选择 当前的用户（admin）- 设置 - 添加新的Token 输入 Token的名称，点击生成，如生成的Token 为：11fa1a1b89200f30dbf4df6275e949de65

3. 将用户名，token 和 jenkins 的地址 配置到 svn post-commit 脚本，用于 svn 提交代码后，向 jenkins 发消息触发构建

    ```python
    #!/usr/bin/env python
    # -*- coding: utf-8 -*-
    import urllib2, sys, os, base64, json
    # 本仓库的目录路径
    repos_path = sys.argv[1]
    # 版本修订号
    revision = sys.argv[2]
    # svnlook命令所在路径
    svnlook = '/usr/bin/svnlook'
    # Jenkins的地址
    baseurl = 'http://192.168.15.143:30000'
    # Jenkins用户和用户的api token
    user_id = 'admin'
    api_token = '111ec492afd7a821259bbfd9d405047cac'

    def my_urlopen(url, data=None, header={ }):
       request = urllib2.Request(url, data)
       base64string = base64.encodestring('%s:%s' % (user_id, api_token)).replace('\n', '')
       header['Authorization'] = 'Basic %s' % base64string
       for key in header:
           request.add_header(key, header[key])
       try:
           response = urllib2.urlopen(request)
       except urllib2.URLError as e:
           print '[Exception URLError]:', e
           sys.exit(1)
       else:
           result = response.read( )
       finally:
           if 'response' in vars( ):
               response.close( )
       return result

    def main( ):
       # 获取uuid值
       command = r'%s uuid %s' % (svnlook, repos_path)
       with os.popen(command) as f:
           uuid = f.read( ).strip( )
       # 获取仓库变更信息
       command = r'%s changed --revision %s %s' % (svnlook, revision, repos_path)
       with os.popen(command) as f:
           data = f.read( ).strip( )
       # 获取crumb
       url = r'%s/crumbIssuer/api/json' % baseurl
       crumb_dict = json.loads(my_urlopen(url))
       # 触发Jenkins
       url = r'%s/subversion/%s/notifyCommit?rev=%s' % (baseurl, uuid, revision)
       header={'Content-Type':'text/plain', 'charset':'UTF-8', crumb_dict['crumbRequestField']:crumb_dict['crumb']}
       my_urlopen(url, data, header)

    main( )
    ```

4. 生成 svn 检出代码对应的流水线语句

    进入 jenkins 首页，选择 `凭据` ，点击 `全局`， `添加凭据`，输入 svn 的用户名、密码，id 填入 svn 标记该凭据为 svn ，输入自定义的描述，然后点击确定。

    进入 test 项目中，点击`流水线语法`，在`片段生成器` - `步骤` - `示例步骤` 中选择 `checkout: Check out from version control`， `SCM` 选择 `Subversion`，在 `Repository URL` 输入 svn 的项目地址，在 `Credentials` 选择记录有svn用户名和密码的凭据，然后点击生成流水线脚本，将脚本保存，下一步在 pipeline 中的 `CheckOut` 阶段使用。

5. 新建流水线，用于自动构建 svn 上的项目

    依次点击 新建Item - 输入任务名称（如 test） - 选择流水线 - 点击确定

    在`构建触发器`中选择 `Poll SCM`，表示 jenkins 收到 svn 的消息后，触发该流程。

    在`流水线`中输入流水线脚本，然后点击保存。

    test 项目的流水线脚本如下：

    ```pipeline
    pipeline {
      agent {
        kubernetes {
          //cloud 'kubernetes'
          //defaultContainer ''
          yaml  """
                kind: Pod
                metadata:
                  namespace: jenkins-ns
                  labels:
                    label: jenkins-agent
                spec:
                  serviceAccountName: jenkins-sa
                  containers:
                  - name: jnlp
                    image: core.harbor.k8s.abc/library/jenkins-slaver-oraclejdk8-debian:20191029
                    imagePullPolicy: IfNotPresent
                    env:
                    - name: "JENKINS_DIRECT_CONNECTION"
                      value: "192.168.15.5:50000"
                    ports:
                    - name: http
                      containerPort: 8080
                    - name: agent
                      containerPort: 50000
                    volumeMounts:
                    - name: hosts
                      mountPath: /etc/hosts
                    - name: docker-config
                      mountPath: /etc/docker
                    - name: docker-bin
                      mountPath: /usr/bin/docker
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
                    - name: maven-repo
                      mountPath: /maven/repo
                  volumes:
                  - name: hosts
                    hostPath:
                      path: /etc/hosts
                  - name: docker-config
                    hostPath:
                      path: /etc/docker
                  - name: docker-bin
                    hostPath:
                      path: /usr/bin/docker
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
                  - name: maven-repo
                    persistentVolumeClaim:
                      claimName: maven-repo-claim
                """
            }
        }
        // 对应Do not allow concurrent builds
        options {
            disableConcurrentBuilds()
        }
        //triggers { pollSCM('') }
        environment {
            // ------ 以下内容，每个项目可能均有不同，按需修改 ------
            // branch: 分支，一般是test、 master，对应从哪个分支拉取代码，也对应究竟执行_deploy文件夹下的test配置还是master配置
            branch = "master"
            // namespace:
            namespace = "default"
            // appname：对应deployment中的app
            app = "demo"
            // port：对应deployment中的port
            port= "38082"
            privileged = "true"
            // replicas：对应deployment中的replicas
            replicas = 1
            //svn repo的地址
            //svn="http://192.168.15.143:8888/svn/code/abc"
            //log：对应deployment中的log
            log="/usr/local/tomcat/logs"
            // ------ 以下内容，一般所有的项目都一样，不经常修改 ------
            // harbor inner address
            repoHost = "core.harbor.k8s.abc"
            // harbor的账号密码信息，在jenkins中配置用户名/密码形式的认证信息，命名成harbor即可
            harborCreds = credentials('harbor')
            //harborCreds_USR = "admin"
            //harborCreds_PSW = "Harbor12345"
        }
        // ------ 以下内容无需修改 ------
        stages {
            // 开始构建前清空工作目录
            stage ("CleanWS"){
                steps {
                    script {
                        try{
                            deleteDir()
                        }catch(err){
                            echo "${err}"
                            sh 'exit 1'
                        }
                    }
                }
            }
            // 拉取
            stage ("CheckOut"){
                steps {
                    script {
                        try{
                            checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[cancelProcessOnExternalsFail: true, credentialsId: 'test', depthOption: 'infinity', ignoreExternalsOption: true, local: '.', remote: 'http://192.168.15.143:8888/svn/code/abc/trunk/demo']], quietOperation: true, workspaceUpdater: [$class: 'UpdateUpdater']])
                        }catch(err){
                            echo "${err}"
                            sh 'exit 1'
                        }
                    }
                }
            }
            // 编译
            stage ("Compile"){
                steps {
                    script {
                        try{
                            sh "mvn clean && mvn package"
                            //sh "cp target/*.war "
                        }catch(err){
                            echo "${err}"
                            sh 'exit 1'
                        }
                    }
                }
            }
            // 构建
            stage ("Build"){
                steps {
                    script {
                        try{
                            // 镜像tag用时间戳代表
                            sh "date +%Y%m%d%H%m%S > timestamp"
                            tag = readFile('timestamp').replace("\n", "").replace("\r", "")
                            repoPath = "${repoHost}/${namespace}/${app}:${tag}"
                            // 根据分支，进入_deploy下对应的不同文件夹，通过dockerfile打包镜像
                            sh "cp _deploy/${branch}/* ./"
                            withDockerRegistry(credentialsId: 'harbor', url: 'http://core.harbor.k8s.abc') {
                                // some block
                                //sh "docker login -u ${harborCreds_USR} -p ${harborCreds_PSW} ${repoHost}"
                                sh "docker build -t ${repoPath}  ."
                            }
                            //sh "docker login -u ${harborCreds_USR} -p ${harborCreds_PSW} ${repoHost}"
                            //sh "docker build -t ${repoPath}  ."
                        }catch(err){
                            echo "${err}"
                            sh 'exit 1'
                        }
                    }
                }
            }
            // 镜像推送到harbor
            stage ("Push"){
                steps {
                    script {
                        try{
                            withDockerRegistry(credentialsId: 'harbor', url: 'https://core.harbor.k8s.abc') {

                                sh "docker push ${repoPath}"
                            }

                        }catch(err){
                            echo "${err}"
                            sh 'exit 1'
                        }
                    }
                }
            }
            // 使用pipeline script中复制的变量替换deployment.yaml中的占位变量，执行deployment.yaml进行部署
            stage ("Deploy"){
                steps {
                    script {
                        try{
                            sh "sed -i 's|#namespace#|${namespace}|g' deployment.yaml"
                            sh "sed -i 's|#app#|${app}|g' deployment.yaml"
                            sh "sed -i 's|#image#|${repoPath}|g' deployment.yaml"
                            sh "sed -i 's|#port#|${port}|g' deployment.yaml"
                            sh "sed -i 's|#privileged#|${privileged}|g' deployment.yaml"
                            sh "sed -i 's|#replicas#|${replicas}|g' deployment.yaml"
                            sh "sed -i 's|#log#|${log}|g' deployment.yaml"
                            sh "cat deployment.yaml"
                            sh "kubectl apply -f deployment.yaml"
                        }catch(err){
                            echo "${err}"
                            sh 'exit 1'
                        }
                    }
                }
            }
        }
        post{
            failure {
                script {
                    echo '项目构建失败，发送邮件！'
                    send_email_results("失败","test","hekai@abc.com,276516848@qq.com")
                }
            }
            success {
                script {
                    echo '项目构建成功，发送邮件！'
                    send_email_results("成功","test","hekai@abc.com")
                }
            }
        }
    }
    def send_email_results(status,SVNBranch,to_email_address_list) {
        def subject = "Jenkins 自动构建 : " + env.JOB_NAME + "/" + env.BUILD_ID + " 项目 " +  status
        def result_url = env.BUILD_URL + "console"
        def text =
                """
          <html>
          <style type="text/css">
          </style>
          <body>
          <div id="content">
          <h1>Summary</h1>
          <div id="sum2">
              <h2>Jenkins 自动构建</h2>
              <ul>
              <li>构建任务URL : <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></li>
               <li>构建结果URL : <a href='${result_url}'>${result_url}</a></li>
              </ul>
          </div>
          <div id="sum0">
          <h2>SVN Branch</h2>
          <ul>
          <li>${SVNBranch}</li>
          </ul>
          </div>
          </div></body></html>
        """

        mail body: text, subject: subject,  mimeType: 'text/html', to: to_email_address_list
    }

    ```

    对如上的 pipeline 脚本进行解释：首先定义了 jenkins slaver 的模板，jenkins slaver 使用的镜像为 `core.harbor.k8s.abc/library/jenkins-slaver-oraclejdk8-debian:20191029` ，并且将一些目录挂载到了 jenkins slaver 的容器中，从而使 jenkins slaver 可以执行 docker 的命令。下面定义了一些环境变量，这些环境变量针对不同的项目需要有不同的配置，用于生成部署服务时的 yaml 文件。再往下就是 jenkins slaver 进行项目构建的不同阶段，分别为清空当前工作目录，拉取代码（第 4 步生成），编译、打包代码，生成该服务的镜像，将镜像推送到 Harbor 仓库，使用变量替换 deployment.yaml 中的对应值，生成部署用的 yaml，并执行 deployment.yaml 生成服务。在下面就是对构建成功或失败进行发送邮件，当前使用的是 jenkins 自带的邮件工具，需要配置下才能正常发送邮件，配置方法见下一步。

6. 配置邮件

    进入 jenkins 首页，点击 `Manage Jenkins` - `Config System` - `邮件通知`， 输入 `SMTP服务器`，如 `smtp.abc.com`，输入`用户默认邮件后缀`，如 `@abc.com`, 选中 `使用SMTP认证`，输入邮箱的用户名和对应的密码，选中`使用SSL协议`， `SMTP端口`输入`465`。这样就配置好了。

    测试下邮件通不通，在Reply-To Address 中输入某个邮箱，选中 `通过发送测试邮件测试配置`， jenkins 就会向该邮箱发送测试邮件，收到测试邮件，就表示邮件配置好了。

7. 至此，jenkins 上的项目就配置好了，进入项目，点击 `Build Now`，查看是否能构建成功，没问题的话就可以通过提交代码触发项目的自动构建和部署了

k8s 部署的 jenkins 与普通部署的 jenkins 的区别：
在该流程中，使用docker 部署 jenkins master 不是必须的，也可以使用普通部署的 jenkins，见：<https://github.com/jenkinsci/kubernetes-plugin>，其中指出 `It is not required to run the Jenkins master inside Kubernetes.`，所以，普通部署的 jenkins 也可以通过安装 kubernetes 插件实现上述的功能。有点不同的是若是普通部署的jenkins，在配置 kubernetes 时（第一步），因为当前的 jenkins master 并不在 k8s 的集群中，所以无法通过 `https://kubernetes.default` 访问到 k8s，需要配为 k8s master 的 api server 的地址，如：`https://192.168.15.5:6443`，还需要配置 `Kubernetes 服务证书 key`

k8s 部署的 jenkins master 可以保证当该 jenkins master 因为意外的原因服务停止的时候，docker 可以自动重启 jenkins master。

### rook-ceph

### istio

### knative

### wayne

### prometheus

### efk

### gitlab

[gitlab 安装记录](../post/gitlab安装记录.md/index.html)

使用 k8s 安装 gitlab https://docs.gitlab.com/charts/

gitlab 官方的 helm chart 地址：https://gitlab.com/gitlab-org/charts/gitlab

下载 master 分支的 chart

```console
mkdir /home/k8s/gitlab && cd /home/k8s/gitlab
wget https://gitlab.com/gitlab-org/charts/gitlab/-/archive/master/gitlab-master.tar.gz
tar -zxf gitlab-master.tar.gz
```

下载依赖的子chart

```console
helm dep up ./gitlab-master
```

1. coredns 挂载宿主机 /etc/hosts 并配置 hosts 插件

export release=mygitlab

kubectl create ns gitlab

1. 使用 CA 证书生成 gitlab-runner 连接 gitlab server 认证用的 secret
kubectl create secret generic gitlab.k8s.abc.crt --from-file=gitlab.k8s.abc.crt=ca.crt -n gitlab
2. 生成 gitlab server 与 registry 交互使用的证书
生成私钥和证书签名请求
openssl req -new -newkey rsa:4096 -subj "/CN=gitlab-issuer" -nodes -keyout registry-k8s-abc.key -out registry-k8s-abc.csr
使用CA证书根据证书签名请求签发证书
openssl x509 -req -sha256 -days 365 -in registry-k8s-abc.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out registry-k8s-abc.crt -n gitlab

> 原文是自签名生成一个证书和私钥
> openssl req -new -newkey rsa:4096 -subj "/CN=gitlab-issuer" -nodes -x509 -keyout certs/registry-example-com.key -out certs/registry-example-com.crt

根据生成的证书和私钥建一个secret
kubectl create secret generic ${release}-registry-secret --from-file=registry-auth.key=registry-k8s-abc.key --from-file=registry-auth.crt=registry-k8s-abc.crt -n gitlab

然后生成yaml时可以通过 global.registry.certificate.secret 引用该 secret

生成 incoming secret所需的

kubectl create secret generic gitlab-incoming-imap-pwd-secret --from-file=./gitlab-incoming-imap-pwd-key -n gitlab
kubectl create secret generic gitlab-outgoing-smtp-pwd-secret --from-file=gitlab-outgoing-smtp-pwd-key=gitlab-incoming-imap-pwd-key -n gitlab


生成 gitlab.yaml

helm template ./gitlab-master --name ${release} --set global.edition=ce --set global.time_zone=Beijing --set global.hosts.domain=k8s.abc --set nginx-ingress.enabled=false --set global.ingress.class=nginx --set prometheus.install=false --set certmanager.install=false --set global.ingress.configureCertmanager=false  --set gitlab.unicorn.ingress.tls.secretName=release-gitlab-tls --set registry.ingress.tls.secretName=release-registry-tls --set minio.ingress.tls.secretName=release-minio-tls --set global.ingress.annotations."cert-manager\.io\/cluster-issuer"=cluster-issuer --set global.registry.certificate.secret=${release}-registry-secret --namespace gitlab > gitlab.yaml

检查 gitlab.yaml，少 namespace的加上 namespace，imageTag 为 latest 的imagePullPolicy配置为Alaways,其他的配置为IfNotPresent

将 gitlab.k8s.abc.crt 挂载到 gitlab-runner pod 的/home/gitlab-runner/.gitlab-runner/certs/

然后就可以通过 gitlab.yaml 安装gitlab了。

获取gitlab root 用户的密码

kubectl get secret -n gitlab mygitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ; echo



gitlab 配置smtp，用来新建用户时为用户发送邮件

根据文档里的描述配置 smtp 参数后，新建用户时，查看容器日志
kubectl logs -n gitlab -f mygitlab-sidekiq-all-in-1-b8b4d49cd-947nb

邮件未成功发送，日志中出现错误

```console
[ActiveJob] [ActionMailer::DeliveryJob] [962a7c24-d6a8-4450-a8ff-5356779d2f4a] Sent mail to 276516848@qq.com (17488.3ms)
[ActiveJob] [ActionMailer::DeliveryJob] [962a7c24-d6a8-4450-a8ff-5356779d2f4a] Error performing ActionMailer::DeliveryJob (Job ID: 962a7c24-d6a8-4450-a8ff-5356779d2f4a) from Sidekiq(mailers) in 17550.92ms: EOFError (end of file reached):
```

百度下错误：`EOFError (end of file reached)`

见：<https://www.jianshu.com/p/af142a66d781>

如果 smtp 使用 25 端口，则不要配置 ssl 相关的项目

如果 smtp 使用 465 或其他端口，则需要配置如下参数

```console
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['smtp_openssl_verify_mode'] = 'none'
```

所以文件配置如下：

```console
helm template . \
  --name ${release} \
  --set global.edition=ce \
  --set global.time_zone=Beijing \
  --set global.hosts.domain=k8s.abc \
  --set global.grafana.enabled=false \
  --set global.ingress.class=nginx \
  --set global.ingress.configureCertmanager=false \
  --set gitlab.unicorn.ingress.tls.secretName=release-gitlab-tls \
  --set registry.ingress.tls.secretName=release-registry-tls \
  --set minio.ingress.tls.secretName=release-minio-tls \
  --set global.ingress.annotations."cert-manager\.io\/cluster-issuer"=cluster-issuer \
  --set global.registry.certificate.secret=${release}-registry-secret \
  --set global.email.display_name=hekai \
  --set global.email.from=hekai@abc.com \
  --set global.email.reply_to=hekai@abc.com \
  --set global.email.subject_suffix=@abc.com \
  --set global.smtp.enabled=true \
  --set global.smtp.address=smtp.abc.com \
  --set global.smtp.port=465 \
  --set global.smtp.authentication=login \
  --set global.smtp.openssl_verify_mode=none \
  --set global.smtp.starttls_auto=true \
  --set global.smtp.tls=true \
  --set global.smtp.password.key=gitlab-outgoing-smtp-pwd-key \
  --set global.smtp.password.secret=gitlab-outgoing-smtp-pwd-secret \
  --set global.smtp.user_name=hekai@abc.com \
  --set certmanager.install=false \
  --set nginx-ingress.enabled=false \
  --set prometheus.install=false \
  --namespace gitlab \
  > gitlab.yaml
```

再配置下接收邮件，接收邮件的作用如下：
<https://docs.gitlab.com/ee/administration/incoming_email.html>

```console
GitLab has several features based on receiving incoming emails:

    Reply by Email: allow GitLab users to comment on issues and merge requests by replying to notification emails.
    New issue by email: allow GitLab users to create a new issue by sending an email to a user-specific email address.
    New merge request by email: allow GitLab users to create a new merge request by sending an email to a user-specific email address.
    Service Desk: provide e-mail support to your customers through GitLab.
```

日常用的应该不多。

incomingEmail 使用的端口为 143，不使用ssl 的，测试过程中，若使用993端口，找不到证书。

```console
[root@k8s-m1 ~]# kubectl logs -n gitlab -f mygitlab-mailroom-7b9d9c9cc-c9g2p
+ /scripts/set-config /etc /etc
Begin parsing .erb files from /etc
+ exec /bin/sh -c '/usr/bin/mail_room -c /var/opt/gitlab/mail_room.yml'
/usr/lib/ruby/2.6.0/net/protocol.rb:44:in `connect_nonblock': SSL_connect returned=1 errno=0 state=error: certificate verify failed (error number 1) (OpenSSL::SSL::SSLError)
	from /usr/lib/ruby/2.6.0/net/protocol.rb:44:in `ssl_socket_connect'
	from /usr/lib/ruby/2.6.0/net/imap.rb:1534:in `start_tls_session'
```

```console
helm template . \
  --name ${release} \
  --set global.edition=ce \
  --set global.time_zone=Beijing \
  --set global.hosts.domain=k8s.abc \
  --set global.grafana.enabled=false \
  --set global.ingress.class=nginx \
  --set global.ingress.configureCertmanager=false \
  --set gitlab.unicorn.ingress.tls.secretName=release-gitlab-tls \
  --set registry.ingress.tls.secretName=release-registry-tls \
  --set minio.ingress.tls.secretName=release-minio-tls \
  --set global.ingress.annotations."cert-manager\.io\/cluster-issuer"=cluster-issuer \
  --set global.registry.certificate.secret=${release}-registry-secret \
  --set global.email.display_name=hekai \
  --set global.email.from=hekai@abc.com \
  --set global.email.reply_to=hekai@abc.com \
  --set global.email.subject_suffix=@abc.com \
  --set global.smtp.enabled=true \
  --set global.smtp.address=smtp.abc.com \
  --set global.smtp.port=465 \
  --set global.smtp.authentication=login \
  --set global.smtp.openssl_verify_mode=none \
  --set global.smtp.starttls_auto=true \
  --set global.smtp.tls=true \
  --set global.smtp.password.key=gitlab-outgoing-smtp-pwd-key \
  --set global.smtp.password.secret=gitlab-outgoing-smtp-pwd-secret \
  --set global.smtp.user_name=hekai@abc.com \
  --set global.appConfig.incomingEmail.enabled=true \
  --set global.appConfig.incomingEmail.host=imap.abc.com \
  --set global.appConfig.incomingEmail.address=hekai@abc.com \
  --set global.appConfig.incomingEmail.port=143 \
  --set global.appConfig.incomingEmail.password.key=gitlab-incoming-imap-pwd-key \
  --set global.appConfig.incomingEmail.password.secret=gitlab-incoming-imap-pwd-secret \
  --set global.appConfig.incomingEmail.ssl=false \
  --set global.appConfig.incomingEmail.startTls=false \
  --set global.appConfig.incomingEmail.user=hekai@abc.com \
  --set certmanager.install=false \
  --set nginx-ingress.enabled=false \
  --set prometheus.install=false \
  --namespace gitlab \
  > gitlab.yaml
```

修改 postgresql 和 gitlab-runner 相关资源对应的命名空间

不加 incoming email配置了，作用并不大
helm v3 将 `--name` 改为了 `--name-template`

```console
helm template . \
  --name-template ${release} \
  --set global.edition=ce \
  --set global.time_zone=Beijing \
  --set global.hosts.domain=k8s.abc \
  --set global.grafana.enabled=false \
  --set global.ingress.class=nginx \
  --set global.ingress.configureCertmanager=false \
  --set gitlab.unicorn.ingress.tls.secretName=release-gitlab-tls \
  --set registry.ingress.tls.secretName=release-registry-tls \
  --set minio.ingress.tls.secretName=release-minio-tls \
  --set global.ingress.annotations."cert-manager\.io\/cluster-issuer"=cluster-issuer \
  --set global.registry.certificate.secret=${release}-registry-secret \
  --set global.email.display_name=hekai \
  --set global.email.from=hekai@abc.com \
  --set global.email.reply_to=hekai@abc.com \
  --set global.email.subject_suffix=@abc.com \
  --set global.smtp.enabled=true \
  --set global.smtp.address=smtp.abc.com \
  --set global.smtp.port=465 \
  --set global.smtp.authentication=login \
  --set global.smtp.openssl_verify_mode=none \
  --set global.smtp.starttls_auto=true \
  --set global.smtp.tls=true \
  --set global.smtp.password.key=gitlab-outgoing-smtp-pwd-key \
  --set global.smtp.password.secret=gitlab-outgoing-smtp-pwd-secret \
  --set global.smtp.user_name=hekai@abc.com \
  --set certmanager.install=false \
  --set nginx-ingress.enabled=false \
  --set prometheus.install=false \
  --namespace gitlab \
  > gitlab.yaml
```

根据 doc/installation/storage.md 中描述，建议 storageclass 的 reclaimPolicy 使用 retain 模式

```console
The default storage class should:

- Use fast SSD storage when available
- Set `reclaimPolicy` to `Retain`

> Uninstalling GitLab without the `reclaimPolicy` set to `Retain` allows automated jobs to completely delete the volume, disk and data.
> Some platforms set the default `reclaimPolicy` to `Delete`. The `gitaly` persistent volume claims do not follow this rule because
> they belong to a [StatefulSet][].
```

由于 上文创建的 managed-nfs-storage 的默认 reclaimPolicy 使用的是 Delete 模式，并且已经有 一些 容器使用 这个 storageclass 了，所以可以新建一个 nfs storage class 来支持 retain 模式，并把该 storage class 置为默认。

```console
[root@k8s-m1 deploy]# cat class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: nfs-storage # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
[root@k8s-m1 deploy]# cat class-retain.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage-retain
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" # 将该sc配置为默认的sc
provisioner: nfs-storage # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
# Supported policies: Delete, Retain
reclaimPolicy: Retain
[root@k8s-m1 deploy]# kubectl apply -f class.yaml -f class-retain.yaml
```

安装 helm v3

1. 下载 helm v3 版本，下载地址：<https://github.com/helm/helm/releases>
2. 将下载得到的 tar 包解压

    ```console
    tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
    ```

3. 将 helm 二进制文件放到可执行目录中。

   ```console
   mv linux-amd64/helm /usr/local/bin/helm
   ```

helm v3 取消了 tiller，所以不需要安装 服务端的tiller， helm 默认从 `~/.kube/config` 读取与k8s的连接信息。

https://helm.sh/docs/intro/quickstart/

添加仓库，官方仓库需要翻墙，可以使用微软提供的镜像
参考：https://github.com/BurdenBear/kube-charts-mirror

```
[root@k8s-m1 ~]# helm repo add stable https://kubernetes-charts.storage.googleapis.com/
"stable" has been added to your repositories
[root@k8s-m1 ~]# helm repo remove stable
"stable" has been removed from your repositories
[root@k8s-m1 ~]# helm repo add stable http://mirror.azure.cn/kubernetes/charts/
"stable" has been added to your repositories
[root@k8s-m1 ~]# helm repo add incubator http://mirror.azure.cn/kubernetes/charts-incubator/
"incubator" has been added to your repositories
[root@k8s-m1 ~]# 
```


### external-dns



debian中使用 ip 命令
apt install iproute2

容器内获取物理机ip
https://blog.csdn.net/kozazyh/article/details/79463688
https://github.com/kubernetes/kubernetes/issues/24657
https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/

1. 使用 hostNetwork
2. 使用环境变量的方式


coredns 的 hosts 插件无法动态更新 /etc/hosts里的地址
