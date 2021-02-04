# Vscode支持jdk8



https://blog.csdn.net/u014792301/article/details/107575799

https://github.com/redhat-developer/vscode-java/wiki/JDK-Requirements#java.configuration.runtimes


由于 Language Support for Java™ by Red Hat这个拓展更新到0.65.0，该扩展依赖eclipsejdt.ls服务器的，Eclipse平台决定将JDK11作为最低要求，所以该扩展不在支持jdk1.8了

解决方案：

在 `settings.json` 中添加 `java.configuration.runtimes` ，其中配置jdk1.8作为默认运行环境，同时依旧需要将 jdk11 作为 java home

如果安装了spring，还需要配置 `spring-boot.ls.java.home`，否则会出现 `Error trying to find JVM: TypeError: Cannot read property 'isJdk' of null`

电脑没有配置JAVA_HOME或不想使用电脑环境遍历里配置的JAVA_HOME，这种情况无法在vscode的终端中执行maven package等命令，若不希望在电脑环境遍历中配置JAVA_HOME，可以在settings.json中配置

```json
    "terminal.integrated.env.windows": {
        "JAVA_HOME": "D:\\Develop\\Java\\jdk1.8.0_202",
		"PATH": "D:\\Develop\\Java\\jdk1.8.0_202;${env:PATH}"
    },
```
这样，vscode的终端就使用了jdk1.8.0_202，而不是电脑配置的jdk
vscode执行maven package等命令用的也是jdk1.8.0_202


完整的java配置如下

```json
    "terminal.integrated.env.windows": {
        "JAVA_HOME": "D:\\Develop\\Java\\jdk1.8.0_202",
        "PATH": "D:\\Develop\\Java\\jdk1.8.0_202;${env:PATH}"
    },
    "terminal.integrated.env.osx": {
        "JAVA_HOME": "/usr/local/jdk/oraclejdk/jdk1.8.0_202.jdk/Contents/Home",
        "PATH": "/usr/local/jdk/oraclejdk/jdk1.8.0_202.jdk/Contents/Home:$PATH"
    },
    // java
    "java.home": "D:\\Develop\\openjdk\\jdk-14.0.1",
    "spring-boot.ls.java.home": "D:\\Develop\\Java\\jdk1.8.0_202",
    "maven.executable.path": "D:\\Develop\\Maven\\apache-maven-3.6.3\\bin\\mvn.cmd",
    "java.configuration.maven.userSettings": "C:\\Users\\He\\.m2\\settings.xml",
    "java.project.importOnFirstTimeStartup": "automatic",
    "java.configuration.checkProjectSettingsExclusions": false,
    "java.refactor.renameFromFileExplorer": "preview",
    "java.configuration.runtimes": [
        {
            "name": "JavaSE-1.8",
            "path": "D:\\Develop\\Java\\jdk1.8.0_202",
            "default": true
        },
        {
            "name": "JavaSE-11",
            "path": "D:\\Develop\\Java\\jdk-11.0.5"
        },
        {
            "name": "JavaSE-14",
            "path": "D:\\Develop\\openjdk\\jdk-14.0.1"
        }
    ],
    "java.templates.typeComment": [
        "/**",
        " * ${type_name}",
        " * @author: ${user}",
        " */"
    ],
    "java.codeGeneration.generateComments": true,
    "java.implementationsCodeLens.enabled": true,
    "java.referencesCodeLens.enabled": true,
    "java.maven.downloadSources": true,
    "java.saveActions.organizeImports": true,
    "java.showBuildStatusOnStart.enabled": true,
    "java.completion.guessMethodArguments": true,
    "java.dependency.showMembers": true,
    "java.dependency.packagePresentation": "flat",
    //lombok
    "java.jdt.ls.vmargs": "-XX:+UseParallelGC -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -Dsun.zip.disableMemoryMapping=true -Xmx1G -Xms100m -javaagent:\"c:\\Users\\He\\.vscode\\extensions\\gabrielbb.vscode-lombok-1.0.1\\server\\lombok.jar\"",
    "settingsSync.ignoredSettings": [
        "spring-boot.ls.java.home",
        "java.jdt.ls.vmargs",
        "java.configuration.maven.userSettings",
    ],
```
