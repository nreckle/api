## Mainline specific patterns [ML] <a name="mainline"></a>

[Mainline](http://go/what-is-mainline) 是一个允许单独更新 `Android` 操作系统的子系统（“主线模块”）的项目，而不是更新整个系统映像。

主线模块必须与核心平台“**分离**”，这意味着每个模块与世界其他部分之间的所有交互都必须通过正式（**公共或系统**）`API` 完成。

主线模块应遵循某些设计模式。 本节介绍它们。

### `[Module]FrameworkInitializer` 模式

如果 `mainline` `module` 需要公开`@SystemService`类（例如`JobScheduler`)，使用以下模式：

-   从模块中公开`[YourModule]FrameworkInitializer` 类。 该类需要位于`$BOOTCLASSPATH` 中。 例子：
    [`StatsFrameworkInitializer`](https://cs.android.com/android/platform/superproject/+/master:packages/modules/StatsD/framework/java/android/os/StatsFrameworkInitializer.java)。

-   用`@SystemApi(client = MODULE_LIBRARIES)`标记它。 

-   添加一个 `public static void registerServiceWrappers()` 方法。

-   当需要引用`Context`时，使用`SystemServiceRegistry.registerContextAwareService()`来注册服务管理器类。

-   当不需要引用`Context`时，使用`SystemServiceRegistry.registerStaticService()`来注册服务管理器类。

-   使用 [`SystemServiceRegistry`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/SystemServiceRegistry.java) 的`registerServiceWrappers()` 方法来静态初始化。

### `[Module]ServiceManager` 模式

通常，为了注册系统服务`binder`对象和/或获取对它们的引用，可以使用 [`ServiceManager`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java)，但 `mainline` 模块无法使用它，因为它是隐藏的。 此类被隐藏，因为 `mainline`  模块不应该注册或引用由不可更新平台或其他模块公开的系统服务`binder`对象。

`mainline` 模块可以使用以下模式来注册并获取对模块内部实现的`binder`服务的引用。

-   创建一个 `[YourModule]ServiceManager` 类，遵循 [`TelephonyServiceManager`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/TelephonyServiceManager.java)的设计
-   将类公开为`@SystemApi`。 如果您只需要从`$BOOTCLASSPATH`类或`system server`类访问它，可以使用`@SystemApi(client=MODULE_LIBRARIES)`； 否则使用`@SystemApi(client=PRIVILEGED_APPS)`。
-   该类将包括：

    -   隐藏的构造函数，因此只有不可更新的平台代码才能实例化它。
    -   ~~嵌套类 `ServiceRegisterer`~~ `TODO(omakoto) b/245801672`：它不应该是内部类。 为此创建一个可共享的类。
    -   返回特定名称的`ServiceRegisterer`实例的公共 `getter` 方法。 如果您有一个`binder`对象，那么您需要一种 `getter` 方法。 如果你有两个，那么你需要两个 `getters`。
    -   在 [`ActivityThread.initializeMainlineModules()`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityThread.java) 中，实例化这个类，并将其传递给模块公开的静态方法。 通常，您可以在接受它的`FrameworkInitializer`类中添加一个静态`@SystemApi(client=MODULE_LIBRARIES)` `API`。

这种模式会阻止其他`mainline`模块访问这些 `API`，因为其他模块无法获取`[YourModule]ServiceManager`的实例，即使`get()`和`register()` `API` 对它们可见。

以下是电话 [^telephonymodule] 如何获取对电话服务的引用：[code search link](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/telephony/java/android/telephony/TelephonyManager.java)。

`telephonymodule` ：电话功能（还）不是`mainline`模块，但这仍然显示出首选模式。

如果您在`native`代码中实现服务`binder`对象，则可以使用 [`AServiceManager` 原生 `API`](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ndk/include_platform/android/binder_manager.h)。
这些 `API` 对应于`ServiceManager` `Java` `API`，但`native` `API` 直接暴露给`mainline`模块。 **不要**使用它们来注册或引用不属于您的模块的`binder`对象。 如果您从 `native` 中暴露 `binder` 对象，则您的`[YourModule]ServiceManager.ServiceRegisterer` 不需要`register()`方法。
