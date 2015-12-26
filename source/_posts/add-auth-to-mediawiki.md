date: 2014-02-22 00:00:00
title: Mediawiki 登录操作
categories: Try
tags: [mediawiki]
---

[Mediawiki][1]是一个广泛使用的wiki系统之一。

整个wiki的搭建过程十分的简便，几乎是各种下一步的傻瓜安装方式。安装时在有Memcached的机器的前提下，强烈建议配置上Memcached，在配与不配Memcached这个选项上，配置了Memcached之后速度会有明显的提升。

在使用wiki的过程中，可能我们需要对外公开wiki的内容，但是不能让未授权用户编辑的情况，这时候就需要使用认证。在[官方网站][2]上有很多的认证插件可以选择，不过有时需要加入自己使用的wiki系统的一些特性的时候（比如用公司的统一登录接口），插件就不一定适用了，需要自己编写插件。

# 基本信息

下面是Mediawiki默认的user表表结构，在下文中对于表字段的操作依据都来源于下方：

```
CREATE TABLE IF NOT EXISTS `prefix_user` (
  `user_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `user_name` varchar(255) CHARACTER SET utf8 COLLATE utf8_bin NOT NULL DEFAULT '',
  `user_real_name` varchar(255) CHARACTER SET utf8 COLLATE utf8_bin NOT NULL DEFAULT '',
  `user_password` tinyblob NOT NULL,
  `user_newpassword` tinyblob NOT NULL,
  `user_newpass_time` binary(14) DEFAULT NULL,
  `user_email` tinytext NOT NULL,
  `user_touched` binary(14) NOT NULL DEFAULT '\0\0\0\0\0\0\0\0\0\0\0\0\0\0',
  `user_token` binary(32) NOT NULL DEFAULT '\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0',
  `user_email_authenticated` binary(14) DEFAULT NULL,
  `user_email_token` binary(32) DEFAULT NULL,
  `user_email_token_expires` binary(14) DEFAULT NULL,
  `user_registration` binary(14) DEFAULT NULL,
  `user_editcount` int(11) DEFAULT NULL,
  PRIMARY KEY (`user_id`),
  UNIQUE KEY `user_name` (`user_name`),
  KEY `user_email_token` (`user_email_token`),
  KEY `user_email` (`user_email`(50))
) ENGINE=InnoDB  DEFAULT CHARSET=utf8 AUTO_INCREMENT=1 ;
```

# 插件编写

关于插件的编写，可以首先看看官网上对于认证插件的[说明][3]，自己尝试编写了两种登录的方式，一种是通过继承 ***`AuthPlugin`*** 类并重写 ***`authenticate`*** 方法实现，另一种是编写 ***`UserLoadFromSession`*** 的对应处理方法处理利用现有的Cookie进行登录的方法。第一种方法个人觉得适合使用同一登录的情况下使用，第二种方法适合wiki与提供认证的服务处于同域的情况下使用，实现同步登录。

## 重写 ***`authenticate`*** 方法

首先我们要做的就是继承 ***`AuthPlugin`*** 类，并且将标记认证对象的全局变量 ***`$wgAuth`*** 设定为一个这一类型的对象。

```
<?php
    require_once dirname(dirname(dirname(__FILE__))) . '/includes/AuthPlugin.php';

    global $wgAuth;
    $wgAuth = new MyAuth();

    class MyAuth extends AuthPlugin
    {
    }
```

参见 ***`AuthPlugin`*** 类中 ***`authenticate`*** 方法的定义：

```
/**
 * @param string $username username.
 * @param string $password user password.
 * @return bool
 */
public function authenticate( $username, $password ) {
	# Override this!
	return false;
}
```

需要实现的就是一个传入用户名密码并在验证之后返回结果的方法。用户在wiki点击登录动作发送来的用户名密码将会在这个方法中得到认证，利用使用公司邮箱密码登录，利用统一的LDAP登录接口获取数据，判断是否登录成功。

对于登录成功的，应当返回 ***true***，登录不成功的，应当返回 ***false***。

但是，在使用外部确认的登录方式的情况下，仍然需要在wiki数据库中的 *`WIKI_TABLE_PREFIX_user`* 表中建立用户（*`WIKI_TABLE_PREFIX_`* 表示是），在认证通过之后，需要构建一个 ***`User`*** 类的对象，并且将表示当前登录用户的全局变量 ***[$wgUser][4]*** 设定为生成的User类对象。例如登录成功之后返回用户的用户名，假设用户名是唯一的情况下，可以在数据库中检索出相关的数据，构建用户对象。

启用这一方式进行登录验证还需要在wiki代码根目录下的 ***LocalSettings.php*** 文件中引入这一文件（根据实际位置决定）。

```
require_once "$IP/extensions/MyAuth/MyAuth.php";
```

如果是想要将wiki通过iframe形式加入其它页面的，记得还需要在 ***LocalSettings.php*** 中设定

```
$wgEditPageFrameOptions = false;
```

Mediawiki默认是不允许wiki被iframe加载的。

用如下代码大致可以表示整个方法的实现：

```
<?php
    require_once dirname(dirname(dirname(__FILE__))) . '/includes/AuthPlugin.php';

    /**
     * 数据库连接参数
     */
    global $wgDBserver;
    global $wgDBname;
    global $wgDBuser;
    global $wgDBpassword;
    global $wgDBprefix;

    define('DB_HOST', $wgDBserver);
    define('DB_PORT', 10000);
    define('DB_USERNAME', $wgDBuser);
    define('DB_PASSWORD', $wgDBpassword);
    define('DB_NAME', $wgDBname);
    define('DB_TABLE_NAME', $wgDBprefix . 'user');

    class MyAuth extends AuthPlugin
    {
        public static function getUserInfoByUserName($userName, $key)
        {
            try
            {
                /**
                 * 根据用户名选取数据库中的字段$key
                 */
                $dsn = 'mysql:host=' . DB_HOST . ';port=' . DB_PORT . ';dbname=' . DB_NAME;
                $dbIns = new PDO($dsn, DB_USERNAME, DB_PASSWORD);
                $sql = "SELECT `?` FROM " . DB_TABLE_NAME . " WHERE `user_name` = ?";
                $h = $dbIns->prepare($sql);
                $h->execute(array($key, $userName));
                $dbRet = $h->fetchAll();

                /**
                 * 用户名唯一，所以返回结果超出一个说明存在问题
                 */
                if (1 != count($dbRet))
                {
                    return false;
                }

                $dbRet = $dbRet[0];

                if (isset($dbRet[$key]))
                {
                    return $dbRet[$key];
                }
            }
            catch (PDOException $ex)
            {
                return false;
            }

            return false;
        }
        public static function getUser($userName)
        {
            try
            {
                /**
                 * 根据用户名选取数据库中的id
                 */
                $dsn = 'mysql:host=' . DB_HOST . ';port=' . DB_PORT . ';dbname=' . DB_NAME;
                $dbIns = new PDO($dsn, DB_USERNAME, DB_PASSWORD);
                $sql = "SELECT `user_id` FROM " . DB_TABLE_NAME . " WHERE `user_name` = ?";
                $h = $dbIns->prepare($sql);
                $h->execute(array($userName));
                $dbRet = $h->fetchAll();

                /**
                 * 用户名唯一，所以返回结果超出一个说明存在问题
                 */
                if (1 != count($dbRet))
                {
                    return false;
                }

                $dbRet = $dbRet[0];

                if (isset($dbRet['user_id']))
                {
                    /**
                     * 通过id生成用户对象并返回
                     */
                    $user = User::newFromId($dbRet['user_id']);
                    return $user;
                }
            }
            catch (PDOException $ex)
            {
                return false;
            }

            return false;
        }

        public static function setUser($userInfo)
        {
            try
            {
                /**
                 * 根据用户信息构建用户，更新或者增加用户，并且返回一个User类型的对象
                 */
            }
            catch (PDOException $ex)
            {
                return false;
            }

            return false;
        }

        public function userExists($username)
        {
            if (self::getUser($username))
            {
                return true;
            }

            return false;
        }

        /**
         * 通过外部登录可以不允许自动的创建用户
         */
        public function autoCreate()
        {
            return false;
        }

        /**
         * wiki数据库并没有设定用户的密码，并且因为会用外部的登录方式，为了减少用户的困惑，不允许修改密码
         */
        public function allowPasswordChange()
        {
            return false;
        }

        /**
         * 类似上述原因
         */
        public function allowSetLocalPassword()
        {
            return false;
        }

        public function authenticate($username, $password)
        {
            global $wgUser;
            global $wgOut;

            /**
             * 在yourAuthMethod中实现真正的登录方法，例如LDAP等，成功则返回用户信息，反之为false
             */
            $userInfo = yourAuthMethod($userName, $password);

            if ($userInfo)
            {
                $wgUser = self::setUser($userInfo);
                return true;
            }

            return false;
        }
    }

```

## 编写 ***`UserLoadFromSession`*** 的对应处理方法

在请求数据的过程中， 通过注册到 ***`$hooks`*** 中的方法，可以进行进行验证(***`$hooks`*** 参见[官方说明][5]，是Mediawiki中可以让用户自定义方法在特定情况下使用的机制），对于这个方法，名称可以任意取，但是参数必须为：

```
/**
 *  @param User $user user object.
 *  @param bool $result determined whether user is a valid user.
function onUserLoadFromSession($user, &$result)
{
    global $wgUser;

    $result = false;
    return true;
}
```

即传入一个 ***`User`*** 类的对象和一个标记用户是否已登录的布尔值，其中布尔值用于确定用户登录状态。

此方法的返回值，[官方文档][6]建议一直返回 ***true*** （即*In any case, return `true`*）。

对于判定用户是否登录，在同域的情况下，自己尝试的思路是检查Cookie中是否带有指定的字段信息，例如token值以及用户名，将其与数据库中的数据进行比对，通过则构建一个用户对象并将其赋值给 ***`$wgUser`*** 全局变量。

可以在上述类中添加如下代码完成功能：
```
public static function parseCookie($key)
{
    $ret = array();

    if (isset($_COOKIE[$key]))
    {
        $originalLUString = urldepre(urldecode($_COOKIE[$key]));
        parse_str($originalLUString, $ret);

        if (!is_array($ret) || !isset($ret['name']) || !isset($ret['token'] || || !isset($ret['exp']))
        {
            return false;
        }
    }

    return $ret;
}

public static function onUserLoadFromSession($user, &$result)
{
    global $wgUser;

    $ret = self::parseCookie('AUTH_INFO');

    if (false === $ret)
    {
        $result = false;
        return true;
    }

    $userName = $ret['name'];
    $tokenFromCookie = $ret['token'];
    $token = self::getUserInfoByUserName($userName, 'user_token');
    /**
     * user_token_expires 字段的解释参见下文
     */
    $tokenExpiration = self::getUserInfoByUserName($userName, 'user_token_expires');

    if (empty($token) || $token != $tokenFromCookie || time() > $tokenExpiration)
    {
        return false;
    }

    $user = self::getUser($userName);

    if ($user)
    {
        $wgUser = $user;
        $result = true;
        return true;
    }

    $result = false;
    return true;
}
```

这里需要注意token是应当有超时的，我的处理方式是在user表中增加如下一个字段来记录超时的时间戳：

```
`user_token_expires` bigint(20) NOT NULL DEFAULT '0'
```

通过在取回token的同时取回超时时间戳，并与当前时间戳做对比，若当前时间戳小于超时，则认定未过期。

如果需要实现同步登录状态，那么在还需要在主要的登录环境中在退出时，取消用户登录状态，例如将token字段置空，这样可以标记用户已经退出。同时也要对直接从wiki登录的用户做处理，可以针对 ***`UserLogout`*** 这个hook编写一个处理方法，名称任意，在这一处理方法中将token字段置空。在这里需要在代码中注册 ***`UserLoadFromSession`*** 以及 ***`UserLogout`*** 对应的处理方法：

```
<php?
    require_once dirname(dirname(dirname(__FILE__))) . '/includes/AuthPlugin.php';

    global $wgHooks;
    $wgHooks['UserLoadFromSession'][] = 'MyAuth::UserLoadFromSession';
    $wgHooks['UserLogout'][] = 'MyAuth::UserLogout';

    global $wgAuth;
    $wgAuth = new MyAuth();

    class MyAuth extends AuthPlugin
    {
    }

```

可以考虑使用Redis或者Memcached缓存token等信息，并且设定上数据超时，能减少数据库访问并且相应速度更快。




[1]: http://www.mediawiki.org/wiki/MediaWiki
[2]: http://www.mediawiki.org/wiki/categories:User_identity_extensions
[3]: http://www.mediawiki.org/wiki/Authentication
[4]: http://www.mediawiki.org/wiki/Manual:$wgUser
[5]: http://www.mediawiki.org/wiki/Hooks
[6]: http://www.mediawiki.org/wiki/Manual:Hooks/UserLoadFromSession
