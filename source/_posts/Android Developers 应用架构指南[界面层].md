---
title: Android Developers 应用架构指南[界面层]
date: 2024-07-11 10:00:00
tags: [Android]
categories: [应用架构指南]
---
界面的作用是在屏幕上显示应用数据，并充当主要的用户互动点。每当数据发生变化时，无论是因为用户互动（例如按了某个按钮），还是因为外部输入（例如网络响应），界面都应随之更新，以反映这些变化。实际上，界面是从数据层获取的应用状态的直观呈现。

不过，从数据层获取的应用数据的格式通常不同于您需要显示的信息的格式。例如，您可能只需要在界面中显示部分数据，或者可能需要合并两个不同的数据源，以便提供切合用户需求的信息。无论您应用的是什么逻辑，都需要向界面传递完全呈现界面所需的所有信息。界面层是一个流水线，负责将应用数据变化转换为界面可以呈现的形式，然后将其显示出来。

![图 1. 界面层在应用架构中的作用](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-ui-overview.png?hl=zh-cn)

> 注意：本页中提供的建议和最佳实践可广泛应用于许多应用。遵循这些建议和最佳实践可以提升应用的可扩展性、质量和稳健性，并使应用更易于测试。不过，您应该将这些提示视为指南，并视需要进行调整来满足您的要求。

# 基本案例研究
让我们以一个可获取新闻报道供用户阅读的应用为例。该应用有一个报道屏幕，用于显示可供阅读的报道；另外，该应用允许已登录的用户为真正出众的报道添加书签。考虑到随时都可能有大量的报道，读者应能够按类别浏览报道。总的来说，该应用可让用户执行以下操作：

- 查看可供阅读的报道。
- 按类别浏览报道。
- 登录账号并为特定报道添加书签。
- 使用部分收费功能（如果符合相应条件）。

![图 2. 界面案例研究使用的示例“新闻”应用。](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-ui-basic-case-study.png?hl=zh-cn)

以下几个部分使用此示例作为案例研究，以便介绍单向数据流的原则，并展示在界面层的应用架构上下文中，这些原则有助于解决的问题。

# 界面层架构
“界面”这一术语是指用于显示数据的**Activity**和**Fragment**等界面元素，无论它们使用哪个 API（Views 还是 Jetpack Compose）来显示数据。由于数据层的作用是存储和管理应用数据，以及提供对应用数据的访问权限，因此界面层必须执行以下步骤：

1. 使用应用数据，并将其转换为界面可以轻松呈现的数据。
2. 使用界面可呈现的数据，并将其转换为用于向用户呈现的界面元素。
3. 使用来自这些组合在一起的界面元素的用户输入事件，并根据需要反映它们对界面数据的影响。
4. 根据需要重复第 1-3 步。

本指南的其余部分展示了如何实现用于执行这些步骤的界面层。具体来说，本指南涵盖以下任务和概念：

- 如何定义界面状态。
- 单向数据流 (UDF)，作为提供和管理界面状态的方式。
- 如何根据 UDF 原则使用可观察数据类型公开界面状态。
- 如何实现使用可观察界面状态的界面。

其中最基本的便是定义界面状态。

# 定义界面状态
请参阅上文所述的案例研究。简言之，界面会显示一个报道列表，以及每篇报道的部分元数据。该应用向用户显示的这些信息便是界面状态。

换言之，如果界面是相对用户而言的，那么界面状态就是相对应用而言的。这就像同一枚硬币的两面，界面是界面状态的直观呈现。对界面状态所做的任何更改都会立即反映在界面中。

![图 3. 界面是将屏幕上的界面元素与界面状态绑定在一起的结果。](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-ui-elements-state.png?hl=zh-cn)

以案例研究为例，为了满足“新闻”应用的要求，可以将完全呈现界面所需的信息封装在如下定义的**NewsUiState**数据类中
```kotlin
data class NewsUiState(
    val isSignedIn: Boolean = false,
    val isPremium: Boolean = false,
    val newsItems: List<NewsItemUiState> = listOf(),
    val userMessages: List<Message> = listOf()
)

data class NewsItemUiState(
    val title: String,
    val body: String,
    val bookmarked: Boolean = false,
    ...
)
```

## 不可变性
以上示例中的界面状态定义是不可变的。这样的主要好处是，不可变对象可保证即时提供应用的状态。这样一来，界面便可专注于发挥单一作用：读取状态并相应地更新其界面元素。因此，切勿直接在界面中修改界面状态，除非界面本身是其数据的唯一来源。违反这个原则会导致同一条信息有多个可信来源，从而导致数据不一致和轻微的 bug。

例如，如果案例研究中来自界面状态的**NewsItemUiState**对象中的 bookmarked 标记在**Activity**类中已更新，那么该标记会与数据层展开竞争，以争取成为报道的“已添加书签”状态的来源。不可变数据类对于防止此类反模式非常有用。
> 要点：只有数据源或数据所有者才应负责更新其公开的数据。

## 本指南中的命名惯例
在本指南中，界面状态类是根据其描述的屏幕或部分屏幕的功能命名的。具体命名惯例如下：

功能 + UiState。

例如，用于显示新闻的屏幕的状态可以称为**NewsUiState**，新闻报道列表中的新闻报道的状态可以为**NewsItemUiState**。

# 使用单向数据流管理状态
上一部分中指出，界面状态是呈现界面所需的详细信息的不可变快照。不过，应用中数据的动态特性意味着状态可能会随时间而变化。这可能是因为用户互动，也可能是因为其他事件修改了用于填充应用的底层数据。

这些互动可以受益于处理它们的 mediator，从而定义要为每个事件应用的逻辑，并对后备数据源执行必要的转换，以便创建界面状态。这些互动及其逻辑可以位于界面本身中，但随着界面开始担任其名称所表明的角色以外的角色（数据所有者、提供方、转换器等），这可能很快就会变得难以掌控。此外，这可能会影响可测试性，因为生成的代码是紧密耦合的代码，没有可辨别的边界。归根结底，界面能够受益于减轻的负担。除非界面状态非常简单，否则界面的唯一职责应该是使用和显示界面状态。

本部分介绍了单向数据流 (UDF)，这是一种架构模式，有助于强制实施这种健康的职责分离。

## 状态容器
符合以下条件的类称为状态容器：负责提供界面状态，并且包含执行相应任务所必需的逻辑。状态容器有多种大小，具体取决于所管理的界面元素的作用域（从底部应用栏等单个微件，到整个屏幕或导航目的地，不一而足）。

在后一种情况下，典型的实现是**ViewModel**的实例，不过根据应用的要求，使用简单的类可能就足够了。例如，案例研究中的“新闻”应用使用**NewsViewModel**类作为状态容器，以便为该部分显示的屏幕画面提供界面状态。
> 要点：ViewModel 类型是推荐的实现，用于管理屏幕级界面状态，具有数据层访问权限。此外，它会在配置发生变化后自动继续存在。ViewModel 类用于定义要为应用中的事件应用的逻辑，并提供更新后的状态作为结果。

您可以通过多种方式为界面与其状态提供方之间的互相依赖关系建模。不过，由于界面与其**ViewModel**类之间的互动在很大程度上可以理解为事件输入及其随后的状态输出，因此这种关系可以按下图所示来表示：

![图 4. UDF 在应用架构中的运作方式图示。](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-ui-udf.png?hl=zh-cn)

状态向下流动、事件向上流动的这种模式称为单向数据流 (UDF)。这种模式对应用架构的影响如下：

- ViewModel 会存储并公开界面要使用的状态。界面状态是经过 ViewModel 转换的应用数据。
- 界面会向 ViewModel 发送用户事件通知。
- ViewModel 会处理用户操作并更新状态。
- 更新后的状态将反馈给界面以进行呈现。
- 系统会对导致状态更改的所有事件重复上述操作。

对于导航目的地或屏幕，**ViewModel**会使用存储库或用例类来获取数据并将其转换为界面状态，同时纳入可能会导致状态更改的事件的影响。前面提到的案例研究包含一个报道列表，其中每篇报道都有标题、说明、来源、作者名称、发布日期，以及是否添加了书签。每篇报道的界面如下所示：

![图 5. 案例研究应用中的报道界面。](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-ui-basic-case-study-item.png?hl=zh-cn)

用户请求为报道添加书签就是一个可能会导致状态更改的事件示例。作为状态提供方，**ViewModel**的职责是定义所有必需的逻辑，以便填充界面状态中的所有字段，并处理界面完全呈现所需的事件。

为何使用 UDF UDF 可为状态提供周期建模（如图 4 所示）。它还可以将以下位置分离开来：状态变化来源位置、转换位置以及最终使用位置。这种分离可让界面只发挥其名称所表明的作用：通过观察状态变化来显示信息，并通过将这些变化传递给**ViewModel**来传递用户 intent。

换句话说，UDF 有助于实现以下几点：

- **数据一致性**。界面只有一个可信来源。
- **可测试性**。状态来源是独立的，因此可独立于界面进行测试。
- **可维护性**。状态的更改遵循明确定义的模式，即状态更改是用户事件及其数据拉取来源共同作用的结果。

# 公开界面状态
定义界面状态并确定如何管理相应状态的提供后，下一步是将提供的状态发送给界面。由于您使用 UDF 管理状态的提供，因此您可以将提供的状态视为数据流，换句话说，随着时间的推移，将提供状态的多个版本。因此，您应在**LiveData**或**StateFlow**等可观察数据容器中公开界面状态。这样做是为了使界面可以对状态的任何变化做出反应，而无需直接从**ViewModel**手动拉取数据。这些类型还有一个好处是，始终缓存界面状态的最新版本，这对于在配置发生变化后快速恢复状态非常有用。
```kotlin
class NewsViewModel(...) : ViewModel() {

    val uiState: StateFlow<NewsUiState> = …
}
```

如果向界面公开的数据相当简单，通常值得将数据封装在界面状态类型中，因为它能传达状态容器的发出与其关联的屏幕或界面元素之间的关系。此外，随着界面元素变得越来越复杂，添加界面状态的定义来容纳呈现界面元素所需的额外信息始终会更加容易。

创建**UiState**流的一种常用方法是，将后备可变数据流作为来自**ViewModel**的不可变数据流进行公开，例如将**MutableStateFlow<UiState>**作为**StateFlow<UiState>**进行公开。
```kotlin
class NewsViewModel(...) : ViewModel() {

    private val _uiState = MutableStateFlow(NewsUiState())
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    ...

}
```

这样一来，**ViewModel**便可以公开在内部更改状态的方法，以便发布供界面使用的更新。以需要执行异步操作的情况为例，可以使用**viewModelScope**启动协程，并且可以在操作完成时更新可变状态。
```kotlin
class NewsViewModel(
    private val repository: NewsRepository,
    ...
) : ViewModel() {

    private val _uiState = MutableStateFlow(NewsUiState())
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    private var fetchJob: Job? = null

    fun fetchArticles(category: String) {
        fetchJob?.cancel()
        fetchJob = viewModelScope.launch {
            try {
                val newsItems = repository.newsItemsForCategory(category)
                _uiState.update {
                    it.copy(newsItems = newsItems)
                }
            } catch (ioe: IOException) {
                // Handle the error and notify the UI when appropriate.
                _uiState.update {
                    val messages = getMessagesFromThrowable(ioe)
                    it.copy(userMessages = messages)
                 }
            }
        }
    }
}
```

在上面的示例中，**NewsViewModel**类会尝试获取特定类别的报道，然后在界面状态中反映尝试结果（成功或失败），其中界面可以对其做出适当反应。
> 注意：单向数据流有多种常用的实现，上面示例中显示的模式（通过 ViewModel 上的函数更改状态）便是其中的一种。

除了前面的指南之外，公开界面状态时还要考虑以下事项：
- **界面状态对象应处理彼此相关的状态**。 这样可以减少不一致的情况，并让代码更易于理解。如果您在两个不同的数据流中分别公开新闻报道列表和书签数量，可能会发现其中一个已更新，但另一个没有更新。当您使用单个数据流时，这两个元素都会保持最新状态。此外，某些业务逻辑可能需要组合使用数据源。例如，可能只有在用户已登录并且是付费新闻服务订阅者时，您才需要显示书签按钮。您可以按如下方式定义界面状态类：
    ```kotlin
    data class NewsUiState(
        val isSignedIn: Boolean = false,
        val isPremium: Boolean = false,
        val newsItems: List<NewsItemUiState> = listOf()
    )
    
    val NewsUiState.canBookmarkNews: Boolean get() = isSignedIn && isPremium
    ```
    在此声明中，书签按钮的可见性是两个其他属性的派生属性。随着业务逻辑变得越来越复杂，拥有单个**UiState**类，并且其中的所有属性都是立即可用的，变得越来越重要。

- **界面状态**：单个数据流还是多个数据流？是选择在单个数据流中还是在多个数据流中公开界面状态，关键指导原则是前面提到的要点：发出的内容之间的关系。在单个数据流中进行公开的最大优势是便捷性和数据一致性：状态的使用方随时都能立即获取最新信息。不过，在有些情况下，可能适合使用来自**ViewModel**的单独的状态流：
    - **不相关的数据类型**：呈现界面所需的某些状态可能是完全相互独立的。在此类情况下，将这些不同的状态捆绑在一起的代价可能会超过其优势，尤其是当其中某个状态的更新频率高于其他状态的更新频率时。
    - **UiState diffing**：**UiState**对象中的字段越多，数据流就越有可能因为其中一个字段被更新而发出。由于视图没有 diffing 机制来了解连续发出的数据流是否相同，因此每次发出都会导致视图更新。这意味着，可能必须要对**LiveData**使用 Flow API 或**distinctUntilChanged()**等方法来缓解这个问题。

# 使用界面状态
如需在界面中使用**UiState**对象流，您可以对所使用的可观察数据类型使用终端运算符。例如，对于**LiveData**，您可以使用**observe()**方法；对于**Kotlin**数据流，您可以使用**collect()**方法或其变体。

在界面中使用可观察数据容器时，请务必考虑界面的生命周期。这非常重要，因为当未向用户显示视图时，界面不应观察界面状态。使用**LiveData**时，**LifecycleOwner**会隐式处理生命周期问题。使用数据流时，最好通过适当的协程作用域和**repeatOnLifecycle**API 来处理这一任务：
```kotlin
class NewsActivity : AppCompatActivity() {

    private val viewModel: NewsViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        ...

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect {
                    // Update UI elements
                }
            }
        }
    }
}
```

> 注意：在没有处于活跃状态的收集器时，本示例中使用的具体 StateFlow 对象不会停止执行工作，但您在处理数据流时，可能并不知道它们是如何实现的。借助生命周期感知型数据流收集功能，您可以在以后对 ViewModel 数据流进行这些类型的更改，而无需重新访问下游收集器代码。

## 显示正在执行的操作
在**UiState**类中表示加载状态的一种简单方法是使用布尔值字段：
```kotlin
data class NewsUiState(
    val isFetchingArticles: Boolean = false,
    ...
)
```

此标记的值表示界面中是否存在进度条。
```kotlin
class NewsActivity : AppCompatActivity() {

    private val viewModel: NewsViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        ...

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Bind the visibility of the progressBar to the state
                // of isFetchingArticles.
                viewModel.uiState
                    .map { it.isFetchingArticles }
                    .distinctUntilChanged()
                    .collect { progressBar.isVisible = it }
            }
        }
    }
}
```

## 在屏幕上显示错误
在界面中显示错误与显示正在执行的操作类似，因为无论是错误，还是正在执行的操作，都能通过用于表明它们是否存在的布尔值来轻松表示。不过，错误可能还包括要传回给用户的关联消息，或包含与其关联的操作（旨在重试失败的操作）。因此，无论正在执行的操作是否正在加载，可能都需要使用托管以下数据的数据类对错误状态进行建模：适合错误上下文的元数据。

以上一部分中的示例为例，它在获取报道时会显示进度条。如果此操作导致错误，您可能希望向用户显示一条或多条消息，详细说明出现了什么错误。
```kotlin
data class Message(val id: Long, val message: String)

data class NewsUiState(
    val userMessages: List<Message> = listOf(),
    ...
)
```

然后，错误消息便能够以界面元素（例如信息提示控件）的形式呈现给用户。

# 线程处理和并发
在**ViewModel**中执行的所有工作都应具有主线程安全性（即从主线程调用是安全的）。这是因为数据层和网域层负责将工作移至其他线程。

如果**ViewModel**执行长时间运行的操作，则还要负责将相应逻辑移至后台线程。**Kotlin**协程是管理并发操作的绝佳方式，**Jetpack**架构组件则为其提供内置支持。

# 导航
应用导航的变化通常是由类似于事件的发出操作驱动的。例如，在**SignInViewModel**类执行登录后，**UiState**可能会有一个**isSignedIn**字段被设为**true**。此类触发器的使用方式应与上面使用界面状态部分介绍的方式相同，不过使用实现应遵从导航组件。

# Paging
Paging 库通过一个称为**PagingData**的类型在界面中使用。由于**PagingData**表示并包含可以随时间变化的内容（换句话说，它不是不可变类型），因此它不应以不可变界面状态表示。相反，您应在单独的流中独立地从**ViewModel**中公开它。

# 动画
为了提供流畅的顶级导航过渡，您可能需要等待第二个屏幕加载数据，然后再启动动画。Android 视图框架提供了一些钩子，以便通过**postponeEnterTransition**和**startPostponedEnterTransition** API 延迟 fragment 目的地之间的过渡。这些 API 提供了一种方法来确保做到以下一点：在界面通过动画过渡到第二个屏幕之前，第二个屏幕上的界面元素（通常是从网络获取的图片）已做好显示准备。


