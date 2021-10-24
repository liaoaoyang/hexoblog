title: IDEA插件开发尝试
date: 2021-10-23 13:57:57
categories: Try
tags: [Java,IDEA,插件]
---

# TL;DR

JetBrains IDEA 基本上是目前最好的 Java 开发 IDE ，根据自己的一些个性化需求定制插件也是可行的。本篇会通过修改一个开源插件的的方式，描述一些简单修改插件的方法以及可能遇到的问题。

此处感谢 [ArthasHotSwap](https://github.com/xxxtai/ArthasHotSwap/) 插件，这一个插件为开发工作提供了便利。

<!-- more -->

# 场景

此处以一个场景作为案例。

[ArthasHotSwap](https://github.com/xxxtai/ArthasHotSwap/) 插件是一个使用 Aliyun OSS 作为加密后 class 文件作为中转，近似“一键”完成热更新操作的 IDEA 插件。

插件功能很简单，仅在右侧菜单提供一个 `Swap this class` 选项：

![Swap this class](https://blog.wislay.com/wp-content/uploads/2021/10/WX20211023-141000.png)

查看插件源代码：

```java
// com.xxxtai.arthas.facade.impl.OssFacadeImpl#uploadString
OSS ossClient = new OSSClientBuilder().build(ossInfo.endpoint, ossInfo.accessKeyId, ossInfo.accessKeySecret);
PutObjectRequest putObjectRequest = new PutObjectRequest(ossInfo.bucketName, DIRECTORY + key,
    new ByteArrayInputStream(content.getBytes()));

ObjectMetadata metadata = new ObjectMetadata();
metadata.setHeader(OSSHeaders.OSS_STORAGE_CLASS, StorageClass.Standard.toString());
metadata.setObjectAcl(CannedAccessControlList.PublicRead);
putObjectRequest.setMetadata(metadata);

ossClient.putObject(putObjectRequest);
ossClient.shutdown();
```

可以看到实际上是生成了一个 OSS 上的公共读对象，通过复杂的文件名来提供一定的安全性，并且满足可以在服务器上访问的的链接。

那么下面修改的目标就是`提供一个选项，决定是否生成带有过期时间的链接`。

# 开发

## 开发环境搭建

需求明确后先要搭建开发环境。

[ArthasHotSwap](https://github.com/xxxtai/ArthasHotSwap/) 插件使用 gradle 进行工程的组织。

使用 gradle 可能会遇到依赖下载速度缓慢的问题，考虑将 `build.gradle` 文件中调整一下源：

```
repositories {
    // mavenCentral()
    maven{ url 'https://maven.aliyun.com/repository/central/'}
    maven{ url 'https://maven.aliyun.com/repository/public/'}
    maven{ url 'https://maven.aliyun.com/repository/google/'}
    maven{ url 'https://maven.aliyun.com/repository/gradle-plugin/'}
}
```

开发插件还会用到 [gradle-intellij-plugin](https://github.com/JetBrains/gradle-intellij-plugin/) 。在配置文件中声明的版本可能也需要下载，这个速度也可能很慢，考虑直接替换成本地的 IDEA：

```
// See https://github.com/JetBrains/gradle-intellij-plugin/
intellij {
    // version '2020.3.3'
    // type 'IC'
    version '2021.2'
    localPath '/Users/l/Library/Application Support/JetBrains/Toolbox/apps/IDEA-U/ch-0/212.4746.92/IntelliJ IDEA.app'
    updateSinceUntilBuild false
    sameSinceUntilBuild false
}
```

其中 `localPath` 替换成自己的目录即可，`version` 也是必须配置的。

这两步完成后可能可以加快投入开发的时间。


## 插件结构

开发首先看文档：https://plugins.jetbrains.com/docs/intellij/plugin-structure.html

关注 `src/main/resources/META-INF/plugin.xml`。

一些常规的字段之外，本次修改最关键的部分，就是配置窗口的声明：

```
<projectConfigurable parentId="tools" instance="com.xxxtai.arthas.dialog.SettingDialog"
                     id="com.xxxtai.arthas.dialog.SettingDialog" displayName="ArthasHotSwap"/>
<projectService serviceImplementation="com.xxxtai.arthas.domain.AppSettingsState"/>
```

从这里可以看到入口是 `com.xxxtai.arthas.dialog.SettingDialog` 这个类。

同时因为目标是提供修改配置的功能，参考文档：https://plugins.jetbrains.com/docs/intellij/settings-guide.html 。

结合文档以及配置，[ArthasHotSwap](https://github.com/xxxtai/ArthasHotSwap/) 插件提供的是项目级别的配置功能，并且目录中位于 Tools 目录下：

![ArthasHotSwap](https://blog.wislay.com/wp-content/uploads/2021/10/WX20211023-145110.png)

## 修改方法

增加一个配置，整体方法就明确了：

+ 选择合适的显示组件
+ 增加配置字段
+ 功能层面读取字段完成功能分支

### 选择合适的显示组件

可以考虑 CheckBox 形式，勾选上表示 true。

注意 `com.xxxtai.arthas.dialog.SettingDialog` 是配置文件中的入口，而此类会调用 `com.xxxtai.arthas.dialog.AppSettingsComponent` 对象构建面板。

修改 `com.xxxtai.arthas.dialog.AppSettingsComponent` ，增加对象以及获取以及赋值方法：

```java
public class AppSettingsComponent {
    private final JPanel myMainPanel;
    private final JBTextField ossEndpointText = new JBTextField();
    // ...
    private final JBCheckBox privateAccessRadioButton = new JBCheckBox();

    public AppSettingsComponent() {
        myMainPanel = FormBuilder.createFormBuilder()
            .addLabeledComponent(new JBLabel("Enter OSS Endpoint: "), ossEndpointText, 1, false)
            // ...
            .addLabeledComponent(new JBLabel("Private OSS access: "), privateAccessRadioButton, 1, false)
            .addComponentFillVertically(new JPanel(), 0)
            .getPanel();
    }

    public JPanel getPanel() {
        return myMainPanel;
    }

    public JComponent getPreferredFocusedComponent() {
        return ossEndpointText;
    }

    @NotNull
    public String getOssEndpointText() {
        return ossEndpointText.getText();
    }

    public void setOssEndpointText(@NotNull String newText) {
        ossEndpointText.setText(newText);
    }

    // ...
    
    public boolean getPrivateAccess() {
        return privateAccessRadioButton.isSelected();
    }

    public void setPrivateAccess(boolean privateAccess) {
        privateAccessRadioButton.setSelected(privateAccess);
    }
}

```

### 增加配置字段

`com.xxxtai.arthas.dialog.AppSettingsComponent` 会在 `com.xxxtai.arthas.dialog.SettingDialog` 中通过配置文件中的 `com.xxxtai.arthas.domain.AppSettingsState` 读取和存储配置文件。

```
@State(
    name = "com.xxxtai.arthas.domain.AppSettingsState",
    storages = {@Storage("setting.xml")}
)
public class AppSettingsState implements PersistentStateComponent<AppSettingsState> {

    public String endpoint = "";
    public String accessKeyId = "";
    public String accessKeySecret = "";
    public String bucketName = "";
    public String selectJavaProcessName = "";
    public String specifyJavaHome = "";
    public boolean privateAccess = false;

    public static AppSettingsState getInstance(@NotNull Project project) {
        return ServiceManager.getService(project, AppSettingsState.class);
    }

    @Nullable
    @Override
    public AppSettingsState getState() {
        return this;
    }

    @Override
    public void loadState(@NotNull AppSettingsState state) {
        XmlSerializerUtil.copyBean(state, this);
    }

}
```

增加变量之余，上面的 `@State` 注解也声明了存储位置。

### 功能层面读取字段完成功能分支

此处不多描述，即 `com.xxxtai.arthas.facade.impl.OssFacadeImpl#uploadString` 会给脚本返回 OSS 对象链接，需要加密也是需要在此处理。即读取配置后，决定是否返回加密链接。


# 测试

逻辑开发完成之后，项目 gradle 任务中有一个 `runIde` 任务，运行此任务之后，会启动一个测试 IDEA 实例，可以在此 IDE 中进行功能的测试。

此处可见刚才的私有访问配置已经添加成功：

![Added checkbox](https://blog.wislay.com/wp-content/uploads/2021/10/WX20211023-153849.png)

配置也存储到`@State` 注解配置对应文件中：

![config saved](https://blog.wislay.com/wp-content/uploads/2021/10/WX20211023-154020.png)

至此，一个插件的修改过程结束。安装上可以通过 gradle 打包 fatJar 再进入 IDEA 进行安装即可。

至于如何增加全局配置等其他功能的开发，之后有时间再记录。

# 参考链接

+ [ArthasHotSwap](https://github.com/xxxtai/ArthasHotSwap/)
+ [gradle-intellij-plugin](https://github.com/JetBrains/gradle-intellij-plugin/)
+ https://plugins.jetbrains.com/docs/intellij/plugin-structure.html
+ https://plugins.jetbrains.com/docs/intellij/settings-guide.html

