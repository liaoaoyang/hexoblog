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


