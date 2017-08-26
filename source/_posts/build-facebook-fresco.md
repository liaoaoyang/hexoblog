date: 2015-03-29 00:00:00
title: Build Facebook fresco
categories: Android
tags: [fresco,Docker]
---

Facebook最近开源了他们的Android图片加载库 [fresco][1] ，3月26号到现在两天多时间在github上收获了1000+ star，足见大家对这一个库的肯定。

自己自然也想尝试这一个库，首要工作就是build。

<!-- more -->

# 在OS X中进行build

这个库build过程中查看了`build.gradle`发现需要 [ndk][2] 支持，那么首要工作自然是安装ndk。

在OS X 10.9上的build过程比较简单，需要注意的是要把sdk以及ndk的位置加入`PATH`环境变量中，之后按照github上的`README.md`中的命令build即可，即：

```
git clone https://github.com/facebook/fresco.git
cd fresco
./gradlew build
```
build过程中可能出现的问题是中间有一步可能还需要挂代理（用到了 [chromium/webm/libwebp][3] ，gradle会执行一个clone操作），国内网络环境中可能会有连接不上的情况。


# Docker中进行build

搭建环境的繁琐之处程序员们自然体会了无数次，还好出现Docker，拯救了程序员。

这里为了方便大家，我简单的构建了一个 [Docker image][4] 用于方便大家build，基于 [dockerbase/android][5] 添加了support library。

使用过程很简单，自然是要先clone `fresco`：

```
docker run -i -t nginlion/fresco-build /bin/bash
git clone https://github.com/facebook/fresco.git
```

为了在此镜像中build `fresco` ，需要编辑根目录下的 `build.gradle` ，从:

```
buildscript {
    repositories {
        jcenter()
    }
    ...
```

变成：

```
buildscript {
    repositories {
        jcenter()
        mavenCentral() // 实际上只是添加了这一行
    }
    ...
```

之后继续使用命令 `./gradlew build`完成build工作, enjoy~





[1]: https://github.com/facebook/fresco
[2]: https://developer.android.com/tools/sdk/ndk/index.html
[3]: https://chromium.googlesource.com/webm/libwebp
[4]: https://registry.hub.docker.com/u/nginlion/fresco-build
[5]: https://registry.hub.docker.com/u/dockerbase/android/

