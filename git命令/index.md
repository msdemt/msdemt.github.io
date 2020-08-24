# Git命令


## git 配置代理

> 参考：https://gist.github.com/laispace/666dd7b27e9116faece6

配置 http 协议七层代理

```
# 添加代理
git config --global https.proxy http://127.0.0.1:1080
git config --global https.proxy https://127.0.0.1:1080
# 删除代理
git config --global --unset http.proxy
git config --global --unset https.proxy
```

## git 子模块的管理和使用

> 参考：https://www.jianshu.com/p/9000cd49822c

git 中可以使用子模块 `submodule` 管理不同的项目， `submodule` 允许你将一个 git 仓库当作另一个 git 仓库的子目录。

本文以 hugo 静态网页作为示例，运行如下命令新建一个名为 submoduleTest 的 hugo 静态网页项目

```
hugo new site submoduleTest
```

进入 submoduleTest 静态网页项目中

```
cd submoduleTest
```

将 submoduleTest 初始化为一个 git 项目

```
git init
```

### 添加子模块

将 `LoveIt` 主题作为子模块添加到 `themes` 文件夹下 

```
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

运行 `git status` ，可以看到目录增加了 `.gitmodules` 文件，这个文件用来保存子模块的信息。

```
PS D:\workspace\vscode\submoduleTest> git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   .gitmodules
        new file:   themes/LoveIt

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        archetypes/
        config.toml
```

为了方便管理基于 github 的 hugo 博客，可以将 public 目录作为 hugo 博客的子项目文件夹，比如

```
git submodule add https://github.com/msdemt/msdemt.github.io.git public
```

### 查看子模块

查看子模块命令

```
git submodule
```

示例：

```
PS D:\workspace\vscode\submoduleTest> git submodule
 23a52ef1030b2c2057aa0bb1da57b6247da645a5 themes/LoveIt (v0.2.6-4-g23a52ef)
```

### 更新子模块

- 更新项目内的子模块到上次同步的版本
    ```
    git submodule update
    ```

- 更新子模块为远程项目的最新版本
    ```
    git submodule update --remote
    ```
### 提交包含子模块的项目

登陆 github ，新建仓库 submoduleTest 

本地 git 仓库 submoduleTest 已经执行过 git init 初始化命令了，再依次执行如下命令将本地 submoduleTest项目提交到 github 仓库中。

```
git add .
git commit -m "first commit"
git remote add origin git@github.com:msdemt/submoduleTest.git
git push -u origin master
```

> 提交后发现 github 仓库中缺少目录，原因是 git 仅跟踪文件的变动，不跟踪目录。  
> 解决办法是再空目录下创建 .gitkeep 文件 
> .gitkeep 是一个约定俗成的文件名并不会带有特殊规则


### 克隆包含子模块的项目

克隆包含子模块的项目有二种方法：一种是先克隆父项目，再更新子模块；另一种是直接递归克隆整个项目。

- 克隆父项目，再更新子模块
  1. 克隆父项目
     ```
     git clone https://github.com/msdemt/submoduleTest.git
     ```
  2. 查看子模块
      ```
      PS D:\workspace> cd .\submoduleTest\
      PS D:\workspace\submoduleTest> git submodule
      -23a52ef1030b2c2057aa0bb1da57b6247da645a5 themes/LoveIt
      ```
      子模块前面有一个 `-` ，说明子模块还未检入（空文件夹）

   3. 初始化子模块
        ```
        git submodule init
        ```
        初始化模块只需在克隆父项目后运行一次。
    4. 更新子模块到上次同步的版本
        ```
        git submodule update
        ```

- 递归克隆整个项目，包括子模块

    ```
    git clone https://github.com/msdemt/submoduleTest.git --recursive
    ``` 

    若想将项目文件内容克隆到指定文件夹，则可以在 git clone url 后加上指定的文件夹名称，比如

    ```
    git clone https://github.com/msdemt/submoduleTest.git assets --recursive
    ```

### 修改子模块

在子模块文件夹中修改文件后，在子模块文件中中执行如下命令提交到子模块对应的远程项目分支

```
git add .
git commit -m "commit"
git push origin master
```

### 删除子模块

删除子模块比较麻烦，需要手动删除相关的文件，否则在添加子模块时有可能出现错误  

1. 删除 `LoveIt` 子模块文件夹
    ```
    git rm --cache themes/LoveIt 
    rm -rf themes/LoveIt
    ```
2. 删除 `.gitmodules` 文件中相关子模块信息
    ```
    [submodule "themes/LoveIt"]
        path = themes/LoveIt
        url = https://github.com/dillonzq/LoveIt.git
    ```
3. 删除 `.git/config` 中相关子模块信息
    ```
    [submodule "themes/LoveIt"]
        url = https://github.com/dillonzq/LoveIt.git
    ```
4. 删除 .git 文件夹中的相关子模块文件
   ```
   rm -rf .git/modules/themes/LoveIt
   ```

## git 的回滚

git 提交失败后回滚到上一版本

git log 查看历史本信息
```
PS C:\Users\He\OneDrive\Documents\vscode\mysite> git log
commit 135e549469efe632297600fbb22e4eb9f59e8ee3 (HEAD -> master, origin/master, origin/HEAD)
Author: msdemt <276516848@qq.com>
Date:   Sun Jun 7 10:20:47 2020 +0800

    提交更新

commit b1c4b5812fd787f3b1ad3dbb95f382f325b5b983
Author: msdemt <276516848@qq.com>
Date:   Wed Jun 3 21:45:17 2020 +0800

    java基础
```

git reset --hard 历史id 回滚到某个id版本
```
PS C:\Users\He\OneDrive\Documents\vscode\mysite> git reset --hard b1c4b5812fd787f3b1ad3dbb95f382f325b5b983    
HEAD is now at b1c4b58 java基础
```
提交到github
```
git push -f origin master
```
回滚后如果想再次恢复上一版本
```
git reflog
```

```
PS C:\Users\He\OneDrive\Documents\vscode\mysite> git reflog
b1c4b58 (HEAD -> master, origin/master, origin/HEAD) HEAD@{0}: reset: moving to b1c4b5812fd787f3b1ad3dbb95f382f325b5b983
135e549 HEAD@{1}: commit: 提交更新
b1c4b58 (HEAD -> master, origin/master, origin/HEAD) HEAD@{2}: clone: from https://github.com/msdemt/mysite.git


使用 git reset --hard 历史id 
PS C:\Users\He\OneDrive\Documents\vscode\mysite> git reset --hard 135e549
HEAD is now at 135e549 提交更新
```
