---
title: Android Developers 应用架构指南[网域层]
date: 2024-07-11 12:00:00
tags: [Android]
categories: [应用架构]
---
网域层是位于界面层和数据层之间的可选层。
网域层负责封装复杂的业务逻辑，或者由多个**ViewModel**重复使用的简单业务逻辑。此层是可选的，因为并非所有应用都有这类需求。因此，您应仅在需要时使用该层，例如处理复杂逻辑或支持可重用性。

![图 1. 网域层在应用架构中的作用。](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-domain-overview.png?hl=zh-cn)

> 注意 ：“网域层”一词在其他软件架构（例如“干净”架构）中使用，具有不同的含义。请勿将 Android 官方架构指南中定义的“网域层”的定义与您在其他地方看到的其他定义相混淆。二者可能存在细微但很重要的差异。

网域层具有以下优势：

- 避免代码重复。
- 改善使用网域层类的类的可读性。
- 改善应用的可测试性。
- 让您能够划分好职责，从而避免出现大型类。
- 为了使这些类保持简单轻量化，每个用例都应仅负责单个功能，且不应包含可变数据。您应在界面或数据层中处理可变的数据。

# 命名惯例
动词原形 + 名词/内容（可选）+ 用例。

例如：**LogOutUserUseCase**、**GetLatestNewsWithAuthorsUseCase**。

# 依赖项
在典型的应用架构中，用例类适合界面层的**ViewModel**与数据层的仓库。这意味着用例类通常依赖于仓库类，并且它们与界面层的通信方式和仓库与界面层的通信方式相同 - 使用回调（Java 代码）或协程（Kotlin 代码）。
例如，您的应用中可能有一个用例类，用于从新闻仓库和作者仓库中提取数据并对它们进行合并：
```kotlin
class FormatDateUseCase(userRepository: UserRepository) {

    private val formatter = SimpleDateFormat(
        userRepository.getPreferredDateFormat(),
        userRepository.getPreferredLocale()
    )

    operator fun invoke(date: Date): String {
        return formatter.format(date)
    }
}
```

在此示例中，**FormatDateUseCase**中的 invoke() 方法允许您像调用函数一样调用类的实例。invoke() 方法不限于任何特定签名，它可以接受任意数量的参数并返回任何类型。您还可以在类中使用不同的签名使 invoke() 重载。您可以按照如下方式调用上例中的用例：
```kotlin
class MyViewModel(formatDateUseCase: FormatDateUseCase) : ViewModel() {
    init {
        val today = Calendar.getInstance()
        val todaysDate = formatDateUseCase(today)
        /* ... */
    }
}
```

# 生命周期
用例没有自己的生命周期，而是受限于使用它们的类。这意味着，您可以从界面层中的类、服务或**Application**类本身调用用例。由于用例不应包含可变数据，因此您每次将用例类作为依赖项传递时，都应该创建一个新实例。

# 线程处理
来自网域层的用例必须是主线程安全的；换句话说，从主线程调用它们必须是安全的。如果用例类执行长期运行的阻塞操作，那么它们负责将该逻辑移至适当的线程。不过，在执行此操作之前，请检查这些阻塞操作是否最好放置在层次结构的其他层中。通常，数据层中会进行复杂的计算，以支持可重用性或缓存。例如，如果某项结果需要缓存起来，以便在应用的多个屏幕上重复使用，那么在数据层中对大列表执行资源密集型操作比在网域层中执行会更好。

以下示例显示了一个在后台线程上执行工作的用例：
```kotlin
class MyUseCase(
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {

    suspend operator fun invoke(...) = withContext(defaultDispatcher) {
        // Long-running blocking operations happen on a background thread.
    }
}
```

# 常见任务
## 可重复使用的简单业务逻辑
您应将界面层中存在的可重复业务逻辑封装到用例类中。这样您就可以更轻松地在使用该逻辑的所有位置应用任何更改，以及单独测试该逻辑。

以前面介绍的**FormatDateUseCase**为例。如果将来关于日期格式的业务要求发生变化，您只需在一个地方更改代码。
> 注意：在某些情况下，用例中可能存在的逻辑可以包含在 Util 类中的静态方法内。不过，不建议采用后一种方式，因为 Util 类通常很难找到，而且其功能也很难发现。此外，用例还可以共享通用功能（例如基类中的线程处理和错误处理），这对规模较大的大型团队很有助益。

## 合并仓库
在新闻应用中，您可能会使用分别用于处理新闻和作者数据操作的**NewsRepository**和**AuthorsRepository**类。**NewsRepository**提供的**Article**类仅包含作者的姓名，但您希望在界面上显示关于作者的更多信息。作者信息可通过**AuthorsRepository**获取。

![图 3. 用于合并多个仓库中所含数据的用例的依赖关系图。](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-domain-multiple-repos.png?hl=zh-cn)

由于该逻辑涉及多个仓库并且可能会变得很复杂，因此您可以创建**GetLatestNewsWithAuthorsUseCase**类，将逻辑从**ViewModel**中提取出来并提高其可读性。这也使得逻辑更易于单独测试，并且可在应用的不同部分重复使用。
```kotlin
/**
 * This use case fetches the latest news and the associated author.
 */
class GetLatestNewsWithAuthorsUseCase(
  private val newsRepository: NewsRepository,
  private val authorsRepository: AuthorsRepository,
  private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    suspend operator fun invoke(): List<ArticleWithAuthor> =
        withContext(defaultDispatcher) {
            val news = newsRepository.fetchLatestNews()
            val result: MutableList<ArticleWithAuthor> = mutableListOf()
            // This is not parallelized, the use case is linearly slow.
            for (article in news) {
                // The repository exposes suspend functions
                val author = authorsRepository.getAuthor(article.authorId)
                result.add(ArticleWithAuthor(article, author))
            }
            result
        }
}
```

该逻辑会映射**news**列表中的所有项；因此，即使数据层是主线程安全的，此工作也不应该阻塞主线程，因为您并不知道它会处理多少项。正因如此，该用例使用默认调度程序将工作移到后台线程。
> 注意：借助 Room 库，您可以查询数据库中不同实体之间的关系。如果数据库是可信来源，您可以创建一个查询，让系统为您执行所有工作。在这种情况下，最好创建仓库类（例如 NewsWithAuthorsRepository），而不是用例。

# 数据层访问权限限制
实现网域层时，还需要考虑应该仍然允许从界面层直接访问数据层，还是应该强制要求所有访问都必须通过网域层进行。

![图 4. 显示系统拒绝界面层访问数据层的依赖关系图](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-domain-data-access-restriction.png?hl=zh-cn)

设置此限制的好处之一是，这会阻止界面绕过网域层逻辑，例如，当您对针对数据层的每个访问请求执行分析日志记录时。

不过，潜在的重要缺陷在于，您不得不添加相关用例，即使只是对数据层进行简单的函数调用时也是如此，而这可能会增加复杂性，却几乎没有什么好处。

一种很好的方法是仅在需要时才添加用例。如果您发现界面层几乎完全通过用例访问数据，那么仅以这种方式访问数据可能是合理的。

最终，是否限制对数据层的访问权限取决于您的具体代码库，以及您倾向于采用更严格的规则还是更灵活的方法。


>网域层(Domain Layer)[https://developer.android.google.cn/topic/architecture/domain-layer?hl=zh-cn](https://developer.android.google.cn/topic/architecture/domain-layer?hl=zh-cn)