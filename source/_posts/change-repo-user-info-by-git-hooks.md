date: 2015-12-03 00:00:00
title: git根据仓库url修改用户信息
categories: Learning
tags: [Git]
---


# 起因

日常Coding使用git的一个问题是，公司工作和个人项目有时候会使用不同的用户信息，例如公司项目要求个人信息必须为`your_name@your_company.com`的形式，而你自己在使用`Gmail`，这里自然有些冲突。

如果忘记在 clone 的项目中配置一下，极有可能会把自己的日常使用邮箱提交到公司的repo中，反之亦然。

虽然git本身可以通过`--global`进行配置，默认各个项目都用自己的常用个人提交信息，但是对于公司项目，每次都要单独配置也是做了太多无用功，那么如果能让这个任务更加自动化想必是可以提高工作效率的。

# 方案

决定采用`git`的[`hooks`][1]进行处理，在每一次的`commit`操作前触发，判断当前项目的远端仓库url是否为公司url，若为公司url则修改当前项目目录下的git用户信息，即：

```
// $1 为公司repo url关键字
// $2 为用户名
// $3 为用户邮箱
if (index(push_url, $1) != 0) {
	name = $2;
	email = $3;
}
```
具体匹配操作运用了`shell`与`awk`的结合，个人感觉`awk`在匹配以及字符串处理上更胜一筹，主要逻辑利用`awk`编写完成。

为了能够动态更改提交信息，需要在`~`目录下增加一个配置文件`~/.git_repo_user_info.conf`，这一配置文件的格式为：

```
url关键字 用户名 邮箱
```

三者通过空格分隔。

主要逻辑如下：

```
push_url=$(git remote -v | awk '/push/ {print $2}' | head -1)

my_username=`whoami`
repo_user_info_config=`eval echo ~$my_username`"/.git_repo_user_info.conf"

if [ ! -f $repo_user_info_config ]; then
	repo_user_info_config="/dev/null"
fi

if [ -n "$push_url" ]; then
	ret=""
	sort -dk 1 $repo_user_info_config | awk -v push_url=$push_url -v global_name=$global_name -v global_email=$global_email -v repository_name=$repository_name -v repository_email=$repository_email 'BEGIN {
		default_name = global_name;
		default_email = global_email;
		if (length(default_name) == 0) {
			print "Info : Please use git config --global user.name ur_default_name to set default name";
			exit 1;
		}
		if (length(default_email) == 0) {
			print "Info : Please use git config --global user.email ur_name@ur_company to set default name";
			exit 1;
		}
		name = default_name;
		email = default_email;
	}
	{
		if (index(push_url, $1) != 0) {
			name = $2;
			email = $3;
		}
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

通过字典序排序，如果有多种匹配情况，则命中字典序靠后的规则。

（以上逻辑2017年08月18日更新）

# 使用

在[`GitHub`][5]上获取代码后，根据实际情况配置好匹配规则以及用户信息，将hook文件置入git安装目录下的 `/share/git-core/templates/hooks/`之下，并`chmod +x`赋予执行权限，如在mac上通过homebrew安装的git，则会在类似：

```
 /usr/local/Cellar/git/1.9.0/share/git-core/templates/hooks/
```

目录下。

同时配置常用的各个repo相关的提交信息配置文件`~/.git_repo_user_info.conf`，形如：

```
github.com foo bar@gmail.com
coding.net bar foo@gmail.com
mycompany.com work work@mycompany.com
```

最后还需要指定项目的默认init模板目录，例如：

```
git config init.templatedir /usr/local/Cellar/git/1.9.0/share/git-core/templates/
```

后续clone的项目都能起到效果了。

如果想要对已有项目也起到对应的效果，可以考虑替换项目根目录中`.git/hooks`目录下的对应文件。

# 参考文档

参考了 [`@liaohuqiu`][2] 大大的博客文章[`git: 提交前强制检查各个项目用户名邮箱设置`][3]，同时修改了[`相关项目`][4]的代码，感谢！

# TODO

在脚本中尝试帮助用户完成提交操作。

[1]: http://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks
[2]: http://www.liaohuqiu.net/about/about-me.html
[3]: http://www.liaohuqiu.net/cn/posts/using-diffrent-user-config-for-different-repository
[4]: https://github.com/liaohuqiu/work-anywhere/blob/master/sample/git-template/hooks/pre-commit
[5]: https://github.com/liaoaoyang/work-anywhere/blob/master/sample/git-template/hooks/pre-commit


