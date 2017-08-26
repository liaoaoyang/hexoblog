title: CentOS 6.x 源码安装 git
date: 2016-04-21 00:13:36
categories: Try
tags: [git, CentOS]
---

# 概述

git 在各个发行版的包管理器中应该都有包含，一般版本也不够新，某些情况下如果需要安装特殊版本的 git 还是需要通过源码进行编译。

<!-- more -->

# 方法

个人常用的发行版是 `CentOS 6.x`，那么这一版本的 git 在 yum 中只是 1.7.1。

想要尝鲜看来就得需要通过源码安装了。

鉴于官方文档已经说得很详细了，一般安装中都不会遇到太多的问题，但是在 `CentOS 6.x` 上出现了找不到 `libiconv` 的情况。

```
libgit.a(utf8.o): In function `reencode_string_iconv':
/opt/files/git/utf8.c:463: undefined reference to `libiconv'
libgit.a(utf8.o): In function `reencode_string_len':
/opt/files/git/utf8.c:502: undefined reference to `libiconv_open'
/opt/files/git/utf8.c:521: undefined reference to `libiconv_close'
/opt/files/git/utf8.c:515: undefined reference to `libiconv_open'
```

出现这个问题并不合理，因为 CentOS 5.4+ 的系统就已包含了。

可能的原因就是没有正确的找到这一 lib 的准确位置。从 GitHub 中下载的 git 源码，从 Makefile 中可以看到：

```
ifdef NEEDS_LIBICONV
	ifdef ICONVDIR
		BASIC_CFLAGS += -I$(ICONVDIR)/include
		ICONV_LINK = -L$(ICONVDIR)/$(lib) $(CC_LD_DYNPATH)$(ICONVDIR)/$(lib)
	else
		ICONV_LINK =
	endif
	ifdef NEEDS_LIBINTL_BEFORE_LIBICONV
		ICONV_LINK += -lintl
	endif
	EXTLIBS += $(ICONV_LINK) -liconv
endif
```

直接 make 的情况下，我们并没有指定 `ICONVDIR`，那么需要做的就是就是正确指定 `libiconv` 的目录了。

git 很贴心的提供了一个 `make configure` 选项，可以用于重新配置编译参数。

通过 `./configure --help` 可以看到：

```
  --with-iconv=PATH       PATH is prefix for libiconv library and headers
                          used only if you need linking with libiconv
```

这一参数可以指定 `libiconv` 的位置。

那么后续的步骤自然是简单了：

```
./configure --with-iconv=/usr/local/
make all
make install
```

以上。


