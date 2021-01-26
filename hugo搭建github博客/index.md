# Hugo搭建GitHub博客


## win10 安装 Hugo

1. 在 [Hugo Release](https://github.com/gohugoio/hugo/releases) 页面下载对应操作系统的 Hugo 二进制文件。  
    如 win10 64 位系统下载 hugo_0.56.3_Windows-64bit.zip ，将该文件解压到指定目录，如 `D:\Develop\Hugo\bin` 。  
2. 配置 Hugo 环境变量  
    win10 系统变量 `PATH` 添加 `D:\Develop\Hugo\bin`  

## 使用 Hugo 创建静态网页

1. 进入期望存储项目的目录，如 `D:\workspace\vscode\git` ，在该目录打开终端（ windows 下为 cmd 或 powershell）进行操作，或者使用 vscode 打开期望存储项目的目录，点击 `ctrl+~` 打开 terminal ，进行操作

2. 在当前目录生成名为 mysite 的站点  
    命令如下， mysite 为生成的项目名

    ```shell
    hugo new site mysite
    ```

    执行后内容如下：

    ```shell
    PS D:\workspace\vscode\git> hugo new site mysite
    Congratulations! Your new Hugo site is created in D:\workspace\vscode\git\mysite.

    Just a few more steps and you're ready to go:

    1. Download a theme into the same-named folder.
    Choose a theme from https://themes.gohugo.io/ or
    create your own with the "hugo new theme <THEMENAME>" command.
    2. Perhaps you want to add some content. You can add single files
    with "hugo new <SECTIONNAME>\<FILENAME>.<FORMAT>".
    3. Start the built-in live server via "hugo server".

    Visit https://gohugo.io/ for quickstart guide and full documentation.
    ```

3. 进入 `mysite` 站点，查看生成的项目目录结构  

    ```shell
    PS D:\workspace\vscode\git> cd .\mysite\
    PS D:\workspace\vscode\git\mysite> ls


        目录: D:\workspace\vscode\git\mysite


    Mode                LastWriteTime         Length Name
    ----                -------------         ------ ----
    d-----       2019-08-01     11:25                archetypes
    d-----       2019-08-01     11:25                content
    d-----       2019-08-01     11:25                data
    d-----       2019-08-01     11:25                layouts
    d-----       2019-08-01     11:25                static
    d-----       2019-08-01     11:25                themes
    -a----       2019-08-01     11:25             82 config.toml
    ```

    - **config.toml** 是网站的配置文件，包括 baseurl， title， copyright 等等网站参数  
    - **archetypes**：包括内容类型，在创建新内容时自动生成内容的配置
    - **content**：包括网站内容，全部使用 markdown 格式  
    - **layouts**：包括了网站的模版，决定内容如何呈现  
    - **static**：包括了 css，js， fonts， media 等，决定网站的外观

4. 安装主题  
    Hugo 社区提供了一些可用的主题，使用如下命令将所有的主题下载到 `themes` 目录下

    ```shell
    git clone --depth 1 --recursive https://github.com/gohugoio/hugoThemes.git themes
    ```

    或者下载特定的主题，使用如下命令下载 beautifulhugo 主题到 `themes/beautifulhugo` 目录

    ```shell
    git clone https://github.com/halogenica/beautifulhugo.git themes/beautifulhugo
    ```

    将 `themes/beautifulhugo/exampleSite/config.toml` 文件内容拷贝到 `mysite` 目录中，覆盖原文件内容。


5. 创建新页面  

    ```shell
    PS D:\workspace\vscode\git\mysite> hugo new post/about.md
    D:\workspace\vscode\git\mysite\content\post\about.md created
    ```

    进入 `content/` 目录，可以看到多了一个 `post` 目录，里面有新建的 `about.md` 文件。打开文件可以看到时间和文件名等信息已经自动加到文件开头，包括标题、创建时间、是否为草稿等。

    about.md 中添加 Hello Hugo! ，添加后内容如下所示：

    ```shell
    PS D:\workspace\vscode\git\mysite> cat .\content\post\about.md
    ---
    title: "About"
    date: 2019-08-01T13:11:32+08:00
    draft: true
    ---

    Hello Hugo!
    ```

    --- 标记的内容是 YAML 格式的，可以换成 TOML 格式（使用 +++ 标记）或者 JSON 格式。  
    **hugo new** 命令生成的文章前面包括的那几行，是用来设置文章属性的，这些属性使用的是 yaml 语法。  
    - **title** 设置文章第一级标题，一个 md 文档中只能有一个一级标题。
    - **date** 时间标签，页面上默认显示 n 篇最新的文章。  
    - **draft** 设置为 false 的时候会被编译为 HTML ， true 则不会编译和发表。
    - **tags** 数组，可以设置多个标签，逗号隔开， Hugo 会自动在你博客主页下生成标签的子 URL ，通过这个 URL 可以看到所有具有该标签的文章。
    - **categories** 文章分类，跟 Tag 功能差不多，只能设置一个字符串。

    更多详见： <https://gohugo.io/content-management/front-matter/>  

6. 运行 hugo  

    ```shell
    hugo server -t beautifulhugo --buildDrafts
    ```

    -t 参数的意思是使用特定主题渲染页面，注意到 about.md 目前是作为草稿，即 draft 参数设置为 true ，运行 Hugo 时要加上 `--buildDrafts` 参数才会将标记为草稿的页面生成  HTML 。  
    在浏览器输入 <http://localhost:1313> ，就可以看到我们刚刚创建的页面。

    ```shell
    PS D:\workspace\vscode\git\mysite> hugo server -t beautifulhugo --buildDrafts
    Building sites …
                    | EN
    +------------------+-----+
    Pages            |  10
    Paginator pages  |   0
    Non-page files   |   0
    Static files     | 183
    Processed images |   0
    Aliases          |   2
    Sitemaps         |   1
    Cleaned          |   0

    Total in 2581 ms
    Watching for changes in D:\workspace\vscode\git\mysite\{archetypes,content,data,layouts,static,themes}
    Watching for config changes in D:\workspace\vscode\git\mysite\config.toml
    Environment: "development"
    Serving pages from memory
    Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
    Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
    Press Ctrl+C to stop
    ```

## 将静态网页部署到 GitHub  

首先在 GitHub 上创建一个 Repository ，命名为 `msdemt.github.io` （ msdemt 替换为你的 github 用户名）。

修改 `mysite/config.toml` 文件， `baseurl` 配置为自己的 `github` 用户名，如本文的配置为  

```bash
baseurl = "https://msdemt.github.io"
```

在站点根目录执行 `Hugo` 命令生成最终页面：

```shell
hugo --theme=beautifulhugo
```

如果一切顺利，所有 `draft` 为 false 的静态页面都会生成到 `public` 目录，将 `pubilc` 目录里所有文件 push 到刚创建的 Repository 的 master 分支。

在 `git shell` 或 `cmd` 中执行如下命令将 `public` 中的内容提交到 github 中

```shell
cd public
git init
git remote add origin https://github.com/msdemt/msdemt.github.io.git
git add -A
git commit -m "init"
git push -u origin master
```

详细步骤如下：

```shell
PS D:\workspace\vscode\git\mysite> hugo --theme=beautifulhugo
Building sites …
                | EN
+------------------+-----+
Pages            |  10
Paginator pages  |   0
Non-page files   |   0
Static files     | 183
Processed images |   0
Aliases          |   2
Sitemaps         |   1
Cleaned          |   0

Total in 4167 ms
PS D:\workspace\vscode\git\mysite> cd .\public\
PS D:\workspace\vscode\git\mysite\public> git init
Initialized empty Git repository in D:/workspace/vscode/git/mysite/public/.git/
PS D:\workspace\vscode\git\mysite\public> git remote add origin https://github.com/msdemt/msdemt.github.io.git
PS D:\workspace\vscode\git\mysite\public> git add -A
PS D:\workspace\vscode\git\mysite\public> git commit -m "init"
[master (root-commit) f5d1d02] init
196 files changed, 23076 insertions(+)
create mode 100644 404.html
create mode 100644 categories/index.html
create mode 100644 categories/index.xml
create mode 100644 css/bootstrap.min.css
create mode 100644 css/codeblock.css
create mode 100644 css/fonts.css
... ...
create mode 100644 tags/index.html
create mode 100644 tags/index.xml
PS D:\workspace\vscode\git\mysite\public> git push -u origin master
Counting objects: 216, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (210/210), done.
Writing objects: 100% (216/216), 4.70 MiB | 156.00 KiB/s, done.
Total 216 (delta 10), reused 0 (delta 0)
remote: Resolving deltas: 100% (10/10), done.
To https://github.com/msdemt/msdemt.github.io.git
* [new branch]      master -> master
Branch 'master' set up to track remote branch 'master' from 'origin'.
```

## 实战

也可以使用子模块的方式安装主题，
如安装 loveit 主题到 themes 目录下：
```
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```
安装 zozo 主题到 themes 目录下
```
git submodule add  https://github.com/varkai/hugo-theme-zozo.git themes/zozo
```

为了便于开发和部署，可以将 hugo 生成的静态网页存放的 public 目录作为 github 静态网页仓库的子模块
如：
```
git submodule add https://github.com/msdemt/msdemt.github.io.git public
```

下载mysite项目，同时下载依赖的子模块
```
git clone https://github.com/msdemt/mysite.git --recursive
``` 

将子模块更新到最新版本

```
git submodule update --remote
```

进入 public 目录，执行如下命令，执行了之后才可以提交该子模块的内容

```
cd public
git checkout origin master
```

进入mysite目录，增加 博客文章，之后执行 hugo 命令生成博客内容

```
cd ..
hugo
```

在进入public 目录，将博客内容提交到github

```
cd public
git add .
git commit -m "test"
git push origin master
```

再进入mysite目录，更新子模块public到最新版本

```
cd ..
git submodule update --remote
```

然后将mysite项目提交到github

```
git add .
git commit -m "test"
git push origin master
```


## 问题汇总

**问题1：**  
生成最终页面时指定了 baseUrl ，将页面部署到 github 上时，页面出错

解决：baseUrl 在 hugo 命令是指定或在 config.toml 中指定一次即可。

```shell
hugo --theme=beautifulhugo --baseUrl="http://msdemt.github.io/"
```

![image](../../images/hugo搭建github异常界面1展示.png)
![image](../../images/hugo搭建github异常界面1错误信息.png)

**问题2：**  
生成最终页面时未指定baseUrl，同时也未在mysite/config.toml文件中指定baseUrl时，将页面部署到github上，页面出错  

解决：baseUrl 在 hugo 命令是指定或在 config.toml 中指定一次即可。

```shell
hugo --theme=beautifulhugo
```

![image](../../images/hugo搭建github异常界面2错误信息.png)

**问题3：**

默认 Hugo 生成的 HTML 页面只有两级标题，对于多级标题的文章，生成标题目录（ tables of content）后会发现本来为三级标题但是显示的是二级标题。

github 上的大佬已经有了解决办法，详见：
<https://github.com/gohugoio/hugo/issues/1778>
<https://gist.github.com/pyrrho/1d77cdb98ba58c7547f2cdb3fb325c62>

解决办法：

1. 将 tables-of-contents.html 放入 themes/beautifulhugo/layouts/partials 目录下，作者认为 4 级标题就够用了，所以配置了最高级别 4 ，下面的文件中配置为了 6 ，tables-of-contents.html 的内容如下：

    ```html
    {{/* This partial was derived from the conversations in and around
        https://github.com/gohugoio/hugo/issues/1778 -- most directly skyzyx's
        gist; https://gist.github.com/skyzyx/a796d66f6a124f057f3374eff0b3f99a
    */}}

    {{/* Minimum heading level to include; default is h2. */}}
    {{- $minLevel := 2 -}}
    {{/* Minimum heading level to include; default is h4. */}}
    {{- $maxLevel := 6 -}}
    {{/* Search for headings as specified by $minLevel and $maxLevel, ignoring those
        that contain no text (ex; "<h2></h2>" will be ignored).
    */}}
    {{- $regex := printf "<h[%d-%d].*?>(.|\n])+?</h[%d-%d]>" $minLevel $maxLevel $minLevel $maxLevel -}}
    {{- $headings := findRE $regex .Content -}}

    {{/* Skip generation if there are no (suitable) headings. */}}
    {{- $hasHeadings  := ge (len $headings) 1 -}}
    {{- if $hasHeadings -}}
    <nav class="enclosing-block" class="customize-as-needed">
    {{- .Scratch.Set "toc__last-level" (sub $minLevel 1) -}}
    {{- range $heading := $headings -}}
        {{- $headingLevel   := substr $heading 2 1 | int -}}
        {{- $headingID      := index (findRE "id=.([a-z0-9-_])*" $heading) 0 | after 4 -}}
        {{- $headingTextRaw := substr (index (findRE ">.*</h" $heading 1) 0) 1 -3 -}}
        {{/* There may be an anchor tag wrapping some or all of the heading text.
            We don't want those tags in our ToC text, so lets get rid of them. */}}
        {{- $headingText    := replaceRE "</?a.*?>" "" $headingTextRaw | markdownify }}
        {{- $lastLevel := $.Scratch.Get "toc__last-level" -}}
        {{- $href := printf "#%s" $headingID -}}
        {{- $levelSeq := seq $lastLevel $headingLevel | after 1 -}}
        {{- $.Scratch.Set "toc__last-level" $headingLevel -}}

        {{- if gt $headingLevel $lastLevel -}}
        {{- range $l := $levelSeq -}}
            <ul class="table-of-contents__h{{ $l }}"><li>
        {{- end -}}
        {{- else if lt $headingLevel $lastLevel -}}
        {{- range $l := $levelSeq -}}
            </li></ul>
        {{- end -}}
        </li><li>
        {{- else -}}
        </li><li>
        {{- end -}}
        <a href="{{ $href }}">{{ $headingText }}</a>
    {{- end -}}

    {{- range seq ($.Scratch.Get "toc__last-level") (sub $minLevel 1) | after 1 -}}
        </li></ul>
    {{- end -}}
    </nav>
    {{- end -}}
    ```

2. 在 themes/beautifulhugo/layouts/_default/single.html 页面中加入

    ```html
    {{- partial "tables-of-contents.html" . -}}
    ```

    加入后 single.html 内容如下所示：

    ```html
    {{ define "main" }}
    <div class="container" role="main">
    <div class="row">
        <div class="col-lg-8 col-lg-offset-2 col-md-10 col-md-offset-1">
        <article role="main" class="blog-post">
            {{- partial "tables-of-contents.html" . -}}
            {{ .Content }}

            {{ if .Params.tags }}
            <div class="blog-tags">
                {{ range .Params.tags }}
                <a href="{{ $.Site.LanguagePrefix | absURL }}/tags/{{ . | urlize }}/">{{ . }}</a>&nbsp;
                {{ end }}
            </div>
            {{ end }}

            {{ if $.Param "socialShare" }}
                <hr/>
                <section id="social-share">
                <div class="list-inline footer-links">
                    {{ partial "share-links" . }}
                </div>
                </section>
            {{ end }}

            {{ if .Site.Params.showRelatedPosts }}
            {{ $related := .Site.RegularPages.Related . | first 3 }}
            {{ with $related }}
            <h4 class="see-also">{{ i18n "seeAlso" }}</h4>
            <ul>
            {{ range . }}
                <li><a href="{{ .RelPermalink }}">{{ .Title }}</a></li>
            {{ end }}
            </ul>
            {{ end }}
            {{ end }}
        </article>

        {{ if ne .Type "page" }}
            <ul class="pager blog-pager">
            {{ if .PrevInSection }}
                <li class="previous">
                <a href="{{ .PrevInSection.Permalink }}" data-toggle="tooltip" data-placement="top" title="{{ .PrevInSection.Title }}">&larr; {{ i18n "previousPost" }}</a>
                </li>
            {{ end }}
            {{ if .NextInSection }}
                <li class="next">
                <a href="{{ .NextInSection.Permalink }}" data-toggle="tooltip" data-placement="top" title="{{ .NextInSection.Title }}">{{ i18n "nextPost" }} &rarr;</a>
                </li>
            {{ end }}
            </ul>
        {{ end }}


        {{ if (.Params.comments) | or (and (or (not (isset .Params "comments")) (eq .Params.comments nil)) (and .Site.Params.comments (ne .Type "page"))) }}
            {{ if .Site.DisqusShortname }}
            {{ if .Site.Params.delayDisqus }}
            <div class="disqus-comments">
                <button id="show-comments" class="btn btn-default" type="button">{{ i18n "show" }} <span class="disqus-comment-count" data-disqus-url="{{ trim .Permalink "/" }}">{{ i18n "comments" }}</span></button>
                <div id="disqus_thread"></div>

                <script type="text/javascript">
                var disqus_config = function () {
                this.page.url = '{{ trim .Permalink "/" }}';
                };

            </script>
            </div>
            {{ else }}
            <div class="disqus-comments">
                {{ template "_internal/disqus.html" . }}
            </div>
            {{ end }}
            {{ end }}
            {{ if .Site.Params.staticman }}
            <div class="staticman-comments">
                {{ partial "staticman-comments.html" . }}
            </div>
            {{ end }}
        {{ end }}

        </div>
    </div>
    </div>
    {{ end }}
    ```

进行以上后改后，md 文档使用 hugo 生成 HTML 页面后会自动加入标题目录，不需要在 md 文档中再次加入标题目录了。

这种方法的缺点是会导致博客中所有的文章，如果有标题的话，都会自动生成标题目录，如果不希望这样的话，还有一个办法，是将正确的标题目录替换到 HTML 文件中。

