# Vscode的使用


1. 使用 vscode 的过程中，发现对 yaml 格式的文本进行 tab ，之后内容的格式会发生变化。

    如对下面的 yaml 文本

    ```yaml
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

    使用 tab 键后，结果如下

    ```yaml
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

    解决方法：  
    进入 File - Preferences - Settings ，搜索 `tab` ，将 `Editor: Use Tab Stops` 关掉 或者
    直接在 `settings.json` 中添加 `"editor.useTabStops": false`

2. 缩进转为空格

    将文本复制到 vscode 中时，原文本中的 tab 不会自动转为空格，可以点击 shift+control+p ，然后输入 convert ，
    查找 convert Indention to spaces，可以批量地将当前文本中的 tab 转为空格。

3. markdown 中输入上标和下标

    上标输出 n 的平方有两种方法：
    - n^2^  ```n^2^```
    - n<sup>2</sup>  ```n<sup>2</sup>```

    下标输出 n 下标2有两种方法：
    - n~2~  ```n~2~```
    - n<sub>2</sub>  ```n<sub>2</sub>```

    vscode 自带的 preview 无法显示，需要安装扩展 Markdown Preview Enhanced ，然后使用其 preview 进行预览。

4. vscode 添加代码分割线（垂直标尺）

    在 settings.json 中添加 `"editor.rulers": [100]`

5. powershell core

    在使用 vscode 默认的 terminal 时，会弹出一个提示，如下：

    ```text
    Try the new cross-platform PowerShell https://aka.ms/pscore6
    ```

    尝试新的跨平台的 powershell。

    最初，Windows PowerShell 是在 .NET Framework 基础之上构建而成，仅适用于 Windows 系统。 在最新版本中，
    PowerShell Core 使用 .NET Core 2.x 作为运行时。 PowerShell Core 支持 Windows、macOS 和 Linux 平台。

    那就安装下这个跨平台版本的 powershell，项目地址位于 github <https://github.com/powershell/powershell>

    可以从该 github 的 release 页面下载最新的发布版本进行安装。

    安装后，点击 vscode 的终端界面（使用 crtl + ~ 打开），选择终端界面右侧终端选择的下拉框中的 `选择默认Shell` ，
    选择新安装的 Power Shell Core 即可，选择了之后，会在 settings.json 中自动添加如下配置

    ```text
    "terminal.integrated.shell.windows": "C:\\Program Files\\PowerShell\\6\\pwsh.exe",
    ```

6. vscode 的缩进空格数

    发现 md 文档中，某些 md 文档的缩进空格数是 4 ，某些是 2 ，为什么呢？通过测试，发现如果 md 文档中含有 yaml 格式的代码块，
    会将缩进空格数置为 2 。

    因为在 vscode 的设置中，启用了 `Detect Indentation` ，它会检测文件内容从而自动配置缩进空格数和tab转为空格，将其关闭即可。
    关闭后，会在 settings.json 中自动生成如下配置

    ```text
    "editor.detectIndentation": false,
    ```

