title: 微信多回调域名
date: 2019-01-28 23:27:34
tags: [WeChat,微信,OAuth]
categories: 系统
---

# TL;DR

只有一个公众号但需要让多个域名获取OAuth返回的用户信息或微信登录功能，可以考虑使用集中的回调域名进行中转。

本文也是对个人实验项目 [SWAN](https://github.com/liaoaoyang/swan) 相关功能的说明。

<!-- mutilple-wechat-oauth-callback-urls -->
<!-- more -->

# 分析

参见微信开发文档[微信网页授权](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140842)，需要在后台配置`redirect_uri`，但是`redirect_uri`每月配置次数以及个数都是有限的。

如果业务有多个域名，都需要使用微信登录或者获取微信基础用户信息，就会存在问题。

整个流程中，配置的`redirect_uri`是掌握在自己手中的，可以考虑使用回调url再做一次重定向，将用户信息带给其他域名。

# 实现

## 时序图

以个人目中的实现 [MyWXTAuth.php](https://github.com/liaoaoyang/swan/blob/master/app/Utils/MyWXTAuth.php) 举例：

```
     ,----.          ,---.             ,----.                      ,------.
     |User|          |T3P|             |SWAN|                      |WeChat|
     `-+--'          `-+-'             `-+--'                      `--+---'
       |               |                 |                            |    
       | ------------->|                 |                            |    
       |               |                 |                            |    
       |               |bid/url/key/scope|                            |    
       |               |----------------->                            |    
       |               |                 |                            |    
       |               |                 | OAuth callback url/code/...|    
       |               |                 | --------------------------->    
       |               |                 |                            |    
       |               |                 |                            |    
       | <-------------------------------------------------------------    
       |               |                 |                            |    
       |               |            confirm                           |    
       | ------------------------------------------------------------->    
       |               |                 |                            |    
       |               |                 |      user information      |    
       |               |                 | <---------------------------    
       |               |                 |                            |    
       |               |user information |                            |    
       |               |<-----------------                            |    
     ,-+--.          ,-+-.             ,-+--.                      ,--+---.
     |User|          |T3P|             |SWAN|                      |WeChat|
     `----'          `---'             `----'                      `------'
```

## 步骤

### 请求SWAN授权URL

常规微信登录可以访问拼接好的微信相关URL，此处修改为类似的方式，访问 SWAN 实现的授权URL，参数说明：

#### bid

业务ID，SWAN 会通过这一参数，读取配置信息中允许跳转的域名。

#### url

业务方的回跳URL。在微信授权成功后，会在回跳URL中追加一个参数`wx_tauth_data`，参数内容为JSON格式的用户信息

#### key

可以为随机数。

#### scope

即微信 OAuth 中的 scope 值，即`snsapi_base` 与 `snsapi_userinfo`。

### 举例

假设域名为 `wxtauth.foo.com`，`bid` 为 `test`，回调地址 `url` 为 `https://wxtauth.foo.com/callback`，`scope` 为 `snsapi_userinfo`，客户端和服务端需要完成的工作为：

#### 客户端

假设服务端的域名为 `swan.bar.com`，默认情况下（假设服务端使用HTTPS），用于中转的登录接口 URL 为 `https://swan.bar.com/wechat/swan/wxtauth`。

完成自己的业务逻辑之后拼接出 URL：

 `https://swan.bar.com/wechat/swan/wxtauth?bid=test&url=https://wxtauth.foo.com/callback&scope=snsapi_userinfo&key=123456`
 
 并302重定向到这一 URL 之上。
 
在弹出新窗口中完成授权操作后，需要在 `url` 参数指定的 URL 上实现相应的登录成功处理逻辑，即读取回调中的携带的 `wx_tauth_data` 参数，之后继续完成对应的业务逻辑。

#### 服务端

处理逻辑参见 [\App\Http\Controllers\WeChatController::wxtauth](https://github.com/liaoaoyang/swan/blob/master/app/Http/Controllers/WeChatController.php)。

服务端首先需要在 `.env` 文件中配置 `bid` 为 `test` 的请求所允许的回调域名。配置格式：

WX_TAUTH_CLIENT_***TEST***_AUTHENTIC_DOMAINS=wxtauth.foo.com

请注意加粗的 `TEST` ，每新增一个bid，需要增加一行配置，在加粗部分替换为大写的 `bid` 的值。

之后服务端会完成微信相关的授权功能，这里特别感谢 [安正超](https://github.com/overtrue) 优秀的开源项目 [EasyWeChat](https://github.com/overtrue/wechat)，让微信接入变得极为简便。



