## `Android`平台`API`的演变<a name="evolution"></a>

关于可以对现有 `Android` `API` 进行哪些类型的更改以及如何实施这些更改以最大限度地提高与现有应用程序和代码库的兼容性的政策。

### 破坏二进制的更改 <a name="binary-compat"></a>

最终确定的公共 `API` 中不允许进行二进制破坏性更改，并且在运行`make update-api`时通常会引发错误。然而，可能存在 `Metalava` 的 `API` 检查无法捕获的边缘情况。 如有疑问，请参阅 `Eclipse Foundation` 的 [`Evolving Java-based APIs`](https://wiki.eclipse.org/Evolving_Java-based_APIs_2) 指南，了解有关哪些类型的 `API` 更改兼容的详细说明。非公共（例如系统）`API` 中的二进制破坏更改应遵循 [弃用/替换](#deprecation) 周期。

### 破坏源代码的更改 <a name="source-compat"></a>

即使不破坏二进制文件，也不鼓励破坏源代码的更改。二进制兼容但破坏源代码的更改的一个示例是向现有类添加泛型，这是[二进制兼容](https://wiki.eclipse.org/Evolving_Java-based_APIs_2#Turning_non-generic_types_and_methods_into_generic_ones)，但可能由于继承或不明确的引用而引入编译错误。
运行 `make update-api` 时，破坏源代码的更改**不会**引发错误，因此必须注意了解现有 `API` 签名更改的影响。

在某些情况下，为了改善开发人员体验或代码正确性，需要进行破坏性的更改。 例如，向 `Java` 源代码添加可空性注释可提高与 `Kotlin` 代码的互操作性并降低出错的可能性，但通常需要对源代码进行更改（有时是重大更改）。

### 对私有 `API` 的更改 (`@SystemApi`, `@TestApi`) <a name="private-apis"></a>

用`@TestApi`注释的 `API` 可能随时更改。

使用`@SystemApi`注释的 `API` 必须保存三年。 系统 `API` 的删除或重构必须按以下时间表进行：

*   `API` y - 添加
*   `API` y+1 - [`Deprecation`](#deprecation)废弃
    *   将代码标记为`@Deprecated`
    *   添加替换，并使用 `@deprecated` 标记链接到 `javadoc` 中已弃用代码的替换。
    *   在开发周期中期，针对内部用户提交错误，告诉他们 `API` 即将消失，让他们有机会确保足够替换 `API` 。
*   `API` y+2 - [`Soft removal`](#soft-removal)软删除
    *   将代码标记为`@removed`
    *   （可选）对于针对当前版本的 `sdk` 级别的应用程序，抛出或无操作
*   `API` y+3 - [`Hard removal`](#hard-removal)硬删除
    *   代码已从源代码树中完全删除

### `Deprecation` <a name="deprecation"></a>

弃用被视为 `API` 更改，可能会在主要版本中发生。 弃用 `API` 时，请同时使用`@Deprecated`源注释和`@deprecated <summary>`文档注释。 您的摘要**必须**包含迁移策略，该策略可能链接到替换 `API` 或解释不应使用该 `API` 的原因。

```java {.good}
/**
 * Simple version of ...
 *
 * @deprecated Use the {@link androidx.fragment.app.DialogFragment}
 *             class with {@link androidx.fragment.app.FragmentManager}
 *             instead.
 */
@Deprecated
public final void showDialog(int id)
```

在 `XML` 中定义并在 `Java` 中公开的 `API`，包括在`android.R`类中公开的属性和可样式属性，**也必须**通过摘要弃用。

```xml {.good}
<!-- Attribute whether the accessibility service ...
     {@deprecated Not used by the framework}
 -->
<attr name="canRequestEnhancedWebAccessibility" format="boolean" />
```

#### 什么时候适合弃用 `API`？ <a name="deprecation-appropriate"></a>

弃用对于阻止在*新代码*中采用该 `API` 最为有用。

我们还要求 `API` 在 [`@removed`](#soft-removal) 之前被标记为 `@deprecated`，但这并没有为开发人员迁移他们已经使用的 `API` 提供强烈的动力。

在弃用 `API` 之前，请考虑对开发人员的影响。 弃用 `API` 的影响包括：

-   `javac` 将在编译期间发出警告
    - 弃用警告无法全局抑制或基线化，因此使用`-Werror`的开发人员需要单独修复或抑制*每个*弃用 `API` 的使用，然后才能更新其编译 `SDK` 版本
    - 无法抑制导入已弃用类的弃用警告，因此开发人员需要为已弃用类的*每个*使用内联完全限定的类名，然后才能更新其编译 `SDK` 版本
-   `d.android.com` 上的文档将显示弃用通知
-   像 `Android` `Studio` 这样的 `IDE` 会在 `API` 使用站点显示警告
-   `IDE `*可能*降低 `API` 的排名或隐藏其自动完成功能

因此，弃用 `API` 可能会阻止最关心代码健康的开发人员（使用`-Werror`的开发人员）采用新的 `SDK`。
另一方面，不关心现有代码中警告的开发人员可能会完全忽略弃用。

当 `SDK` 引入大量弃用内容时，这两种情况都会变得更糟。

因此，我们建议仅在以下情况下弃用 `API`：

1.  `API` 在未来版本中将是`@removed`;
1.  使用 `API` 会导致不正确或未定义的行为，无法在不破坏兼容性的情况下修复这些行为;

当 `API` 被弃用并被新 `API` 取代时，我们*强烈建议*将相应的兼容性 `API` 添加到 `Jetpack` 库（如`androidx.core`）中，以简化对新旧设备的支持。

我们*不*建议弃用按预期工作并将在未来版本中继续工作的 `API`。

```java {.bad}
/**
 * ...
 * @deprecated Use {@link #doThing(int, Bundle)} instead.
 */
@Deprecated
public void doThing(int action) {
  ...
}

public void doThing(int action, @Nullable Bundle extras) {
  ...
}
```

```java {.good}
/**
 * ...
 * @deprecated No longer displayed in the status bar as of API 21.
 */
@Deprecated
public RemoteViews tickerView;
```

#### 对已弃用 `API` 的更改 <a name="deprecation-changes"></a>

**必须**维护已弃用 `API` 的行为，这意味着测试实现必须保持不变，并且在 `API` 被弃用后测试必须继续通过。 如果`API`没有测试，则应添加测试。

已弃用的 `API` 表面 **不应** 在未来版本中进行扩展。 `Lint` 正确性注释（例如`@Nullable`）可以添加到现有的已弃用的 `API` 中，但不应将新的 `API` 添加到已弃用的类或接口中。

新的 `API` **不应** 添加为已弃用的 `API`。 在预发布周期内添加并随后弃用的 `API`（因此最初会作为弃用的形式进入公共 `API` 表面）应在 `API` 最终确定之前删除。

### 软删除 <a name="soft-removal"></a>

软删除是一种破坏源代码的更改，除非得到 `API` 委员会的明确批准，否则应在公共 `API` 中避免使用。 如需批准，请发送电子邮件至 [`Android API Council` 邮件列表](mailto:android-api-council@google.com)。 对于系统 `API`，软删除之前必须在主要版本期间进行弃用。 删除对 `API` 的所有文档引用，并在软删除 `API` 时使用`@removed <summary>`文档注释。 您的摘要**必须**包括删除的原因，并且可能包括弃用中所述的迁移策略。

软删除 `API` 的行为**可以**保持原样，但更重要的是**必须**保留，以便现有调用者在调用 `API` 时不会崩溃。 在某些情况下，这可能意味着保留行为。

**必须**维持测试覆盖率，但测试的内容可能需要更改以适应行为变化。 测试仍然必须确保现有的调用者不会在运行时崩溃。

```java {.good}
/**
 * Ringer volume. This is ...
 *
 * @removed Not functional since API 2.
 */
public static final String VOLUME_RING = ...
```

在技术层面上，该 `API` 已从 `SDK` 存根 `JAR` 和编译时类路径中删除，但仍然存在于运行时类路径中——类似于`@hide` `API`。

从应用程序开发人员的角度来看，当`compileSdk`等于或晚于删除 `API` 的 `SDK` 时，`API` 不再出现在自动完成中，并且引用该 `API` 的源代码将不再编译； 但是，源代码将继续针对早期 `SDK` 成功编译，并且引用 `API` 的二进制文件将继续工作。

某些类别的 `API` **不得** 被软删除。

#### 抽象方法

开发人员可以扩展的类的抽象方法**不能**被软删除。 这样做将使开发人员无法在所有 `SDK` 级别成功扩展该类。

在极少数情况下，开发人员*从未*且*永远不可能*扩展类，抽象方法仍可能被软删除。

### 硬删除 <a name="hard-removal"></a>

硬删除是一种二进制破坏性更改，**绝不应在公共 `API` 中发生**。
对于系统 `API`，在主要版本期间，硬删除**必须**先于软删除。 硬删除 `API` 时删除整个实现。

对硬删除 `API` 的测试**必须**删除，因为它们将不再编译。

### Discouraging {.numbered}

`@Discouraged` 注释用于指示在大多数 (>95%) 情况下不推荐使用该 `API`。 `Discouraged` 的 `API` 与 `deprecated` 的 `API` 的不同之处在于：存在一个狭窄的关键用例，可以防止 `deprecation`。 将 `API` 标记为`@Discouraged` 时，必须提供解释和替代解决方案。

```java {.good}
@Discouraged(message = "Use of this function is discouraged because resource
                        reflection makes it harder to perform build
                        optimizations and compile-time verification of code. It
                        is much more efficient to retrieve resources by
                        identifier (e.g. `R.foo.bar`) than by name (e.g.
                        `getIdentifier()`)")
public int getIdentifier(String name, String defType, String defPackage) {
    return mResourcesImpl.getIdentifier(name, defType, defPackage);
}
```

新的 `API`**不应**被添加为`@Discouraged`。目前，只有性能团队可以添加 `API` 的 `@Discouraged` 注释。

### 改变现有 `API` 的行为 <a name="behavior"></a>

在某些情况下，可能需要更改现有 `API` 的实现行为。 例如，在 Android 7.0 中，我们改进`DropBoxManager`，以便在开发人员尝试发布太大而无法通过`Binder`发送的事件时清晰地进行通信。

但是，为了确保现有应用程序不会对这些行为更改感到惊讶，我们强烈建议为旧应用程序保留安全行为。 我们过去一直根据应用程序的`ApplicationInfo.targetSdkVersion`来保护这些行为更改，但我们最近迁移为要求使用应用程序兼容性框架。 以下是如何使用这个新框架实现行为改变的示例：

```java {.good}
import android.app.compat.CompatChanges;
import android.compat.annotation.ChangeId;
import android.compat.annotation.EnabledSince;

public class MyClass {
  @ChangeId
  // This means the change will be enabled for target SDK R and higher.
  @EnabledSince(targetSdkVersion=android.os.Build.VERSION_CODES.R)
  // Use a bug number as the value, provide extra detail in the bug.
  // FOO_NOW_DOES_X will be the change name, and 123456789 the change id.
  static final long FOO_NOW_DOES_X = 123456789L;

  public void doFoo() {
    if (CompatChanges.isChangeEnabled(FOO_NOW_DOES_X)) {
      // do the new thing
    } else {
      // do the old thing
    }
  }
}
```

使用此应用程序兼容性框架设计，开发人员可以在预览版和测试版发布期间暂时禁用特定的行为更改，作为调试应用程序的一部分，而不是强迫他们同时调整数十个行为更改。

#### 向前兼容性

前向兼容性是一种设计特征，允许系统接受针对其自身更高版本的输入。 在 `API` 设计（尤其是平台 `API`）的情况下，必须特别注意初始设计以及未来的更改，因为开发人员希望编写一次代码，测试一次，然后让它在任何地方运行而不会出现问题。

`Android` 中最常见的向前兼容性问题是由以下原因引起的：

-   将新常量添加到先前假定完整的集合（例如`@IntDef`或`enum`）中，例如 其中`switch`有一个抛出异常的`default`
-   添加对 `API` 表面未直接捕获的功能的支持，例如 支持在 XML 中分配`ColorStateList`类型资源，以前仅支持`<color>`资源
-   放宽对运行时检查的限制，例如 删除旧版本上存在的`requireNotNull()`检查

在所有这些情况下，开发人员只会在运行时发现问题。 更糟糕的是，他们可能只能根据现场旧设备的崩溃报告才能找到答案。

此外，这些案例都是*技术上*有效的 `API` 更改。 它们不会破坏二进制或源代码兼容性，并且 `API` `lint` 不会捕获任何这些问题。

因此，`API` 设计者在修改现有类时必须格外小心。 提出这样的问题：“此更改是否会导致*仅*针对最新版本的平台编写和测试的代码在旧版本上失败？”
