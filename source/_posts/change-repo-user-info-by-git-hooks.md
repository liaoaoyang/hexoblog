date: 2015-12-03 00:00:00
title: git根据仓库url修改用户信息
categories: Learning
tags: [Git]
---


# 起因

日常Coding使用git的一个问题是，公司工作和个人项目有时候会使用不同的用户信息，例如公司项目要求个人信息必须为`your_name@your_company.com`的形式，而你自己在使用`Gmail`，这里自然有些冲突，如果忘记在clone的项目中配置一下，极有可能会把自己的日常使用邮箱提交到公司的repo中，反之亦然。

虽然git本身可以通过`--global`进行配置，日常使用上使用都用自己的常用个人信息，但是公司项目每次都要单独配置也是做了太多无用功，那么如果能让这个任务更加自动化想必是可以提高工作效率的。

# 方案

解决的思路决定采用`git`的[`hooks`][1]进行处理，在每一次的`commit`操作前触发，判断当前项目的远端仓库url是否为公司url，若为公司url则修改当前项目目录下的git用户信息，即：

```
	$1 ~ /your-company-git-repo-url/ {
		name = "your_name";
		email = "your_name@your-company.com";
	}
```
具体匹配操作运用了`shell`与`AWK`的结合，个人感觉`AWK`在匹配以及字符串处理上更胜一筹，主要逻辑利用`AWK`完成。


主要逻辑如下：

```
push_url=$(git remote -v | awk '/push/ {print $2}' | head -1)

if [ -n "$push_url" ]; then
	ret=""
	echo $push_url | awk -v global_name=$global_name -v global_email=$global_email -v repository_name=$repository_name -v repository_email=$repository_email 'BEGIN {
		default_name = global_name;
		default_email = global_email;
		if (length(default_name) == 0) {
			default_name = "your_default_name";
		}
		if (length(default_email) == 0) {
			default_email = "your_default_name@your-default-email.com";
		}
		name = default_name;
		email = default_email;
	}
	$1 ~ /your-company-git-repo-url/ {
		name = "your_name";
		email = "your_name@your-company.com";
	}
	$1 ~ /other-git-url/ {
		name = "your_name";
		email = "your_name@other-git.com";
	}
	END {
		if (repository_name == name && repository_email == email) {
			exit 0;
		}
		if (length(repository_name) == 0 || repository_name != name) {
			name = (length(name) > 0 ? name : default_name);
		}
		if (length(repository_email) == 0 || repository_email != email ) {
			email = (length(email) > 0 ? email : default_email);
		}
		if (length(name) > 0 && length(email) > 0) {
			cmd = "git config user.name "name";git config user.email "email";";
			print "Info : "cmd;
			system(cmd);
			exit 1;
		}
	}' || ret="1"

	if [ ! -z $ret ]; then
		echo "Pls retry commit"
		exit 1
	fi
fi
```

其中通过awk的exit值向shell传递信息，通知当前状态，若修改完成，提示用户重新进行commit操作。

# 使用

在[`GitHub`][5]上获取代码后，根据实际情况配置好匹配规则以及用户信息，将hook文件置入git安装目录下的 `/share/git-core/templates/hooks/`之下，并`chmod +x`赋予执行权限，如在mac上通过homebrew安装的git，则会在类似：

```
 /usr/local/Cellar/git/1.9.0/share/git-core/templates/hooks/
```

目录下。

最后还需要指定项目的默认init模板目录，例如：

```
git config init.templatedir /usr/local/Cellar/git/1.9.0/share/git-core/templates/
```

后续clone的项目都能起到效果了。

# 参考文档

参考了 [`@liaohuqiu`][2] 大大的博客文章[`git: 提交前强制检查各个项目用户名邮箱设置`][3]，同时修改了[`相关项目`][4]的代码，感谢！

# TODO

在脚本中尝试帮助用户完成提交操作。

[1]: http://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks
[2]: http://www.liaohuqiu.net/about/about-me.html
[3]: http://www.liaohuqiu.net/cn/posts/using-diffrent-user-config-for-different-repository
[4]: https://github.com/liaohuqiu/work-anywhere/blob/master/sample/git-template/hooks/pre-commit
[5]: https://github.com/liaoaoyang/work-anywhere/blob/master/sample/git-template/hooks/pre-commit
