title: 编写 October CMS Form Widget
date: 2016-09-24 22:00:32
tags: [OctoberCMS, Laravel, PHP]
categories: Try
---

# 概述

近期接触 [`Laravel`](https://laravel.com/) 这一框架之后，对使用便捷性和功能的丰富感到十分满意，同时开发者与相关的社区都相当活跃，这样的框架算是相当之理想的了。

了解了框架的使用之后，自然是希望能够找到这一框架构建的应用进行进一步的学习。CMS作为常见的系统，模式上个人理解起来比较容易，同时涉及面也比较多，同时还有一定的实用性，学习起来有价值。

使用 Laravel 的 CMS 不少，有本文的主角[October CMS](https://github.com/octobercms/october)、[Asgard CMS](https://github.com/AsgardCms)、[Lavalite](https://github.com/LavaLite/cms)，但是后续的这些，无论从 GitHub star数目上，还是更新频率上，以及社区活跃度上，完全无法与 October CMS 相提并论。

于是决定简单了解一下October CMS（以下将会用October表示，二者不做区分）。

本文写作的前提是：

+ 使用过基本的OctoberCMS的Ajax框架
+ 基本了解后端管理界面的组织方式
+ 了解基本的jQuery/CSS知识
+ 了解基本的Laravel使用

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

自动建立的目录结构如下：

```
→ tree
.
└── formwidgets
    ├── RelatedRecords.php
    └── relatedrecords
        ├── assets
        │   ├── css
        │   │   └── relatedrecords.css
        │   └── js
        │       └── relatedrecords.js
        └── partials
            └── _relatedrecords.htm
```

文件不多，简而言之，`RelatedRecords.php`用来描述表单组件以及编写处理逻辑，`relatedrecords.css`自然是控制表单后台操作样式，`relatedrecords.js`则是实现表单项目在后台的前端操作逻辑，最后`partials`中的`_relatedrecords.htm`则定义了表单组件在后台的html展示部分。

如果不想通过内置工具生成表单组件，同样也可以直接在对应plugin的目录中中的`widgets`子目录中直接新增插件的方式完成，仿照October默认的modules/backend中表单插件的结构即可：

```
→ tree widgets
widgets
├── Filter.php
├── Form.php
├── Lists.php
├── ReportContainer.php
├── Search.php
├── Table.php
├── Toolbar.php
├── filter
│   └── partials
│       ├── _filter.htm
...
│       └── _scope_switch.htm
├── form
│   ├── assets
│   │   └── js
│   │       └── october.form.js
│   └── partials
│       ├── _field-container.htm
...
│       └── _section.htm
├── lists
│   ├── assets
│   │   └── js
│   │       └── october.list.js
│   └── partials
│       ├── _list-container.htm
...
│       └── _setup_form.htm
├── reportcontainer
│   ├── assets
│   │   ├── css
│   │   │   └── reportcontainer.css
│   │   ├── js
│   │   │   └── reportcontainer.js
│   │   ├── less
│   │   │   └── reportcontainer.less
│   │   └── vendor
│   │       └── isotope
│   │           ├── jquery.isotope.js
│   │           └── jquery.isotope.min.js
│   └── partials
│       ├── _container.htm
...
│       └── _widget_toolbar.htm
├── search
│   └── partials
│       └── _search.htm
├── table
│   ├── ClientMemoryDataSource.php
│   ├── DataSourceBase.php
│   ├── README.md
│   ├── ServerEventDataSource.php
│   ├── assets
│   │   ├── css
│   │   │   └── table.css
│   │   ├── images
│   │   │   └── table-icons.gif
│   │   ├── js
│   │   │   ├── build-min.js
...
│   │   │   └── table.validator.required.js
│   │   └── less
│   │       └── table.less
│   └── partials
│       └── _table.htm
└── toolbar
    └── partials
        └── _toolbar.htm
```

本文的[demo](https://github.com/liaoaoyang/RelatedRecordsFormWidgetDemo)即是通过这种方式生成的，结构如下：

```
→ tree
widgets
├── RelatedRecords.php
└── relatedrecords
    ├── assets
    │   ├── css
    │   │   └── relatedrecords.css
    │   ├── fonts
    │   ├── img
    │   │   └── default.png
    │   ├── js
    │   │   └── relatedrecords.js
    │   └── libs
    │       └── materialize
    │           ├── LICENSE
    │           ├── README.md
    │           ├── css
    │           │   ├── materialize.css
    │           │   └── materialize.min.css
    │           ├── fonts
    │           │   └── roboto
    │           │       ├── Roboto-Bold.eot
...
    │           │       └── Roboto-Thin.woff2
    │           └── js
    │               ├── materialize.js
    │               └── materialize.min.js
    └── partials
        └── _field_relatedrecords.htm
```

可以看到，二者并无太大差别，都需要有css/js/partial以及主要的逻辑所在的.php文件，后续以demo中的文件组织结构进行描述。

## 组件实现细节简述

### 主要逻辑文件RelatedRecords.php

这一文件要直接放置于`widgets`的目录之下。

需要继承`Backend\Classes\FormWidgetBase`这一基类，同时我们需要实现一些关键的方法，以便完成逻辑以及正确显示组件。

下面，将会说明几个关键的方法。

#### init()

`init()`方法顾名思义，即初始化表单组件，一些组件自身需要进行的变量定义，参数定义等操作可以在这里编写。

在使用过程中，每个表单组件都或多或少有一些自定义的参数，譬如富文本编辑器的大小等，这些参数通过yaml文件配置，但是如何才能在表单组件中的读取到呢？

可以在`init()`中调用`fillFromConfig()`这一成员方法，通过数组的形式，将参数名传入，之后将会出现同名的成员变量，其值就是传入的参数名称。如果没有设定，那么也可以指定默认值。

为了完成需求，我们需要允许配置：

+ `titleField` `card的标题域数据库字段名`
+ `imageField` `card的图片域数据库字段名`
+ `contentField` `card的正文域数据库字段名`
+ `modelClass` `需要操作的Model名称`
+ `whereClause` `查询数据时的WHERE子句内容`
+ `recordsPerPage` `弹出列表页每页显示个数`

在实际使用中，在`models/yourmodel/`下的`fields.yaml`文件中指定使用这一表单组件时，可以通过指定这些参数的形式，影响展现以及效果，形如：

```
related_samples:
  tab: sample_tab
  type: relatedrecords
  titleField: title
  imageField: img
  contentField: content
  recordsPerPage: 10
  modelClass: Your\Plugin\Models\Sample
  whereClause: 'published = true'
```

#### render()

`render()`方法，即完成组件的渲染，如果已经设定好了模板中需要的值，那么只需要通过`makePartial`这一个成员方法即可对当前组件渲染完成。

实现以上两个方法之后，几乎这一组件就能基本上完成数据获取并展现的功能了。其他方法，也可以按照默认生成的文件的模式进行组织。

### 模板文件_relatedrecords.htm

模板文件作为一个[partial](http://octobercms.com/docs/cms/partials)，文件名需要以下划线开头，放置在组件的`partials`目录下。

由于实际要修改的数值是一个JSON格式的数组，提交时，实际上是作为表单的一对键值。为了完成表单提交，需要设定input标签的name属性。

那么问题来了，如何才能设定正确的name属性呢？

OctoberCMS框架在实现表单组件时，会根据表单组件所对应的Model和列的名称，生成key的名字，即input标签的name属性，可以通过成员变量`formField`的方法`getName()`获取。

在主逻辑文件`RelatedRecords.php`的`render()`（或者其他被这一方法调用的方法之中），为name属性赋值：

```
public function prepareData()
{
   $this->vars['value'] = $this->getLoadValue();
   ...
   $this->vars['name']           = $this->formField->getName();
   ...
   $this->vars['recordId']       = isset($this->model->id) ? $this->model->id : 0;
}

public function render()
{
   $this->addCss($this->widgetDir . '/assets/css/relatedrecords.css');
   $this->addJs($this->widgetDir . '/assets/js/relatedrecords.js');
   $this->prepareData();
   return $this->makePartial('field_relatedrecords');
}
```

在partial中进行相应变量赋值代码编写即可。

```
<div class="input-group">
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
</div>
```

而展现上，使用了部分[materialize](https://github.com/Dogfalo/materialize)的样式，实现了图文card的显示效果。

### 弹出列表模板文件_list_related_records_sample.htm

图文card的底部，有添加记录的按钮，但是如何才能最便捷的展现出可以加入的记录呢？

这里可以借用OctoberCMS内置的一个特性：[Remote popups](http://octobercms.com/docs/ui/popup)。

#### 实现简要说明

想要通过列表展示数据对于OctoberCMS来说简直是轻而易举。从操作步骤上来看，可以描述如下：

+ 点击`添加`
+ 触发Ajax请求，请求对应Controller上的方法
+ 对应方法获取指定数据，渲染partial返回

Ajax请求如何能够返回并显示html呢？作为OctoberCMS的一个重要特性，简要来说即通过特定的请求方法发出请求，指定操作的handler。这里可以通过js完成。

参见下的`assets/js/relatedrecords.js`文件：

```
var fetchRelatedRecords = function() {
   var $this = $(this);
   ...

   $(this).popup({
       handler: 'onLoadRelatedRecords',
       extraData: {
           page:           page,
           excludeIds:     excludeIds,
           recordsPerPage: CONFIG.recordsPerPage,
           modelClass:     CONFIG.modelClass,
           whereClause:    CONFIG.whereClause,
           titleField:     CONFIG.titleField,
           imageField:     CONFIG.imageField,
           contentField:   CONFIG.contentField
       }
   });
};
```

通过popup方法，指定了handler为`onLoadRelatedRecords`方法。

handler方法归属的Controller，即当前编辑对象所对应的
Controller，所以，需要在Controller中实现对应的同名方法。demo中编辑的是Sample这一个对象，那么方法实现起来将会类似：

```
class Sample extends Controller
{
    public function onLoadRelatedRecords()
    {
        $page           = Request::input('page');
        $recordsPerPage = Request::input('recordsPerPage');
        $excludeIds     = Request::input('excludeIds');
        $modelClass     = Request::input('modelClass');
        $whereClause    = Request::input('whereClause');
        if (!class_exists($modelClass)) {
            echo "{$modelClass} not exists";
        }
        /**
         * var $model | Model;
         */
        $model = new $modelClass();
        if (!is_array($excludeIds)) {
            $excludeIds = json_decode($excludeIds);
        }
        $records = $model->whereNotIn('id', $excludeIds);
        if ($whereClause) {
            $records = $records->whereRaw($whereClause);
        }
        $records = $records->orderBy('id', 'desc')->paginate($recordsPerPage, $page);
        return [
            'result' => $this->makePartial('list_related_records', [
                'records' => $records,
                'modelClass' => $modelClass
            ])
        ];
    }
}
```

考虑到不能重复加入关联记录，所以每次请求要通过`excludeIds`传入已加入的id，进行排除。

通过popup方式弹出列表之后，选择加入这些操作就相当容易实现了。

其他如前端的操作、样式的编写等等不在赘述。

### 注册表单组件

完成上述工作之后，还需要关键的一步，即在`Plugin.php`中的`registerFormWidgets()`方法注册编写完成的组件。

```
public function registerFormWidgets()
    {
        // register the widget
        // relatedrecords in fields.yaml will represent this form widget
        return [
            'Your\Plugin\Widgets\RelatedRecords' => [
                'label' => 'Related Records',
                'code'  => 'relatedrecords'
            ]
        ];
    }
```

## 组件效果

通过弹出的Modal窗口添加关联：

![select related records](https://github.com/liaoaoyang/RelatedRecordsFormWidgetDemo/blob/master/images/select_records.png?raw=true)

选择后可供调整顺序或移除：

![selected](https://github.com/liaoaoyang/RelatedRecordsFormWidgetDemo/blob/master/images/selected.png?raw=true)

