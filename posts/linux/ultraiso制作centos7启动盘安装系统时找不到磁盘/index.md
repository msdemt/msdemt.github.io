# Ultraiso制作centos7启动盘安装系统时找不到磁盘


> 参考：
> https://blog.csdn.net/ouyangxs/article/details/84193077  
> https://blog.csdn.net/weixin_30897233/article/details/96292006  
> https://jingyan.baidu.com/article/ab0b563061fa84c15afa7d8c.html


使用 ultraiso 制作 centos7 u 盘启动盘后，使用 u 盘启动盘安装 centos7 系统，出现如下的错误：

![](/images/20200604-u盘安装centos7系统找不到设备1.png)

![](/images/20200604-u盘安装centos7系统找不到设备2.png)

原因：

使用 ultraiso 制作的 centos7 u 盘启动盘的卷标是 `CentOS 7 x8`，然而 centos7 系统镜像中的 `EFI/BOOT/grub.cfg` 文件中卷标写的是 `CentOS\x207\x20x86_64`，`isolinux/isolinux.cfg` 文件中卷标写的是 `CentOS\x207\x20x86_64` ，`isolinux/syslinux.cfg` 文件中卷标写的是 `CentOS\x207\x20x8`

![](/images/20200604-u盘启动盘卷标.png)

Win 系统下 fat32 分区卷的信息只能写入 11 字符而且不可以有 `\` 字符，而且只能为大写字母，如果是小写字母会自动转为大写。


解决办法一：

修改 centos7 u 盘启动盘 `grub.cfg` 文件中的 `CentOS\x207\x20x86_64` 、`isolinux.cfg` 文件中的 `CentOS\x207\x20x86_64` 、`syslinux.cfg` 文件中 `CentOS\x207\x20x8` 统一修改为 `CENTOS7` 或者其他大写字母。

将 u 盘启动盘的盘符修改为 `CENTOS7` 或者其他大写字母，与上面的修改保持一致。

然后就可以使用该 u 盘启动盘安装 centos7 系统了。

![](/images/20200604-u盘启动盘grub.cfg文件对比.png)

![](/images/20200604-u盘启动盘isolinux.cfg文件对比.png)

![](/images/20200604-u盘启动盘syslinux.cfg文件对比.png)

![](/images/20200604-修改盘符为CENTOS7.png)


解决方法二：

参考：

https://jingyan.baidu.com/article/ab0b563061fa84c15afa7d8c.html
