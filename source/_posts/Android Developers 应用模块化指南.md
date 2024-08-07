---
title: Android Developers 应用模块化指南
date: 2024-07-12 09:00:00
tags: [Android]
categories: [应用架构]
---
模块化是按多个松散耦合的独立部分整理代码库的做法。每个部分都是一个模块。每个模块都是独立的，并且都有明确的用途。通过将问题划分为更小、更易于解决的子问题，您可以降低设计和维护大型系统的复杂性。

![图 1：多模块代码库示例的依赖关系图](https://developer.android.google.cn/static/topic/modularization/images/1_sample_dep_graph.png?hl=zh-cn)

# 高内聚和低耦合原则
表征模块化代码库的一种方式是使用耦合和内聚属性。耦合用于衡量模块相互依赖的程度。在本指南中，内聚用于衡量单个模块的不同元素在功能上的相关性。一般而言，您应尽力实现低耦合和高内聚：

- **低耦合**是指模块应尽可能相互独立，这样一来，对一个模块所做的更改将对其他模块产生零影响或极小的影响。模块相互之间不应了解对方的内部运行原理。
- **高内聚**是指多个模块的代码集合应当像一个系统一样运行。它们应具有明确定义的职责，并始终位于特定领域知识范围以内。假设有一个电子书应用示例。在同一个模块中融合图书相关代码和付款相关代码是不合适的，因为图书和付款是两个不同的功能领域。

> 提示：如果两个模块相互严重依赖对方的信息，则可能表明这两个模块应当作为一个系统运行。相反，如果一个模块的两个部分并不经常相互交互，则这两个部分应当成为两个独立模块。

# 模块类型
您主要依赖于应用架构来组织模块。下面列出了您在遵循我们推荐的应用架构的同时，可以在应用中引入的一些通用模块类型。

## 数据模块
数据模块通常包含存储库、数据源和模型类。数据模块的三个主要职责包括：

1. 封装特定领域的所有数据和业务逻辑：每个数据模块都应负责处理表示特定领域的数据。它可以处理许多类型的数据，只要这些数据相关即可。
2. 将存储库公开为外部 API：数据模块的公共 API 应为存储库，因为它们负责向应用的其余部分公开数据。
3. 对外部隐藏所有实现细节和数据源：只能由同一模块中的存储库访问数据源。它们对外部始终处于隐藏状态。您可以使用 Kotlin 的**private**或**internal**可见性关键字来实现此操作。

![图 1. 示例数据模块及其内容。](https://developer.android.google.cn/static/topic/modularization/images/2_data_modules.png?hl=zh-cn)

## 功能模块
功能是应用功能的独立部分，通常对应于一个屏幕或一系列密切相关的屏幕，例如注册或结账流程。如果您的应用具有底部导航栏，则每个目标位置都可能是一项功能。
> 关键术语：“功能模块”也是 Play Feature Delivery 中使用的一个术语，用于描述可以按条件分发或按需下载的模块。不过，在本指南中，功能模块是指用于封装应用功能不同部分的模块。

![图 2. 此应用的每个标签页均可定义为功能。](https://developer.android.google.cn/static/topic/modularization/images/2_bottom_bar.png?hl=zh-cn)

功能与应用中的页面或目标位置相关联。因此，它们可能具有相关联的界面和**ViewModel**，用于处理其逻辑和状态。一项功能并不一定仅限于单一视图或导航目标位置。功能模块依赖于数据模块。

![图 3. 功能模块及其内容示例。](https://developer.android.google.cn/static/topic/modularization/images/2_feature_modules.png?hl=zh-cn)

## 应用模块
应用模块是应用的入口点。它们依赖于功能模块，并且通常提供根导航。由于支持 build 变体，因此单个应用模块可以编译为许多不同的二进制文件。

![图 4. *演示版*和*完整版*产品变种模块依赖关系图。](https://developer.android.google.cn/static/topic/modularization/images/2_demo_full_dep_graph.png?hl=zh-cn)

如果您的应用以多种设备类型（例如汽车、穿戴式设备或电视）为目标平台，请分别为每种设备类型定义一个应用模块。这有助于分离特定于平台的依赖项。

![图 5. Wear 应用依赖关系图。](https://developer.android.google.cn/static/topic/modularization/images/2_wear_dep_graph.png?hl=zh-cn)

## 通用模块
通用模块（也称为核心模块）包含其他模块经常使用的代码。它们可减少冗余，并且不代表应用架构中的任何特定层。下面列出了通用模块的一些示例：

- **界面模块**：如果您在应用中使用自定义界面元素或精心设计品牌元素，则应考虑将 widget 集合封装到一个模块中，以便重复使用所有功能。这有助于确保您的界面在不同功能之间保持一致。例如，如果您采用集中式主题，则在更换品牌名称时可以避免痛苦的重构过程。
- **分析模块**：跟踪通常取决于业务需求，而几乎不用考虑软件架构。分析跟踪器经常应用于许多不相关的组件。如果是这种情况，最好创建一个专用分析模块。
- **网络模块**：当许多模块需要网络连接时，您可以考虑创建一个专用于提供 http 客户端的模块。当客户端需要自定义配置，该模块尤为实用。
- **实用程序模块**：实用程序（也称为辅助程序）通常是在应用中重复使用的小段代码。实用程序的示例包括测试辅助程序、货币格式设置函数、电子邮件验证程序和自定义运算符。

## 测试模块
测试模块是仅用于测试用途的 Android 模块。 这些模块包含仅运行测试需要而应用运行时不需要的测试代码、测试资源和测试依赖项。测试模块可将测试专用代码与主应用分开，使模块代码更易于管理和维护。

- **共享测试代码**：如果您的项目中有多个模块，并且某些测试代码适用于多个模块，那么您可以创建一个测试模块来共享代码。这有助于减少代码重复，使测试代码更易于维护。共享测试代码可以包括实用程序类或函数（例如自定义断言或匹配器）以及测试数据（例如模拟 JSON 响应）。

- **更简洁的 build 配置**：测试模块可让您拥有更简洁的 build 配置，因为它们可以拥有自己的**build.gradle**文件。这样您就不必在应用模块的**build.gradle**文件中堆满仅用于测试的配置。

- **集成测试**：测试模块可用于存储集成测试，这些测试用于测试应用的不同部分（包括界面、业务逻辑、网络请求和数据库）之间的互动查询。

- **大型应用**：测试模块对于具有复杂代码库和多个模块的大型应用特别有用。在这种情况下，测试模块可以帮助改进代码的组织和可维护性。

![图 6. 测试模块可用于隔离原本相互依赖的模块。](https://developer.android.google.cn/static/topic/modularization/images/2_test_modules.png?hl=zh-cn)

# 模块间通信
很少有模块是完全隔离的。模块之间通常相互依赖并相互通信。即使多个模块协同运行并频繁交换信息，也务必要保持低耦合。有时，与架构约束一样，两个模块之间进行直接通信是不可取的方式。此外，在使用循环依赖项等情况下，两个模块直接进行通信也是不可行的。

![图 7. 使用循环依赖项时，在模块之间进行直接双向通信是不可行的。需要通过一个中介模块来协调两个其他独立模块之间的数据流。](https://developer.android.google.cn/static/topic/modularization/images/2_mediator.png?hl=zh-cn)

为了克服此问题，您可以在两个模块之间使用第三个模块作为中介。中间模块可以监听来自这两个模块的消息，并根据需要转发消息。在我们的示例应用中，即使事件源自属于不同功能的单独页面，结账页面也需要知道要购买哪本图书。在这种情况下，拥有导航图的模块将充当中间模块（通常是应用模块）。在此示例中，我们使用导航组件将数据从主屏幕功能传递至结账功能。
```kotlin
navController.navigate("checkout/$bookId")
```

结账目标会接收图书 ID 作为参数，用于获取图书的相关信息。您可以使用已保存的状态句柄来检索目标功能的**ViewModel**内的导航参数。
```kotlin
class CheckoutViewModel(savedStateHandle: SavedStateHandle, …) : ViewModel() {

   val uiState: StateFlow<CheckoutUiState> =
      savedStateHandle.getStateFlow<String>("bookId", "").map { bookId ->
          // produce UI state calling bookRepository.getBook(bookId)
      }
      …
}
```

您不应将对象作为导航参数来传递，而是应使用简单的 ID，以便功能使用 ID 从数据层访问和加载所需资源。这样一来，您就可以保持低耦合，并且不会违反单一信息源原则。

在以下示例中，两个功能模块均依赖于同一个数据模块。这样可以尽可能减少中间模块需要转发的数据量，并在模块之间保持低耦合。模块应传递基元 ID 并从共享数据模块加载资源，而不是传递对象。

![图 8. 两个功能模块依赖于一个共享数据模块。](https://developer.android.google.cn/static/topic/modularization/images/2_shared_data.png?hl=zh-cn)

# 依赖项反转
依赖项反转是指整理代码，使抽象与具体实现分离开来。

- **抽象**：定义应用中的组件或模块如何彼此互动的协定。抽象模块定义系统的 API，并包含接口和模型。
- **具体实现**：依赖于抽象模块并实现抽象行为的模块。

依赖于抽象模块中定义的行为的模块应仅依赖于抽象本身，而不是特定的实现。

![图 9. 高级别模块和实现模块依赖于抽象模块，而不是高级别模块直接依赖于低级别模块。](https://developer.android.google.cn/static/topic/modularization/images/2_di_concept.png?hl=zh-cn)

## 示例
假设有一个需要数据库才能正常运行的功能模块。该功能模块与数据库的实现方式无关，无论是本地 Room 数据库还是远程 Firestore 实例，都是如此。只需要存储和读取应用数据。

为了实现这一点，功能模块依赖于抽象模块，而不是特定的数据库实现。此抽象定义了应用的数据库 API。换言之，它规定了如何与数据库互动的规则。这样，功能模块便可以使用任何数据库，而无需了解其底层实现细节。

具体实现模块提供抽象模块中定义的 API 的实际实现。为此，实现模块还依赖于抽象模块。

## 依赖项注入
现在，您可能想知道功能模块如何与实现模块连接。答案是依赖项注入。功能模块不会直接创建所需的数据库实例，而是指定所需的依赖项。然后，从外部提供（通常位于应用模块中）这些依赖项。
```kotlin
releaseImplementation(project(":database:impl:firestore"))

debugImplementation(project(":database:impl:room"))

androidTestImplementation(project(":database:impl:mock"))
```
> 注意： 您可以为不同的 build 类型定义不同的依赖项。例如，发布 build 可以使用 Firestore 实现，调试 build 可以依赖于本地 Room 数据库，而插桩测试可以采用模拟实现。

## 益处
分离 API 及其实现会带来以下益处：

- **可互换性**：通过明确分离 API 和实现模块，您可为同一 API 开发多个实现，并在不更改使用该 API 的代码的情况下在这些实现之间进行切换。如果您想要在不同的上下文中提供不同的功能或行为，这种方法特别有用。例如，用于测试的模拟实现与用于生产的真实实现。
- **分离**：这种分离意味着使用抽象的模块不依赖于任何特定的技术。如果您之后选择将数据库从 Room 更改为 Firestore，更改会更加容易，因为更改只会在执行该作业的特定模块（实现模块）中发生，而不会影响使用数据库 API 的其他模块。
- **可测试性**：将 API 与其实现分离有助于更方便地进行测试。您可以根据 API 协定编写测试用例。您还可以使用不同的实现来测试各种场景和极端情况，包括模拟实现。
- **提高了构建性能**：当您将某个 API 及其实现拆分为不同的模块时，实现模块中的更改不会强制构建系统根据 API 模块重新编译这些模块。这样可以缩短构建时间并提高工作效率，尤其是对于构建时间可能非常长的大型项目。

## 何时分离
在以下情况下，将 API 与其实现分离将大有益处：

- **多样功能**：如果您可以通过多种方式实现系统的某些部分，一个明晰的 API 有助于促成不同实现之间的可互换性。例如，您可能有一个使用 OpenGL 或 Vulkan 的渲染系统，或者有一个可与 Play 或内部结算 API 配合使用的结算系统。
- **多个应用**：如果您要针对不同平台开发多个具有共享功能的应用，可以定义通用 API 并针对每个平台开发特定的实现。
- **独立团队**：这种分离可让不同的开发者或团队同时处理代码库的不同部分。开发者应专注于了解 API 协定并正确使用它们，而无需担心其他模块的实现细节。
- **大型代码库**：如果代码库很大或很复杂，将 API 与实现分离可使代码更易于管理，让您可以将代码库细分为更精细、更易于理解且更易于维护的单元。

## 如何实现？
如需实现依赖项反转，请按以下步骤操作：

1. **创建抽象模块**：此模块应包含定义功能的行为的 API（接口和模型）。
2. **创建实现模块**：实现模块应依赖于 API 模块，并实现抽象的行为。
    ![图 10. 实现模块依赖于抽象模块。](https://developer.android.google.cn/static/topic/modularization/images/2_api_impl.png?hl=zh-cn)
3. **使高级别模块依赖于抽象模块**：使模块依赖于抽象模块，而不是直接依赖于特定的实现。高级别模块不需要知道实现细节，它们只需要协定 (API)。
    ![图 11. 高级别模块依赖于抽象，而不是实现。](https://developer.android.google.cn/static/topic/modularization/images/2_api_impl_feature.png?hl=zh-cn)
4. **提供实现模块**：最后，您需要为依赖项提供实际实现。特定实现取决于您的项目设置，不过应用模块通常是一个不错的选择。如需提供实现，请将其指定为您选择的 build 变体或测试源代码集的依赖项。
    ![图 12. 应用模块提供实际实现。](https://developer.android.google.cn/static/topic/modularization/images/2_api_impl_app.png?hl=zh-cn)

# 最佳实践
正如开头所述，开发多模块应用并没有一种通用的正确方式。就像许多软件架构一样，也可以采用许多不同的方式来创建模块化应用。不过，以下一般性建议可帮助您提高代码的可读性、可维护性和可测试性。

## 保持配置一致
每个模块都会引入配置开销。如果模块数量达到特定阈值，保持一致的配置将成为一项挑战。例如，模块使用相同版本的依赖项非常重要。如果您需要更新大量模块来增加一个依赖项版本，这不仅是一项艰巨的任务，而且也很容易发生错误。如需解决此问题，您可以使用任一 Gradle 工具来集中管理配置：

- 版本目录是 Gradle 在同步期间生成的类型安全的依赖项列表。您可以在其中集中声明所有依赖项，并且可供项目中的所有模块使用。
- 使用惯例插件在模块之间共享 build 逻辑。

## 尽可能少公开信息
应尽量减少模块的公共接口，并且仅公开基本信息。不应向外部泄露任何实现细节。请尽可能缩小范围。使用 Kotlin 的 private 或 internal 可见性范围将模块声明设为私有模块。在模块中声明依赖项时，请优先使用 implementation 而不是 api。后者会向模块的使用方公开传递依赖项。使用实现可以缩短构建时间，因为这样可以减少需要重新构建的模块数量。

## 首选 Kotlin 和 Java 模块
Android Studio 支持以下三种基本类型的模块：

- **应用模块**是应用的入口点。它们可以包含源代码、资源、资产和 AndroidManifest.xml。应用模块的输出是 Android App Bundle (AAB) 或 Android 应用软件包 (APK)。
- **库模块**具有与应用模块相同的内容。其他 Android 模块使用库模块作为依赖项。库模块的输出是 Android ARchive (AAR)。库模块的输出在结构上与应用模块相同，但是会被编译为 Android ARchive (AAR) 文件，这些文件随后可以作为依赖项供其他模块使用。借助库模块，您可以在多个应用模块中封装和重复使用相同的逻辑和资源。
- **Kotlin 和 Java 库**不包含任何 Android 资源、资产或清单文件。

Android 模块会产生开销，因此您应当优先尽可能多地使用 Kotlin 或 Java 库。

>模块化(modularization)：[https://developer.android.google.cn/topic/modularization/patterns?hl=zh-cn](https://developer.android.google.cn/topic/modularization/patterns?hl=zh-cn)











