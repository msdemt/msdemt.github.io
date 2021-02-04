# Java代码混淆工具proguard问题记录



```xml
    <plugin>
        <groupId>com.github.wvengen</groupId>
        <artifactId>proguard-maven-plugin</artifactId>
        <version>2.3.1</version>
        <executions>
            <execution>
                <phase>package</phase>
                <goals>
                    <goal>proguard</goal>
                </goals>
            </execution>
        </executions>
        <configuration>
            <attach>true</attach>
            <obfuscate>true</obfuscate>
            <attachArtifactClassifier>pg</attachArtifactClassifier>
            <options>
                <option>-printmapping '${project.build.directory}/${project.artifactId}.log'</option>
                <option>-ignorewarnings</option>
                <option>-dontshrink</option>
                <option>-dontoptimize</option>
                <option>-dontusemixedcaseclassnames</option>
                <option>-dontskipnonpubliclibraryclasses</option>
                <option>-dontskipnonpubliclibraryclassmembers</option>
                <option>-allowaccessmodification</option>
                <option>-useuniqueclassmembernames</option>
                <option>-keeppackagenames</option>
                <option>-adaptclassstrings</option>
                <option>-keepdirectories</option>
                <option>-keepattributes
                    Exceptions,InnerClasses,Signature,Deprecated,SourceFile,LineNumberTable,LocalVariable*Table,*Annotation*,Synthetic,EnclosingMethod
                </option>
                <option>-keepnames interface **</option>
                <!-- This option will save all original methods parameters in files
    defined in -keep sections, otherwise all parameter names will be obfuscate. -->
                <option>-keepparameternames</option>
                <option>-keepclassmembers class * {
                    @org.springframework.beans.factory.annotation.Autowired *;
                    @org.springframework.beans.factory.annotation.Value *;
                    @org.springframework.context.annotation.Bean *;
                    @org.springframework.beans.factory.annotation.Qualifier *;
                    @org.springframework.stereotype.Repository *;
                    @org.springframework.data.repository.NoRepositoryBean *;
                    }
                </option>
                <option>-keepclassmembers enum * { *; }</option>
                <option>-keep class * implements java.io.Serializable</option>

                <option>-keep class com.example.ykp.App { *; }</option>
                <option>-keepclassmembers public class * {void set*(***);*** get*();}</option>
            </options>

            <outjar>${project.build.finalName}-pg.jar</outjar>
            <libs>
                <!-- <lib>${java.home}/jmods/java.base.jmod</lib> -->
                <lib>${java.home}/lib/rt.jar</lib>
                <!-- <lib>${java.home}/lib/jce.jar</lib> -->
            </libs>

            <injar>classes</injar>
            <outputDirectory>${project.build.directory}</outputDirectory>
        </configuration>
    </plugin>
```


踩坑

```log
[proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.CompressIndexes]
 [proguard]       (https://www.guardsquare.com/proguard/manual/troubleshooting#duplicateclass)
 [proguard] Warning: com.taoyuanx.ca.ui.controller.ApiController: can't find referenced class javax.sql.rowset.serial.SerialException [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.Decompressor]
 [proguard] Warning: com.taoyuanx.ca.ui.controller.ApiController: can't find referenced class javax.sql.rowset.serial.SerialExceptionNote: duplicate definition of library class [jdk.internal.jimage.decompressor.ResourceDecompressor$StringsProvider]
 [proguard]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.ResourceDecompressor]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.ResourceDecompressorFactory]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.ResourceDecompressorRepository]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.SignatureParser$ParseResult]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.SignatureParser]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.StringSharingDecompressor]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.StringSharingDecompressorFactory]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.ZipDecompressor]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.decompressor.ZipDecompressorFactory]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageBufferCache$1]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageBufferCache$2]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageBufferCache$BufferReference]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageBufferCache]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageHeader]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageLocation]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageReader$Directory]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageReader$LinkNode]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageReader$Node]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageReader$Resource]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageReader$SharedImageReader$LocationVisitor]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageReader$SharedImageReader]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageReader]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageReaderFactory$1]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageReaderFactory]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageStream]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageStrings]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.ImageStringsReader]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.NativeImageBuffer$1]
 [proguard] Note: duplicate definition of library class [jdk.internal.jimage.NativeImageBuffer]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.ExplodedImage$PathNode]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.ExplodedImage]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtDirectoryStream$1]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtDirectoryStream]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileAttributes]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileAttributeView$1]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileAttributeView$AttrID]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileAttributeView]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileStore]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileSystem$1]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileSystem]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileSystemProvider$1]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileSystemProvider$JrtFsLoader]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtFileSystemProvider]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtPath$1]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtPath$2]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtPath]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.JrtUtils]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.SystemImage$1]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.SystemImage$2]
 [proguard] Note: duplicate definition of library class [jdk.internal.jrtfs.SystemImage]
 [proguard] Warning: there were 2 unresolved references to classes or interfaces.
 [proguard]          You may need to add missing library jars or update their versions.
 [proguard]          If your code works fine without the missing classes, you can suppress
 [proguard]          the warnings with '-dontwarn' options.
 [proguard]          (https://www.guardsquare.com/proguard/manual/troubleshooting#unresolvedclass)
 [proguard] Error: Please correct the above warnings first.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  13.817 s
```

问题
```
[proguard] Error: You have to specify '-keep' options if you want to write out kept elements with '-printseeds'.
```

解决：添加
```
<option>-keepnames interface **</option>
```
或
```
<option>-keepclassmembers enum * { *; }</option>
```
或
```
<option>-keep class * implements java.io.Serializable</option>
```
可以避免该问题


```
[proguard] Note: there were 3 references to unknown classes.
 [proguard]       You should check your configuration for typos.
 [proguard]       (https://www.guardsquare.com/proguard/manual/troubleshooting#unknownclass)
 [proguard] Note: there were 4 unkept descriptor classes in kept class members.
 [proguard]       You should consider explicitly keeping the mentioned classes
 [proguard]       (using '-keep').
 [proguard]       (https://www.guardsquare.com/proguard/manual/troubleshooting#descriptorclass)
```

https://blog.csdn.net/maosidiaoxian/article/details/84032387

```
[proguard] Note: there were 3 references to unknown classes.
```

这个错误的原因还有可能是规则里包含了代码里没有用过的方法，比如

```xml
<option>-keepclassmembers class * {
    @org.springframework.beans.factory.annotation.Autowired *;
    @org.springframework.beans.factory.annotation.Value *;
    @org.springframework.context.annotation.Bean *;
    @org.springframework.beans.factory.annotation.Qualifier *;
    @org.springframework.stereotype.Repository *;
    @org.springframework.data.repository.NoRepositoryBean *;
    }
</option>
```
代码里没有用Qualifier和NoRepositoryBean，就会提示2个引用不知道指向的class


