## Android 框架特定规则 [`FW`] <a name="framework"></a>

这些规则涉及 `API`、模式和数据结构，这些规则特定于 `Android` 框架中内置的 `API` 和功能（`Bundle`、`Parcelable`等）。

### `Intent`构建器应使用`create*Intent()`模式 <a name="framework-intent-builder"></a>

意图的创建者应该使用名为`createFooIntent()`的方法。

### 使用`Bundle`而不是创建新的通用数据结构 <a name="framework-bundle"></a>

不要创建新的类型/类来保存各种参数或各种类型，而是考虑简单地使用`Bundle`。

### `Parcelable` 实现者必须具有公共 `CREATOR` 字段 <a name="framework-parcelable-creator"></a>

`Parcelable` 填充是通过 `CREATOR` 公开的，而不是原始构造函数。 如果一个类实现了`Parcelable`，那么它的`CREATOR`字段也必须是公共 `API`，并且采用`Parcel`参数的类构造函数必须是私有的。

### 对 `UI` 字符串使用`CharSequence` <a name="framework-charsequence-ui"></a>

当字符串出现在用户界面中时，使用`CharSequence`来允许`Spannable`。

如果它只是一个键或其他一些用户不可见的标签或值，`String`就可以了。

### 避免使用 `Enums` <a name="framework-avoid-enum"></a>

[`IntDef`](https://developer.android.com/reference/kotlin/androidx/annotation/IntDef)s 必须在所有平台 `API` 中使用而不是 `enum`s, 在非捆绑的库 `API` 中应重点考虑。 仅当您确定不会添加新值时才使用枚举。

`IntDef`的好处

*   随着时间的推移可以增加值
    *   `Kotlin` `when` 语句可以[运行时失败](https://youtrack.jetbrains.com/issue/KT-30473) 如果由于平台中添加了枚举值而变得不再详尽。
*   运行时不使用类/对象，仅使用基本类型
    *   虽然 `R8 / Minfication` 可以避免非捆绑库 `API` 的这种成本，但这种优化不会影响平台 `API` 类。

枚举的好处

*   `Java`、`Kotlin` 的惯用语言特性
*   启用详尽的 `switch`、`when` 语句的使用
    * 注意 - 值不得随时间变化，见上文
*   范围清晰、可发现的命名
*   启用编译时验证
    * 例如 `kotlin` 中返回值的 `when` 语句
*   是一个功能类，可以实现接口、具有静态帮助器、公开成员/扩展方法、字段。

### 遵循`Android`包分层结构 <a name="framework-package-layering"></a>

`android.*` 包层次结构具有隐式排序，其中较低级别的包不能依赖于较高级别的包。

### 避免提及 Google、其他公司及其产品 <a name="framework-mentions-google"></a>

`Android` 平台是一个开源项目，旨在保持供应商中立。`API` 应该是通用的，并且具有必要权限的系统集成商或应用程序同样可以使用。

### `Parcelable` 的实现者应该是 `final` <a name="framework-parcelable-final"></a>

平台定义的 `Parcelable` 类始终从`framework.jar`加载，因此应用程序尝试覆盖`Parcelable`实现是无效的。

如果发送应用程序扩展了`Parcelable`，则接收应用程序将无法使用发送者的自定义实现来解包。 请注意向后兼容性：如果您的类历史上不是最终的，但没有公开可用的构造函数，您仍然可以将其标记为`final`。

### 调用系统进程的方法应将`RemoteException`重新抛出为`RuntimeException` <a name="framework-rethrow-remoteexception"></a>

`RemoteException` 通常由内部 `AIDL` 抛出，表示系统进程已终止，或者应用程序正在尝试发送过多数据。 在这两种情况下，公共 `API` 都应该作为`RuntimeException`重新抛出，以确保应用程序不会意外地保留安全或策略决策。

如果您知道`Binder`调用的另一端是系统进程，那么这个简单的样板代码是最佳实践：

```java {.good}
try {
    ...
} catch (RemoteException e) {
    throw e.rethrowFromSystemServer();
}
```

### 实现复制构造函数而不是`clone()`<a name="framework-avoid-clone"></a>

由于缺乏`Object`类提供的 `API` 保证以及扩展使用`clone()`的类所固有的困难，强烈建议不要使用 `Java``clone()`方法。 相反，请使用采用相同类型的对象的复制构造函数。

```java {.good}
/**
 * Constructs a shallow copy of {@code other}.
 */
public Foo(Foo other)
```

依赖 Builder 进行构造的类应考虑添加 Builder 复制构造函数以允许对副本进行修改。

```java {.good}
public class Foo {
    public static final class Builder {
        /**
         * Constructs a Foo builder using data from {@code other}.
         */
        public Builder(Foo other)
```

### 使用 `ParcelFileDescriptor` 而不是 `FileDescriptor`. <a name="framework-parcelfiledescriptor"></a>

`java.io.FileDescriptor`对象的所有权定义很差，这可能会导致模糊的关闭后使用错误。 相反，`API` 应该返回或接受`ParcelFileDescriptor`实例。 如果需要，旧代码可以在 `PFD` 和 `FD` 之间进行转换，可以使用
[dup()](https://developer.android.com/reference/android/os/ParcelFileDescriptor.html##dup\(java.io.FileDescriptor\))
或者
[getFileDescriptor()](https://developer.android.com/reference/android/os/ParcelFileDescriptor.html##getFileDescriptor\(\))。

### Avoid using odd-sized numerical values. <a name="framework-avoid-short-byte"></a>

避免直接使用`short`或`byte`值，因为它们通常会限制您将来开发 `API` 的方式。

### 避免使用 `BitSet`。 <a name="framework-avoid-bitset"></a>

`java.util.BitSet` 非常适合实现，但不适用于公共 `API`。 它是可变的，需要为高频方法调用进行分配，并且不提供每个位代表的语义。

对于高性能场景，请将`int`或`long`与`@IntDef`一起使用。 对于低性能场景，请考虑`Set<EnumType>`。 对于原始二进制数据，请使用`byte[]`。

### Prefer `android.net.Uri`. <a name="framework-android-uri"></a>

`android.net.Uri` 是 Android API 中 URI 的首选封装。

避免使用`java.net.URI`，因为它在解析 URI 方面过于严格，并且永远不要使用`java.net.URL`，因为它的相等性定义被严重破坏。

### 隐藏标记为 `@IntDef`, `@LongDef` 或 `@StringDef`的注释 <a name="framework-hide-typedefs"></a>

标记为`@IntDef`、`@LongDef`或`@StringDef`的注释表示一组可以传递给 `API` 的有效常量。 但是，当它们本身作为 `API` 导出时，编译器会内联常量，并且只有（现在无用的）值保留在注释的 `API` 存根（对于平台）或 `JAR`（对于库）中。

因此，这些注释的使用必须在平台中标记为`@hide`，或者在库中标记为`@hide`和`RestrictTo.Scope.LIBRARY`。 在这两种情况下，它们都必须标记为`@Retention(RetentionPolicy.SOURCE)`，以确保它们不会出现在 `API` 存根或 `JAR` 中。

```java
/** @hide */
@RestrictTo(RestrictTo.Scope.LIBRARY)
@Retention(RetentionPolicy.SOURCE)
@IntDef({
  STREAM_TYPE_FULL_IMAGE_DATA,
  STREAM_TYPE_EXIF_DATA_ONLY,
})
public @interface ExifStreamType {}
```

构建平台 `SDK` 和库 `AAR` 时，工具会提取注释并将其与编译源分开捆绑。 `Android Studio` 读取此捆绑格式并强制执行类型定义。

### 不要添加新的设置提供程序密钥 <a name="framework-avoid-settings"></a>

不要暴露新的密钥从
[`Settings.Global`](https://developer.android.com/reference/android/provider/Settings.Global),
[`Settings.System`](https://developer.android.com/reference/android/provider/Settings.System),
或者
[`Settings.Secure`](https://developer.android.com/reference/android/provider/Settings.Secure).

相反，在相关类中添加适当的 `getter` 和 `setter Java API`，即通常是"`manager`"类。 添加监听机制或者广播来通知以根据需要通知客户端更改。

相比之下，`SettingsProvider`设置相比于`getters/setters`存在许多问题：

* 无类型安全。
* 没有统一的方式提供默认值。
* 没有正确的方法来自定义权限。
  * 例如，无法使用自定义权限保护您的设置。
* 没有适当的方法来正确添加自定义逻辑。
  * 例如，无法根据设置 B 的值来更改设置关联的 A 的值。。

例子：
[`Settings.Secure.LOCATION_MODE`](https://developer.android.com/reference/android/provider/Settings.Secure##LOCATION_MODE) 已经存在很长时间了，但定位团队已弃用它一段时间了，正确的`Java API`
[`LocationManager.isLocationEnabled()`](https://developer.android.com/reference/android/location/LocationManager##isLocationEnabled\(\)) 和 [`MODE_CHANGED_ACTION`](https://developer.android.com/reference/android/location/LocationManager##MODE_CHANGED_ACTION) 广播，这给了团队更大的灵活性，并且语义 `API` 现在更加清晰了。

### 不要扩展 `Activity` 和 `AsyncTask` <a name="framework-forbidden-super-class"></a>

`AsyncTask` 是一个实现细节。 相反，公开一个侦听器，或者在 `androidx` 中公开一个`ListenableFuture` API。

`Activity`子类是不可能组合的。 扩展功能的活动会使其与需要用户执行相同操作的其他功能不兼容。 相反，通过使用诸如[`LifecycleObserver`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-common/src/main/java/androidx/lifecycle/LifecycleObserver.java;l=26?q=LifecycleObserver&ss=androidx%2Fplatform%2Fframeworks%2Fsupport).

### 使用 `Context`'s `getUser()` <a name="framework-context-user"></a>

绑定到`Context`的类（例如从`Context.getSystemService()`返回的任何内容）应该使用绑定到`Context`的用户，而不是公开针对特定用户的成员。

```java {.good}
class FooManager {
  Context mContext;

  void fooBar() {
    mIFooBar.fooBarForUser(mContext.getUser());
  }
}
```

```java {.bad}
class FooManager {
  Context mContext;

  Foobar getFoobar() {
    // Bad: doesn't appy mContext.getUserId().
    mIFooBar.fooBarForUser(Process.myUserHandle());
  }

  Foobar getFoobar() {
    // Also bad: doesn't appy mContext.getUserId().
    mIFooBar.fooBar();
  }

  Foobar getFoobarForUser(UserHandle user) {
    mIFooBar.fooBarForUser(user);
  }
}
```

例外：方法可能接受不代表单个用户的值，例如`UserHandle.ALL`

### 使用 `UserHandle` 而不是普通的 `int`s <a name="framework-userhandle"></a>

首选 `UserHandle` 以确保类型安全并避免将用户 `ID` 与 `uid` 混淆。

```java {.good}
Foobar getFoobarForUser(UserHandle user);
```

```java {.bad}
Foobar getFoobarForUser(int userId);
```

如果不可避免，表示用户 ID 的 int 必须使用 @UserIdInt 进行注释。

```java
Foobar getFoobarForUser(@UserIdInt int user);
```

### 优先选择监听器/回调而不是广播意图 <a name="framework-avoid-broadcast"></a>

广播意图非常强大，但它们导致了可能对系统健康产生负面影响的紧急行为，因此应明智地添加新的广播意图。

以下是一些导致我们不鼓励引入新广播意图的具体问题：

*   当发送没有`FLAG_RECEIVER_REGISTERED_ONLY`标志的广播时，它们将强制启动任何尚未运行的应用程序。 虽然这有时可能是想要的结果，但它可能会导致数十个应用程序崩溃，从而对系统健康产生负面影响。 我们建议使用替代策略，例如`JobScheduler`，以便在满足各种先决条件时更好地进行协调。
*   发送广播时，几乎无法过滤或调整传递给应用程序的内容。 这使得响应未来的隐私问题或基于接收应用程序的目标 SDK 引入行为更改变得困难或不可能。
*   由于广播队列是共享资源，因此它们可能会过载，并且可能无法及时传送您的事件。 我们观察到多个广播队列的端到端延迟达到 10 分钟或更长。

出于这些原因，我们鼓励新功能考虑使用侦听器/回调或其他设施，例如`JobScheduler`，而不是
广播意图。

如果广播意图仍然是理想设计，则应考虑以下一些最佳实践：

*   如果可能，请使用`Intent.FLAG_RECEIVER_REGISTERED_ONLY`将广播限制为已在运行的应用程序。 例如，`ACTION_SCREEN_ON` 使用这种设计来避免唤醒应用程序。
*   如果可能，请使用`Intent.setPackage()`或`Intent.setComponent()`将广播定位到感兴趣的特定应用程序。 例如，`ACTION_MEDIA_BUTTON`使用此设计来专注于当前应用程序处理播放控件。
*   如果可能，请将您的广播定义为`<protected-broadcast>`，以确保恶意应用程序无法冒充操作系统。
