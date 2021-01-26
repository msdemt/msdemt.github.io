# Windos10安装wsl问题汇总


win10安装wsl2文档：https://docs.microsoft.com/zh-cn/windows/wsl/install-win10

win10安装 wsl2 后，就可以在商店里直接安装linux主机了，不用再借助vmware或virtualbox之类的虚拟化软件了

在商店里安装ubuntu。

1. ubuntu不允许使用用户名密码远程连接

    解决：编辑/etc/ssh/sshd_config，去除`PasswordAuthentication yes`前面的注释，然后重启sshd（service ssh restart）

2. ubuntu上root密码为空，使用如下命令配置root用户密码

    ```shell
    sudo passwd root
    ```

3. 无法使用root用户远程连接ubuntu

    解决：编辑/etc/ssh/sshd_config，在`#PermitRootLogin prohibit-password`一行下手动加入`PermitRootLogin yes`，然后重启ssh（service ssh restart）

4. ssh启动失败

    ```shell
    root@HK-DESKTOP:/etc/netplan# vi /etc/ssh/sshd_config
    root@HK-DESKTOP:/etc/netplan# service ssh start
    * Starting OpenBSD Secure Shell server sshd                                                                            sshd: no hostkeys available -- exiting.
                                                                                                                    [fail]
    ```

    解决：执行 `ssh-keygen -A` 命令

    ```shell
    root@HK-DESKTOP:/etc/netplan# ssh-keygen -A
    ssh-keygen: generating new host keys: RSA DSA ECDSA ED25519
    root@HK-DESKTOP:/etc/netplan# service ssh start
    * Starting OpenBSD Secure Shell server sshd                                                                     [ OK ]
    ```

5. wsl ubuntu 的ip是会动态变化的，可以在win10上使用127.0.0.1:22进行远程连接

6. wsl ubuntu 怎么关机？

    参考：https://blog.csdn.net/flynetcn/article/details/108133340

    以管理员运行如下命令
    
    ```
    net stop LxssManager
    ```

总结：可能是使用vmware习惯了，感觉wsl2 linux还是不太方便，没办法替代vmware。
