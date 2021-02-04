# Idea配置字符集编码


idea 安装后，第一件事应该就是配置下字符集编码，因为默认情况下，idea 新建某些文件时使用的是操作系统的默认编码，该编码可能为 GBK ，然而我们平常使用的编码是 UTF-8 。

gbk 编码的文件，使用 utf-8 打开后会出现中文乱码的问题。

安装 idea 后，在左栏中选择 Customize ，然后选择 All settings ，依次点击 Editor -> File Encodings -> 将本页的字符集编码修改为 UTF-8 ，然后保存。这样就配置好idea的默认项目的字符集编码了。新建的项目和新打开的项目都会使用该字符集编码。

如果，安装 idea 后，打开过某个项目A，那么该项目的字符集编码配置会保存在 .idea 文件夹中，当然，现在该项目的字符集编码应该为系统字符集编码，比如 GBK ，这不是我们期望的。
- 解决方式1：在idea中打开该项目，然后依次点击 File -> Settings -> Editor -> File Encodings -> 将本页的字符集编码修改为 UTF-8 ，然后保存。
- 解决方式2: 删除该项目下的 .idea 文件夹，然后使用上述方法配置 idea 的默认字符集编码，然后重新打开该项目

建议，同时配置下包自动导入，可以在开发中节省时间。在左栏中选择 Customize ，然后选择 All settings ，依次点击 Editor -> General -> Auto Import -> 选中 Add unambiguous import on the fly 和 Optimize imports on the fly

这个 All Settings 配置 也可以在已打开项目的 idea 页面 中 File -> New Projects Settings -> Settings For New Projects 进行配置。

建议，配置下 Annotation Processing ，lombok需要该功能，位置：File -> New Projects Settings -> Settings For New Projects -> Build, Execution, Deployment -> Compiler -> Annotation Processors -> 选中 Enable annotation processing -> Apply

