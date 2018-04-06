title: 使用Docker运行基于Nginx+PHP的应用程序
date: 2018-02-28 11:46:19
tags: [Docker, PHP]
categories: 系统
---

# TL;DR

Docker作为目前主流的容器技术之一，可以更高效的利用系统资源，拥有更快的启动时间，提供一致的运行环境，能更轻松的维护和扩展。

作为开发者，在这项技术出现多年之后，是时候将他用于加速自己的App的开发、测试、部署阶段了。

本文是个人的学习笔记，目的在于描述如何让一个基于 Nginx + PHP 的应用程序在 Docker 中运行起来。原理性的内容会另起篇幅。
<!-- more -->

# 运行环境

## OS

使用 `Ubuntu 16.04 LTS` 完成，同样也可以选用 `CentOS 7`，因为问题较多，放弃了在 `CentOS 6.5` 上的尝试（毕竟是很老的系统了）。

## Docker

1.13

## Nginx

选用了比较喜欢的 `Openresty`，版本为 `1.13.6.1`，镜像为 `openresty/openresty:alpine`。

## PHP

本文为 `7.2.2`，镜像为 `php:7.2.2-fpm-alpine3.7`。

# 部署

如果要用 Docker 运行自己的应用程序，首先要对镜像有一定的了解。

## openresty 容器

对于 OpenResty 来说，Dockerfile 需要了解的主要是 Nginx 的配置。

从 Dockerfile 中可以看到 nginx.conf 是在构建时拷贝到容器之中的。

```
COPY nginx.conf /usr/local/openresty/nginx/conf/nginx.conf
```

在 GitHub 上可以看到 [nginx.conf](https://github.com/openresty/docker-openresty/blob/master/nginx.conf) 文件的内容：

```
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

可以了解到如下信息：

+ Nginx 配置文件为 /usr/local/openresty/nginx/conf/nginx.conf
+ vhosts配置文件目录为 /etc/nginx/conf.d/，这一个目录可以放置需要运行的项目的Nginx配置文件


## php-fpm 容器

对于 PHP 来说，Dockerfile 需要了解的至少有如下几个方面：

+ php/php-fpm/php-config 等二进制文件的安装位置
+ php.ini 文件的位置以及内容
+ php-fpm.conf 文件的位置以及内容
+ php-fpm 运行用户与用户组

Docker 在构建镜像时，通过 Dockerfile 对构建镜像的动作进行描述。可以通过分析一下这个文件了解一下选用镜像的特点。

### php:7.2.2-fpm-alpine3.7 Dockerfile

[源文件](https://github.com/docker-library/php/blob/b4e5b2b22ae2e8de03e882a2666d6e39a7c79f93/7.2/alpine3.7/fpm/Dockerfile) 内容虽然多，但是需要关注的东西并不多，带着之前的问题去阅读这个文件。

这个文件事实上完成的主要是如下几个工作：

+ 声明依赖的基础镜像
+ 设定环境变量
+ 拷贝 PHP 环境所需的特定工具（如扩展安装工具）
+ 构建以及配置 PHP 7.2.2
+ 启动 php-fpm 以及暴露端口

### php/php-fpm/php-config 等二进制文件的安装位置

在整个文件中没有看到诸如 `--prefix` 这样的参数，那么可以得知会把这些文件默认安装到 `/usr/local/bin/` 目录下。

启动一个容器进行验证：

```
 ➜ docker run -it --rm php:7.2.2-fpm-alpine3.7 /bin/sh
/var/www/html # ls /usr/local/bin/
docker-php-entrypoint     docker-php-ext-install    peardev                   phar.phar                 phpdbg
docker-php-ext-configure  docker-php-source         pecl                      php                       phpize
docker-php-ext-enable     pear                      phar                      php-config
```

### php.ini 文件的位置以及内容

文件37行:

```
ENV PHP_INI_DIR /usr/local/etc/php
```

在后续构建 configure 阶段（92行开始）：

```
RUN set -xe \
	&& apk add --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		coreutils \
		curl-dev \
		libedit-dev \
		libressl-dev \
		libxml2-dev \
		sqlite-dev \
	\
	&& export CFLAGS="$PHP_CFLAGS" \
		CPPFLAGS="$PHP_CPPFLAGS" \
		LDFLAGS="$PHP_LDFLAGS" \
	&& docker-php-source extract \
	&& cd /usr/src/php \
	&& gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
	&& ./configure \
		--build="$gnuArch" \
		--with-config-file-path="$PHP_INI_DIR" \
		--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
```

可以看到也定义了这一个目录。

然而在整个文件中也没有看到拷贝 ini 文件到对应目录的操作，所以这个容器是没有使用 php.ini 文件进行配置的。

如果想要知道当前默认的配置项，不妨进入容器，通过 `/usr/local/bin/php -i` 进行查看。

### php-fpm.conf 文件的位置以及内容

类似 php.ini 文件的情况。

### php-fpm 运行用户与用户组

这里涉及到一个权限的问题，Docker 可以把应用代码拷贝到容器中，但是也可以通过挂载目录的方式完成，将目录的 owner 设定为当前容器运行的 uid 可以避免权限带来问题。

文件29行可以看到新增了 uid 为 82 的 `www-data` 用户。

```
RUN set -x \
	&& addgroup -g 82 -S www-data \
	&& adduser -u 82 -D -S -G www-data www-data
```

在构建阶段，也声明了 fpm 运行者为 `www-data`：

```
ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data
```

## 启动容器

### 通过 link 关联 openresty 与 php-fpm

Docker 容器之间不能直接访问，可以用 link 的方式，也可以直接把端口映射到主机端口上，通过主机的网络进行通信。

需要先行启动 php-fpm 容器。

#### 启动 php-fpm 容器

因为需要配置诸如时区等配置，然而当前容器又没有默认的 php.ini 文件，可以挂载一个自行配置 php.ini 文件。

`php.ini` 配置文件中将 PHP 错误日志目录定义为 `/var/www/logs/php_error.log`，所以挂载可以写入日志的目录到容器中的 `/var/www/logs` 目录下。

```
docker run --name=test_phpfpm \
-v /data1/www/etc/test/conf/php/php.ini:/usr/local/etc/php/php.ini:ro \
-v /data1/www/etc/test/conf/php:/usr/local/etc/php/conf.d/:ro \
-v /data1/www/htdocs/test:/var/www/html \
-v /data1/www/logs:/var/www/logs \
-d php:7.2.2-fpm-alpine3.7
```

php-fpm 的容器名为 `test_phpfpm` ，与之连接的其他容器可以通过这个名字直接访问容器。

#### 启动 openresty 容器

```
docker run --link=test_phpfpm --name=test_openresty -p 8808:80 \
-v /data1/www/etc/test/conf/nginx/nginx.test.config:/usr/local/openresty/nginx/conf/nginx.conf:ro \
-v /data1/www/etc/test/conf/nginx:/etc/nginx/conf.d/:ro \
-v /data1/www/logs:/usr/local/openresty/nginx/logs \
-v /data1/www/htdocs/test:/var/www/html \
-d openresty/openresty:alpine
```

对应的 Nginx 配置文件为：

```
worker_processes  1;

events {
    worker_connections  65535;
}


http {
    include       mime.types;
    #default_type  application/octet-stream;
    default_type  text/html;

    sendfile        on;

    keepalive_timeout  65;

    server {
        listen 80 default_server;
        root /var/www/html;
        index index.php index.html index.htm;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php($|/) {
            fastcgi_pass          test_phpfpm:9000;
            fastcgi_index         index.php;
            include               fastcgi_params;
            set $path_info        "";
            set $real_script_name $fastcgi_script_name;

            if ($fastcgi_script_name ~ "^(.+?\.php)(/.+)$") {
                set $real_script_name $1;
                set $path_info        $2;
            }

            fastcgi_param SCRIPT_FILENAME $document_root$real_script_name;
            fastcgi_param SCRIPT_NAME     $real_script_name;
            fastcgi_param PATH_INFO       $path_info;
            fastcgi_param PHP_VALUE       open_basedir=$document_root:/tmp/:/proc/:/dev/urandom;
        }
    }

    include /etc/nginx/conf.d/*.conf;
}
```

至此，在宿主机 `/data1/www/htdocs/test` 目录下的 PHP 项目可以对外提供服务了。

