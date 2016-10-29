title: 编写 October CMS Form Widget
date: 2016-09-24 22:00:32
tags: [OctoberCMS, Laravel, PHP]
categories: Try
---

# 概述

近期接触 [`Laravel`](https://laravel.com/) 这一框架之后，对使用便捷性和功能的丰富感到十分满意，同时开发者与相关的社区都相当活跃，这样的框架算是相当之理想的了。

了解了框架的使用之后，自然是希望能够找到这一框架构建的应用进行进一步的学习。CMS作为常见的系统，模式上个人理解起来比较容易，同时涉及面也比较多，同时还有一定的实用性，学习起来有价值。

使用 Laravel 的 CMS 不少，有本文的主角[October CMS](https://github.com/octobercms/october)、[Asgard CMS](https://github.com/AsgardCms)、[Lavalite](https://github.com/LavaLite/cms)，但是后续的这些，无论从 GitHub star数目上，还是更新频率上，以及社区活跃度上，完全无法与 October CMS 相提并论。

于是决定简单了解一下October CMS（以下简称October）。

# 需求

October宣传自身是一个简单、现代、以人为本、通用的、可拓展的、有趣的、可靠地、易学、节省时间的CMS。

对于OctoberCMS，对于一对多的关系，可以在model层进行处理。

然而，处于对体验优化的考虑，简单的关联记录的效果并不能满足。譬如需要完成一个相关文章的编辑表单，如果能够以card的形式展现相关文章的`头图`，`标题`，`摘要`等内容，那么会更加便于后台人员的操作，提供更好的体验。

同时，可以通过这一个简单的表单插件，熟悉October CMS表单插件的二次开发。

总而言之，需求如下：

+ 开发一个相关记录的表单插件
+ 表单插件可以显示相关记录的`头图`，`标题`，`摘要`
+ 相关记录可以调整顺序

# OctoberCMS 表单组件

OctoberCMS的有一类组件，被称为[`Form`](https://octobercms.com/docs/backend/forms)组件，为后台操作数据提供了极大的方便，通过配置各种Form插件的组合，可以对数据进行操作。

常规的管理界面的开发，一种模式是通过编写接口，提供修改对应数据的功能，并且前端开发需要构建对应的操作界面。对于CMS系统，这样无疑是一个低效的行为，使得管理平台开发也成为了开发工作的一大负担，提供更为简便的管理平台开发方式，对于减少开发工作量有很大意义。

OctoberCMS的表单组件通过yaml配置文件的形式，将数据库字段与前端编辑表单组件关联起来，为快速构建针对于数据的管理界面提供了可能。

针对于基本的字符串，时间选择器，富文本编辑器，单选/复选框，OctoberCMS也都默认提供了。

## 组件设计

对于表单组件，功能上可以一言概之，即通过图形方式构建JSON格式的字符串，并写入数据库。

### 设计细化

细化需求来说，关键点在于表单，数据格式，图形界面。

对于`表单`，最简单的情况即通过一个input标签，在进行form提交时，带上需要提交的数据。

对于`数据格式`，我们需要记录的是相关记录的数据库中的自增ID，存储格式会整编为JSON格式。

对于`图形界面`，考虑到操作的便捷性，同时考虑到显示足够的有效信息，通过显示blog的头图、标题、摘要等信息的card的形式，表现关联记录。

#### 进一步细化[表单]

对于表单来说，只需要确定表单中的name属性即可。

```
<input
   type="text"
   name="<?php echo $name ?>"
   id="<?php echo $this->getId() ?>"
   value="<?php echo json_encode($value); ?>"
   placeholder=""
   class="form-control"
   autocomplete="off"
   style="display: none;"
>
```

新增的情况，无需填充记录的默认值，而在更新的情况，则需要进行填充，相关PHP代码的说明会在后续提及。

#### 进一步细化[数据格式]

选用JSON作为数据格式，记录各个关联记录的自增ID。

#### 进一步细化[图形界面]

同时显示头图，标题，摘要，通过card形式展现。

操作上，需要有`增加`，`删除`，`调整顺序`三种。

+ `增加`操作可以通过弹出记录列表的形式，选取后展现。
+ `删除`则直接通过点击card上的删除按钮/文字即可。
+ `调整顺序`通过前移/后移按钮完成。

## 组件目录结构

OctoberCMS的表单插件需要符合一定的目录结构规则。

可以通过直接创建对应的目录以及文件，也可以通过内置的工具进行初始化：

```
php artisan create:formwidget rainlab.blog RelatedRecords
```

`rainlab.blog` 是将要创建这一组件的插件的名称，这里可以换成自己插件的名字。关于插件，参见[OctoberCMS的手册](http://octobercms.com/docs/plugin/registration)。


## 组件实现细节

## 组件效果


