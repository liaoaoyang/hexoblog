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

### scope

即微信 OAuth 中的 scope 值，即`snsapi_base` 与 `snsapi_userinfo`。


