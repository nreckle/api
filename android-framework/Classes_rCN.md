## Classes [CL] <a name="classes"></a>

这些是关于类、接口和继承的规则。

### 从适当的基类继承新的公共类 <a name="classes-inheritance"></a>

继承会在子类中公开 `API`，这可能不合适。 例如，`FrameLayout`的新公共子类将看起来像`FrameLayout`
（加上新功能/`API`）。 如果继承的 `API` 不适合您的用例，请从树的更上层继承（例如，`ViewGroup`甚至`View`，而不是`FrameLayout`）。

如果您想重写基类中的方法以抛出`@UnsupportedOperationException`，请重新考虑您正在使用哪个基类。

### 使用基集合类 <a name="classes-collections"></a>

无论是将集合作为参数还是将其作为值返回，始终优先选择基类而不是特定的实现（例如返回 `List<Foo>` 而不是 `ArrayList<Foo>`）。

使用表达 `API` 适当约束的基类。 例如，其集合必须有序的 `API` 应使用`List`，而其集合必须由唯一元素组成的 `API` 应使用`Set`。

在 `Kotlin` 中，更喜欢不可变集合。 看[集合可变性](#methods-collections-mutability) 了解更多详细信息。

### 抽象类 vs 接口 <a name="classes-interfaces"></a>

Java 8 添加了对默认接口方法的支持，这允许 `API` 设计者向接口添加方法，同时保持二进制兼容性。 平台代码和所有 `Jetpack` 库应针对 `Java 8` 或更高版本。

在默认实现是无状态的情况下，`API` 设计者“应该”更喜欢接口而不是抽象类——也就是说，默认接口方法可以通过调用其他接口方法来实现。

如果默认实现需要构造函数或内部状态，则*必须*使用抽象类。

在这两种情况下，`API` 设计者都可以选择保留单个方法抽象以简化 `lambda` 的使用。

```java {.good}
public interface AnimationEndCallback {
  // Always called, must be implemented.
  public void onFinished(Animation anim);
  // Optional callbacks.
  public default void onStopped(Animation anim) { }
  public default void onCanceled(Animation anim) { }
}
```

### 类名应该反映它们扩展的内容 <a name="classes-subclass-naming"></a>

例如，为了清楚起见，扩展`Service`的类应命名为`FooService`。

```java {.bad}
public class IntentHelper extends Service {}
```

```java {.good}
public class IntentService extends Service {}
```

#### 通用后缀 (`Helper`, `Util`, etc.) <a name="classes-generic-naming"></a>

避免使用`Helper`和`Util`等通用类名后缀来表示实用方法的集合。 相反，请将方法直接放入关联的类或 `Kotlin` 扩展函数中。

如果方法桥接多个类，请为包含的类指定一个有意义的名称来解释它的作用。

在*非常*有限的情况下，使用`Helper`后缀可能是合适的：

- 用于默认行为的组合
- 可能涉及将现有行为委托给新类
- 可能需要持久状态
- 通常涉及*视图*

例如，如果向后移植工具提示需要保留与`View`关联的状态并调用`View`上的多个方法来安装向后移植，则`TooltipHelper`将是可接受的类名。

### 不要将 `AIDL` 生成的代码直接公开为公共 `API`<a name="classes-wrap-aidl"></a>

保留 `AIDL` 生成的代码作为实现细节。 生成的 `AIDL` 类不符合 `API` 样式指南要求（例如，它们不能使用重载），并且不能保证保持语言 `API` 兼容性，因此我们无法将它们嵌入到公共 `API` 中。

相反，应在 `AIDL` 接口之上添加一个公共 `API` 层，即使它最初是一个浅层包装器。

#### `Binder` 接口 <a name="classes-wrap-binder"></a>

如果`Binder`接口是一个实现细节，那么将来可以自由更改它，公共层允许保持所需的向后兼容性。 例如，您可能会发现需要向内部调用添加新参数，或者通过批处理/流、使用共享内存或类似方式优化 `IPC` 流量。 如果您的 `AIDL` 接口也是公共 `API`，那么目前这些都无法完成。

例如，不要直接将 `FooService` 公开为公共 `API`：

```java {.bad}
// BAD: Public API generated from IFooService.aidl
public class IFooService {
   public void doFoo(String foo);
}
```

而是将`Binder`接口包装在管理器或其他类中：

```java {.good}
/**
 * @hide
 */
public class IFooService {
   public void doFoo(String foo);
}

public IFooManager {
   public void doFoo(String foo) {
      mFooService.doFoo(foo);
   }
}
```

如果稍后此调用需要新参数，则内部接口可以保持简单，并且可以将方便的重载添加到公共 `API` 中。 随着实现的发展，包装层也可用于处理其他向后兼容性问题：

```java {.good}
/**
 * @hide
 */
public class IFooService {
   public void doFoo(String foo, int flags);
}

public IFooManager {
   public void doFoo(String foo) {
      if (mAppTargetSdkLevel < 26) {
         useOldFooLogic(); // Apps targeting API before 26 are broken otherwise
         mFooService.doFoo(foo, FLAG_THAT_ONE_WEIRD_HACK);
      } else {
         mFooService.doFoo(foo, 0);
      }
   }

   public void doFoo(String foo, int flags) {
      mFooService.doFoo(foo, flags);
   }
}
```

对于不属于Android平台的`Binder`接口（例如，Google Play Services导出供应用程序使用的服务接口），对稳定的、已发布的、版本化的`IPC`接口的要求意味着它更难改进界面本身。 然而，在它周围有一个包装层仍然是值得的，以匹配其他 `API` 指南，并使得在新版本的 `IPC` 接口中使用相同的公共 `API` 变得更容易（如果有必要的话）。

### 不要使用 `CompletableFuture` 或者 `Future` <a name="classes-avoid-future"></a>

`java.util.concurrent.CompletableFuture`是一个很大的 `API` 接口，允许任意改变 `future` 的值，并且有容易出错的默认值。

相反，`java.util.concurrent.Future` 缺少非阻塞监听，因此很难与异步代码一起使用。

在**平台代码**中，首选完成回调、`Executor`以及 `API` 支持取消的`CancellationSignal`的组合。

```java {.good}
public void asyncLoadFoo(android.os.CancellationSignal cancellationSignal,
    Executor callbackExecutor,
    android.os.OutcomeReceiver<FooResult, Throwable> callback);
```

在**库和应用程序**中，更喜欢 `Guava` 的 `ListenableFuture`。

```java {.good}
public com.google.common.util.concurrent.ListenableFuture<Foo> asyncLoadFoo();
```

如果您的目标是 `Kotlin`，那么更喜欢`suspend`函数。

```kotlin {.good}
suspend fun asyncLoadFoo(): Foo
```

### 不要使用 `Optional` <a name="classes-avoid-optional"></a>

虽然`Optional`在某些 `API` 方面具有优势，但它与现有的 `Android` `API` 方面不一致。 `@Nullable` 和 `@NonNull` 为 `null` 安全性提供工具帮助，而 `Kotlin` 在编译器级别强制执行可空性契约，从而使 `Optional` 变得不必要。

对于可选基元，请使用配对的`has`和`get`方法。 如果未设置该值（即`has`返回`false`），则`get`方法应抛出`IllegalStateException`。

```java {.good}
public boolean hasAzimuth() { ... }
public int getAzimuth() {
  if (!hasAzimuth()) {
    throw new IllegalStateException("azimuth is not set");
  }
  return azimuth;
}
```

### 对不可实例化的类使用私有构造函数 <a name="classes-non-instantiable"></a>

只能由`Builder`创建的类、仅包含常量或静态方法的类或其他不可实例化的类应至少包含一个私有构造函数，以防止通过默认的无参数构造函数进行实例化。

```java {.good}
public final class Log {
  // Not instantiable.
  private Log() {}
}
```

### Singletons <a name="classes-singleton"></a>

`Singleton` 不鼓励使用，因为它们具有以下与测试相关的缺点：

1.  构造器被类管理，组织 `fakes` 对象的使用
1.  由于单例的静态性质，测试不能是密封的
1.  要解决这些问题，开发人员要么必须了解单例的内部细节，要么围绕它创建一个包装器

偏向于使用 [single instance]{#classes-single-instance} 模式，它依赖于抽象基类来解决这些问题。

### 单实例 {#classes-single-instance}

单实例类使用带有`private`或`internal`构造函数的抽象基类，并提供静态`getInstance()`方法来获取实例。 `getInstance()` 方法**必须**在后续调用中返回相同的对象。

`getInstance() `返回的对象应该是抽象基类的私有实现。

```kotlin {.bad}
class Singleton private constructor(...) {
  companion object {
    private val _instance: Singleton by lazy { Singleton(...) }

    fun getInstance(): Singleton {
      return _instance
    }
  }
}
```

```kotlin {.good}
abstract class SingleInstance private constructor(...) {
  companion object {
    private val _instance: SingleInstance by lazy { SingleInstanceImp(...) }
    fun getInstance(): SingleInstance {
      return _instance
    }
  }
}
```

单实例与 [singleton](#classes-singleton) 的不同之处在于，开发人员可以创建`SingleInstance`的假版本并使用自己的依赖注入框架来管理实现，而无需创建包装器，或者库可以提供自己的包装器 在`-testing`工件中伪造。

### 释放资源的类应该实现 `AutoCloseable` <a name="classes-autocloseable"></a>

通过`close`、`release`、`destroy`或类似方法释放资源的类应该实现`java.lang.AutoCloseable`，以允许开发人员在使用`try-with-resources`块时自动清理这些资源。

## Fields [F] <a name="fields"></a>

这些规则是关于类的公共字段的。

### 不要公开原始字段 <a name="fields-avoid-raw"></a>

Java 类不应直接公开字段。 字段应该是私有的，并且只能通过公共 `getter` 和 `setter` 访问，无论这些字段是最终的还是非最终的。

罕见的例外包括简单的数据结构，其中永远不需要增强指定或检索字段的功能。 在这种情况下，
这些字段应该使用标准变量命名约定来命名，例如。 `Point.x` 和 `Point.y`。

Kotlin 类可能会公开属性。

### 暴露的字段应标记为 `final` <a name="fields-mutable-bare"></a>

强烈建议不要使用原始字段 (@see[Do not expose raw fields](#fields-avoid-raw)).。 但在极少数情况下，某个字段作为公共字段公开，请将该字段标记为`final`。

### 内部字段不应暴露<a name="fields-internal"></a>

不要在公共 `API` 中引用内部字段名称。

```java {.bad}
public int mFlags;
```

### 使用 `public` 替代 `protected` <a name="fields-public"></a>

@see [Use public instead of protected](#avoid-protected)

## Constants [C] <a name="constants"></a>

这些是关于公共常量的规则。

### 标志常量应该是不重叠的`int`或`long`值<a name="constants-flags"></a>

“`Flags`” 意味着可以组合成某个联合值的位。 如果不是这种情况，请不要调用变量/常量`flag`。

```java {.bad}
public static final int FLAG_SOMETHING = 2;
public static final int FLAG_SOMETHING = 3;
```

```java {.good}
public static final int FLAG_PRIVATE = 1 << 2;
public static final int FLAG_PRESENTATION = 1 << 3;
```

有关定义公共标志常量的更多信息，See [`@IntDef` for `bitmask` flags](#annotations-intdef-bitmask)。

### `static final` 常量应使用全大写、下划线分隔的命名约定 <a name="constants-naming"></a>

常量中的所有单词都应大写，多个单词应使用`_`分隔。 例如：

```java {.bad}
public static final int fooThing = 5
```

```java {.good}
public static final int FOO_THING = 5
```

### 使用常量的标准前缀 <a name="constants-prefixes"></a>

`Android` 中使用的许多常量都是标准事物，例如`flags`,`keys`和`actions`。 这些常量应该有标准的前缀，以使它们更容易被识别。

例如，意图额外内容应以`EXTRA_`开头。 意图操作应以`ACTION_`开头。 与`Context.bindService()`一起使用的常量应以`BIND_`开头。

### 关键常量的命名和范围 <a name="constants-keys"></a>

字符串常量值应与常量名称本身一致，并且通常应限定在包或域内。 例如：

```java {.bad}
public static final String FOO_THING = “foo”
```

命名既不一致，也没有适当的范围。 相反，请考虑：

```java {.good}
public static final String FOO_THING = “android.fooservice.FOO_THING”
```

作用域字符串常量中的`android`前缀是为`AOSP`保留的。

`Intent actions`和`extras`以及`Bundle`应使用它们在其中定义的包名称进行命名空间。

```java {.good}
package android.foo.bar {
  public static final String ACTION_BAZ = “android.foo.bar.action.BAZ”
  public static final String EXTRA_BAZ = “android.foo.bar.extra.BAZ”
}
```

### 使用 `public` 而不是 `protected` <a name="constants-visibility"></a>

@see [Use public instead of protected](#avoid-protected)

### 使用一致的前缀 <a name="constants-consistency"></a>

相关常量应全部以相同的前缀开头。 例如，对于与标志值一起使用的一组常量：

```java {.bad}
public static final int SOME_VALUE = 0x01;

public static final int SOME_OTHER_VALUE = 0x10;

public static final int SOME_THIRD_VALUE = 0x100;
```

```java {.good}
public static final int FLAG_SOME_VALUE = 0x01;

public static final int FLAG_SOME_OTHER_VALUE = 0x10;

public static final int FLAG_SOME_THIRD_VALUE = 0x100;
```

@see [Use standard prefixes for constants](#constants-prefixes)

### 使用一致的资源名称 <a name="constants-resource-names"></a>

公共标识符、属性和值*必须*使用驼峰命名约定来命名，例如 `@id/accessibilityActionPageUp` 或 `@attr/textAppearance`，类似于 Java 中的公共字段。

在某些情况下，公共标识符或属性可能包含由下划线分隔的公共前缀：

*   平台配置值，例如`@string/config_recentsComponentName`
    [config.xml](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/res/res/values/config.xml;l=2721;drc=60faefffe98cb30abad690ccd183ef858bb7d3eb)
*   布局特定的视图属性，例如`@attr/layout_marginStart`
    [attrs.xml](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/res/res/values/attrs.xml;l=3446;drc=60faefffe98cb30abad690ccd183ef858bb7d3eb)

公共主题和样式*必须*遵循分层 `PascalCase` 命名约定，例如 `@style/Theme.Material.Light.DarkActionBar` 或`@style/Widget.Material.SearchView.ActionBar`，类似于`Java`中的嵌套类。

布局和可绘制资源*不应该*作为公共 `API` 公开。 但是，如果必须公开它们，则公共布局和可绘制对象*必须*使用下划线命名约定来命名，例如 `layout/simple_list_item_1.xml` 或`drawable/title_bar_tall.xml`。

### 当常量可能改变时，让它们变得动态 <a name="constants-min-max"></a>

常量值可以在编译时内联，因此保持值相同被视为 `API` 契约的一部分。 如果`MIN_FOO`或`MAX_FOO`常量的值将来可能会发生变化，请考虑将它们改为动态方法。

```java {.bad}
CameraManager.MAX_CAMERAS
```

```java {.good}
CameraManager.getMaxCameras()
```

### 考虑回调的前向兼容性 <a name="constants-callbacks-compatibility"></a>

面向旧版 `API` 的应用程序无法识别未来 `API` 版本中定义的常量。 因此，传递给应用程序的常量应考虑该应用程序的目标 `API` 版本，并将较新的常量映射到一致的值。 考虑以下场景：

假设的 `SDK` 来源:

```java
// Added in API level 22
public static final int STATUS_SUCCESS = 1;
public static final int STATUS_FAILURE = 2;
// Added in API level 23
public static final int STATUS_FAILURE_RETRY = 3;
// Added in API level 26
public static final int STATUS_FAILURE_ABORT = 4;
```

假设的应用程序具有 `targetSdkVersion="22"`:

```java
if (result == STATUS_FAILURE) {
  // Oh no!
} else {
  // Success!
}
```

在本例中，应用程序是在 `API` 级别 22 的约束下设计的，并做出了（某种程度上）合理的假设，即只有两种可能的状态。 但是，如果应用程序收到新添加的`STATUS_FAILURE_RETRY`，则它
会将其解释为成功。

返回常量的方法可以通过限制其输出以匹配应用程序目标的 `API` 级别来安全地处理此类情况：

```java {.good}
private int mapResultForTargetSdk(Context context, int result) {
  int targetSdkVersion = context.getApplicationInfo().targetSdkVersion;
  if (targetSdkVersion < 26) {
    if (result == STATUS_FAILURE_ABORT) {
      return STATUS_FAILURE;
    }
    if (targetSdkVersion < 23) {
      if (result == STATUS_FAILURE_RETRY) {
        return STATUS_FAILURE;
      }
    }
  }
  return result;
}
```

期望开发人员及其发布的应用程序具有洞察力是不合理的。 如果您使用`UNKNOWN`或`UNSPECIFIED`常量定义一个看起来像是包罗万象的 `API`，那么开发人员会认为他们在编写应用程序时发布的常量是详尽的。 如果你不愿意设置这个期望，重新考虑一个包罗万象的常量对于您的 `API` 是否是一个好主意。

还要考虑到库无法独立于应用程序指定自己的`targetSdkVersion`，并且处理库代码中的`targetSdkVersion`行为更改非常复杂且容易出错。

### 整数或字符串常量？ <a name="constants-integer-or-string"></a>

如果值的命名空间不可在包之外扩展，请使用整数常量和`@IntDef`。 如果命名空间是共享的或者可以通过包外部的代码扩展，请使用字符串常量。

### 数据类 <a name="data-class"></a>

数据类表示一组不可变的属性，并提供一组小型且定义良好的实用函数来与该数据进行交互。

**不要**在公共 `Kotlin API` 中使用`data class`，因为 `Kotlin` 编译器不保证生成代码的语言 `API` 或二进制兼容性。 相反，手动实现所需的功能。

#### 实例化 <a name="data-class-new"></a>

在 `Java` 中，当属性较少时，数据类应提供构造函数；当属性较多时，应使用 [`Builder`](#builders) 模式。

在 `Kotlin` 中，数据类应该提供带有默认参数的构造函数，无论属性的数量如何。 当面向 `Java` 客户端时，`Kotlin` 中定义的数据类也可能受益于提供构建器。

#### 修改和复制 <a name="data-class-copy"></a>

如果需要修改数据，请提供带有复制构造函数 (`Java`) 的 [`Builder`](#builders) 或 `copy()` 成员返回新对象的函数 (`Kotlin`)。

在 `Kotlin` 中提供 `copy()` 函数时，参数必须与类的构造函数匹配，并且必须使用对象的当前值填充默认值。

```kotlin {.good}
class Typography(
  val labelMedium: TextStyle = TypographyTokens.LabelMedium,
  val labelSmall: TextStyle = TypographyTokens.LabelSmall
) {
    fun copy(
      labelMedium: TextStyle = this.labelMedium,
      labelSmall: TextStyle = this.labelSmall
    ): Typography = Typography(
      labelMedium = labelMedium,
      labelSmall = labelSmall
    )
}
```

#### 附加功能 <a name="data-class-misc"></a>

数据类应该实现 [`equals()` 和 `hashCode()`](#equals-and-hashcode)，并且在这些方法的实现中必须考虑每个属性。

数据类可以使用与 [`Kotlin` 的数据类](https://kotlinlang.org/docs/data-classes.html) 实现相匹配的推荐格式来实现 [`toString()`](#toString)，例如 `User（var1 = Alex，var2 = 42）`。
