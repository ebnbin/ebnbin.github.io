---
layout: post
title: LeakCanary 2 初始化方式分析<br>——ContentProvider 妙用
tags: [Android]
---

LeakCanary 2 是一次重大的版本升级。与旧版本相比，依赖与配置 LeakCanary 库的方式变得更加简单，只需要在 app 模块的 `build.gradle` 文件中添加：

``` gradle
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.3'
}
```

官方文档也特别突出了这一点：

> **That’s it, there is no code change needed!**

本篇文章结合 [LeakCanary v2.3 源码](https://github.com/square/leakcanary/tree/v2.3)，分析其“无代码侵入”初始化库的实现方式，以及在自己的库中使用这一特性的注意事项。

* * *

## 使用 `ContentProvider` 初始化库

我们可以在 `leakcanary-object-watcher-android` 模块的 `AndroidManifest.xml` 文件中发现注册了一个 `provider`：

``` xml
<provider
    android:name="leakcanary.internal.AppWatcherInstaller$MainProcess"
    android:authorities="${applicationId}.leakcanary-installer"
    android:exported="false"/>
```

其源码如下：

``` kotlin
/**
 * Content providers are loaded before the application class is created. [AppWatcherInstaller] is
 * used to install [leakcanary.AppWatcher] on application start.
 */
internal sealed class AppWatcherInstaller : ContentProvider() {

  /**
   * [MainProcess] automatically sets up the LeakCanary code that runs in the main app process.
   */
  internal class MainProcess : AppWatcherInstaller()

  /**
   * When using the `leakcanary-android-process` artifact instead of `leakcanary-android`,
   * [LeakCanaryProcess] automatically sets up the LeakCanary code
   */
  internal class LeakCanaryProcess : AppWatcherInstaller() {
    override fun onCreate(): Boolean {
      super.onCreate()
      AppWatcher.config = AppWatcher.config.copy(enabled = false)
      return true
    }
  }

  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    InternalAppWatcher.install(application)
    return true
  }

  override fun query(
    uri: Uri,
    strings: Array<String>?,
    s: String?,
    strings1: Array<String>?,
    s1: String?
  ): Cursor? {
    return null
  }

  override fun getType(uri: Uri): String? {
    return null
  }

  override fun insert(
    uri: Uri,
    contentValues: ContentValues?
  ): Uri? {
    return null
  }

  override fun delete(
    uri: Uri,
    s: String?,
    strings: Array<String>?
  ): Int {
    return 0
  }

  override fun update(
    uri: Uri,
    contentValues: ContentValues?,
    s: String?,
    strings: Array<String>?
  ): Int {
    return 0
  }
}
```

`MainProcess` 是 `AppWatcherInstaller` 的一个内部类，继承了 `AppWatcherInstaller`，用于在主进程注册。`AppWatcherInstaller` 中还有一个 `LeakCanaryProcess` 内部类，同样继承了 `AppWatcherInstaller`，用于在 `:leakcanary` 进程注册。

`AppWatcherInstaller` 是一个 `ContentProvider`，却并没有任何 CRUD 操作，只在 `onCreate()` 中获取应用 `Application` 并在第 26 行执行了 LeakCanary 的初始化操作 `InternalAppWatcher.install(application)`。这相当于 LeakCanary 旧版本中需要我们手动调用的 `LeakCanary.install(Application)` 方法。

如果打 log 可以发现，`ContentProvider` 的 `onCreate` 方法要早于 `Application` 的 `onCreate` 方法执行。相关方法的调用顺序是：

1. `Application.attachBaseContext`

2. `ContentProvider.onCreate`

3. `Application.onCreate`

因此在 `ContentProvider` 的 `onCreate` 中初始化库是可行的。LeakCanary 2 利用了这一特性完成了初始化，同时避免了代码侵入。

不得不说，这并不是 `ContentProvider` 的“正确”打开方式，不过用起来，真香！

* * *

## 初始化依赖问题

上面的分析已经有很多文章做过了，不过在实践过程中发现一个小问题。

我自己需要一个 base 库，上层 app 依赖这个 base 库。像 LeakCanary 这样的基础工具库可以放在 base 库中，不需要让每个使用库的 app 分别依赖。而我的 base 库也希望像 LeakCanary 2 一样，使用 `ContentProvider` 做初始化，于是乎我添加了一个 `BaseInstaller`：

``` kotlin
/**
 * 用于获取应用 [Application] 并初始化库.
 */
internal class BaseInstaller : ContentProvider() {
    override fun onCreate(): Boolean {
        // 自定义配置 LeakCanary.
        initLeakCanary()
        // 其他初始化操作.
        // ...
        return true
    }

    private fun initLeakCanary() {
        // 不显示 LeakCanary 启动器图标.
        LeakCanary.showLeakDisplayActivityLauncherIcon(false)
    }

    // ...
}
```

在我自己的 `BaseInstaller` 中我对 LeakCanary 做了一些额外的自定义配置工作。

运行起来应用却崩溃了，异常信息是：

``` logcat
Caused by: kotlin.UninitializedPropertyAccessException: lateinit property application has not been initialized
   at leakcanary.internal.InternalLeakCanary.setEnabledBlocking(InternalLeakCanary.kt:330)
   at leakcanary.LeakCanary.showLeakDisplayActivityLauncherIcon(LeakCanary.kt:337)
```

根据异常栈大概能猜到原因了，在 `BaseInstaller` 中配置 LeakCanary 时，LeakCanary 还没有完成初始化。

在 `InternalLeakCanary` 中可以看到，`application` 是一个 `lateinit` 字段，在 `invoke(Application)` 方法中第一次赋值，`InternalLeakCanary` 实现了 `(Application) -> Unit` 接口，在 `InternalAppWatcher` 的 `install(Application)` 方法中调用。

``` kotlin
internal object InternalLeakCanary : (Application) -> Unit, OnObjectRetainedListener {
  // ...

  lateinit var application: Application

  // ...

  override fun invoke(application: Application) {
    this.application = application
    // ...
  }

  // ...

  fun setEnabledBlocking(
    componentClassName: String,
    enabled: Boolean
  ) {
    val component = ComponentName(application, componentClassName)
    // ...
  }

  // ...
}
```

之所以抛“未初始化”异常是因为，我们的 `base` 库同样通过注册一个 `ContentProvider` 的方式实现初始化，而 `base` 库依赖了 LeakCanary，相当于在 app 的 `AndroidManifest.xml` 中注册了两个 `ContentProvider`。根据依赖关系，`base` 库中的自定义 `ContentProvider` 会优先于 LeakCanary 中的初始化，因此在 `BaseInstaller` 中配置 LeakCanary 的时机过早。

如何解决这一问题？`provider` 在 `AndroidManifest.xml` 中注册时提供一条属性 `android:initOrder`，可以用来指定初始化顺序。

我们可以先查看一下所有已注册的 `ContentProvider` 以及它们的 `initOrder`：

``` kotlin
context.packageManager
    .getPackageInfo(context.packageName, PackageManager.GET_PROVIDERS)
    .providers
    .forEach {
        log("${it.name}: ${it.initOrder}")
    }
```

输出：

``` logcat
D: *.BaseInstaller: 0
D: leakcanary.internal.LeakCanaryFileProvider: 0
D: leakcanary.internal.AppWatcherInstaller$MainProcess: 0
```

可以看到，第一条是 `base` 库的 `BaseInstaller`，之后有 LeakCanary 的 `FileProvider` 以及我们上面分析的 `AppWatcherInstaller$MainProcess`。默认情况下 `provider` 的 `android:initOrder` 属性值为 0，数值越大越早执行初始化，因此为了保证 LeakCanary 优先初始化完成，我们只要为 `BaseInstaller` 指定一个小于 0 的数：

``` xml
<provider
    android:name=".BaseInstaller"
    android:authorities="..."
    android:exported="false"
    android:initOrder="-1"/>
```

如果依赖了更多的使用 `ContentProvider` 的库并且存在初始化顺序问题，只需要保证被依赖的 `provider` 的 `android:initOrder` 大于调用者的值就可以了。

* * *

## 小结

* `ContentProvider` 的初始化早于 `Application` 的 `onCreate` 且可以由系统自行注册调用，LeakCanary 2 使用了这一特性用来执行初始化操作。

* 在使用自定义 `ContentProvider` 初始化其他第三方库时需要注意依赖关系，使用 `android:initOrder` 属性可以改变初始化顺序。
