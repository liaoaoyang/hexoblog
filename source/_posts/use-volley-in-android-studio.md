date: 2015-03-20 00:00:00
title: Android Studio中使用Volley
categories: Android
tags: [Android Studio, Volley]
---

Android上的通信框架各种各样，比如 [android-async-http][1]，而最近同学们很多都推荐给我用Google家的 [Volley][2]。

# 生成volley aar

官网上的指导手册说明了安装的步骤，首先自然是要下载源码：

```
git clone https://android.googlesource.com/platform/frameworks/volley
```
然而在某些网络环境下，会出现SSL验证问题，这时候就需要暂时关闭git的SSL验证：

```
git config --global http.sslVerify false
```
重新clone完成之后即可。

简单看看clone出的目录结构：

```
➜  volley git:(master) ls
Android.mk           build.xml            pom.xml              proguard.cfg         src
build.gradle         custom_rules.xml     proguard-project.txt rules.gradle
```
可以看到这里提供了通过gradle构建的方式，由于已经安装的Android Studio，那么在

```
~/.gradle/wrapper/dists/gradle-2.2-bin/ca0flae0itb57he40lyj6fhpp/gradle-2.2/bin/
```
这样的目录下可以找到gradle的可执行文件，不同版本的gradle可能不相同，但是位置应该是类似的。

找到gradle之后自然是进行build工作，不过在build之前，需要注意的是需要临时设定一下`ANDROID_HOME`环境变量，指向SDK目录：

```
export ANDROID_HOME=~/Library/Android/sdk
```
同时还需要注意的是检查`build.gradle`文件中的`buildToolsVersion`为已安装的版本，即在SDK Manager中的Tools > Android SDK Build-tools中已安装的版本，目前配置文件中默认版本是`21.1.0`，可能与已安装的版本不同，如：

```
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:0.14.+'
    }
}

apply plugin: 'com.android.library'

android {
    compileSdkVersion 19
    buildToolsVersion = '21.1.2' //修改此处为对应的buildTools版本
}

apply from: 'rules.gradle'
```

之后进行build工作：

```
~/.gradle/wrapper/dists/gradle-2.2-bin/ca0flae0itb57he40lyj6fhpp/gradle-2.2/bin/gradle build
```

如果build成功，会在当前目录下的`build/outputs/aar`目录下找到debug和release的aar包。

# Android Studio引用Volley

在Android Studio中引用Volley的aar包在当前的`1.1.0`版本中是可以按照如下方式进行的，即修改项目的`build.gradle`文件，添加对aar包的引用：

```
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:21.0.3'
    compile(name:'volley', ext:'aar')
}


repositories{
    flatDir{
        dirs 'libs'
    }
}
```

在此之前，应该已经将`volley-release.aar`复制到项目的`libs`目录中并改名为`volley.aar`了。

完成之后就是愉快的coding了。

[1]: http://loopj.com/android-async-http/
[2]: http://developer.android.com/training/volley/index.html
