title: Next主题中从服务器加载Google字体
date: 2017-02-07 23:48:58
categories: Try
tags: Hexo
---

# 概述

想要在Hexo的Next主题中使用Google字体，一个问题在于这个字体的加载并不是很方便（与Next主题本身无关）。

解决的方法可以是修改主题，在没有合适CDN的情况下，从服务器直接加载对应字体也是一种选择。

<!-- more -->
<!-- add-google-fonts-in-hexo-theme-next -->

# 实现

## 准备字体

首先，通过Chrome开发者工具，会发现难以加载的是这一css文件以及相关联的字体：

```
https://fonts.googleapis.com/css?family=Lato:300,400,700,400italic&subset=latin,latin-ext
```

当前使用的Next主题使用的是`Lato`这一个字体，那么应该如何得到这一字体呢？

这时候就需要`Google WebFont Downloader`了。

如果有Node环境，那么直接执行:

```
npm install -g goog-webfont-dl
```

即可。

使用上，可以通过`-h`查看参数定义：

```
→ goog-webfont-dl -h

  Usage: goog-webfont-dl [options] <fontname>

  Options:

    -h, --help                     output usage information
    -V, --version                  output the version number
    -t, --ttf                      Download TTF format
    -e, --eot                      Download EOT format
    -w, --woff                     Download WOFF format
    -W, --woff2                    Download WOFF2 format
    -s, --svg                      Download SVG format
    -a, --all                      Download all formats
    -f, --font [name]              Name of font
    -d, --destination [directory]  Save font in directory
    -o, --out [name]               CSS output file [use - for stdout]
    -p, --prefix [prefix]          Prefix to use in CSS output
    -u, --subset [string]          Subset string [e.g. latin,cyrillic]
    -y, --styles [string]          Style string [e.g. 300,400,300italic,400italic]
    -P, --proxy [string]           Proxy url [e.g. http://www.myproxy.com/]
    -q, --quiet                    Do not print status information
```

通过上面的url，可以确定使用的的字体为`Lato`，那么下载相同的字体需要指定的参数为：

```
goog-webfont-dl -f "Lato" -y "300,400,700,400italic" -u "latin,latin-ext" -a
```

这里`-a`参数保证下载了所有字体。

如果可以顺利的访问对应的资源，那么成功之后将会在当前目录出现字体名为名字的css以及字体目录。

## 修改主题

对应的域名是`googleapis`，可以考虑在Next主题的`vendors`目录下新增一个同名的目录，将css以及字体组织。目录层级会变为：

```
→ tree themes/next/source/vendors
themes/next/source/vendors
├── fancybox
    ...
├── fastclick
    ...
├── font-awesome
    ...
├── googleapis
│   ├── css
│   │   └── Lato.css
│   └── fonts
│       └── Lato
│           ├── Lato-Bold.ttf
│           ├── Lato-Bold.woff
│           ├── Lato-Bold.woff2
│           ├── Lato-Italic.ttf
│           ├── Lato-Italic.woff
│           ├── Lato-Italic.woff2
│           ├── Lato-Light.ttf
│           ├── Lato-Light.woff
│           ├── Lato-Light.woff2
│           ├── Lato-Regular.eot
│           ├── Lato-Regular.svg
│           ├── Lato-Regular.ttf
│           ├── Lato-Regular.woff
│           └── Lato-Regular.woff2
├── jquery
    ...
├── jquery_lazyload
    ...
└── velocity
    ...
```

剩下的工作是，在加载原先的css的`partials`中将url改成加载`verdors`中对应的地址即可：

```
{% if theme.use_font_lato %}
  <!--<link href="//fonts.googleapis.com/css?family=Lato:300,400,700,400italic&subset=latin,latin-ext" rel="stylesheet" type="text/css">-->
  <link href="{{ url_for(theme.vendors) }}/googleapis/css/Lato.css" rel="stylesheet" type="text/css">
{% endif %}
```

当然，如果不想这么麻烦，直接在主题设置中关闭Lato字体即可。


