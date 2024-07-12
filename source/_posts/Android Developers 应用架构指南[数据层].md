---
title: Android Developers 应用架构指南[数据层]
date: 2024-07-11 22:00:00
tags: [Android]
categories: [应用架构]
---
界面层包含与界面相关的状态和界面逻辑，数据层则包含应用数据和业务逻辑。业务逻辑决定应用的价值，它由现实世界的业务规则组成，这些规则决定着应用数据的创建、存储和更改方式。

这种关注点分离使得数据层可用于多个屏幕、在应用的不同部分之间共享信息，以及在界面以外复制业务逻辑以进行单元测试。

# 架构
数据层由多个仓库组成，其中每个仓库都可以包含零到多个数据源。您应该为应用中处理的每种不同类型的数据分别创建一个存储库类。例如，您可以为与电影相关的数据创建一个**MoviesRepository**类，或者为与付款相关的数据创建一个**PaymentsRepository**类。

![图 1. 界面层在应用架构中的作用。](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-data-overview.png?hl=zh-cn)

存储库类负责以下任务：

- 向应用的其余部分公开数据。
- 集中处理数据变化。
- 解决多个数据源之间的冲突。
- 对应用其余部分的数据源进行抽象化处理。
- 包含业务逻辑。

每个数据源类应仅负责处理一个数据源，数据源可以是文件、网络来源或本地数据库。数据源类是应用与数据操作系统之间的桥梁。

层次结构中的其他层绝不能直接访问数据源；数据层的入口点始终是存储库类。状态容器类或用例类绝不能将数据源作为直接依赖项。如果使用仓库类作为入口点，架构的不同层便可以独立扩缩。

**该层公开的数据应该是不可变的**，这样就可以避免数据被其他类篡改，从而避免数值不一致的风险。不可变数据也可以由多个线程安全地处理。

按照依赖项注入方面的最佳实践，存储库应在其构造函数中将数据源作为依赖项：
```kotlin
class ExampleRepository(
    private val exampleRemoteDataSource: ExampleRemoteDataSource, // network
    private val exampleLocalDataSource: ExampleLocalDataSource // database
) { /* ... */ }
```

> 注意： 通常，当代码库仅包含单个数据源且不依赖于其他代码库时，开发者会将代码库和数据源的职责合并到代码库类中。如果您这样做，在应用的更高版本中，如果仓库需要处理来自其他来源的数据，请不要忘记拆分功能。

# 公开 API
数据层中的类通常会公开函数，以执行一次性的创建、读取、更新和删除 (CRUD) 调用，或接收关于数据随时间变化的通知。对于每种情况，数据层都应公开以下内容：

- **一次性操作**：在 Kotlin 中，数据层应公开挂起函数；对于 Java 编程语言，数据层应公开用于提供回调来通知操作结果的函数，或公开 RxJava **Single**、**Maybe**或**Completable**类型。
- **接收关于数据随时间变化的通知**：在 Kotlin 中，数据层应公开数据流；对于 Java 编程语言，数据层应公开用于发出新数据的回调，或公开 RxJava **Observable**或**Flowable**类型。

```kotlin
class ExampleRepository(
    private val exampleRemoteDataSource: ExampleRemoteDataSource, // network
    private val exampleLocalDataSource: ExampleLocalDataSource // database
) {

    val data: Flow<Example> = ...

    suspend fun modifyData(example: Example) { ... }
}
```

# 命名惯例
在本指南中，存储库类以其负责的数据命名。具体命名惯例如下：

数据类型 + Repository。

例如：**NewsRepository**、**MoviesRepository**。

数据源类以其负责的数据以及使用的来源命名。具体命名惯例如下：

数据类型 + 来源类型 + DataSource。

对于数据的类型，可以使用 Remote 或 Local，以使其更加通用，因为实现是可以变化的。例如：**NewsRemoteDataSource**或**NewsLocalDataSource**。在来源非常重要的情况下，为了更加具体，可以使用来源的类型。例如：**NewsNetworkDataSource**或**NewsDiskDataSource**。

请勿根据实现细节来为数据源命名（例如**UserSharedPreferencesDataSource**），因为使用相应数据源的存储库应该不知道数据是如何保存的。如果您遵循此规则，便可以更改数据源的实现（例如，从**SharedPreferences**迁移到**DataStore**），而不会影响调用相应数据源的层。

> 注意：迁移到数据源的新实现时，您可以为数据源创建接口，并使用两种数据源实现：一种用于旧的后备技术，另一种用于新的技术。在这种情况下，您可以将技术名称用作数据源类名称（尽管它是一个实现细节），因为存储库只能看到接口，而看不到数据源类本身。完成迁移后，您可以重命名新类，使其名称中不包含实现细节。

# 多层存储库
在某些涉及更复杂业务要求的情况下，存储库可能需要依赖于其他存储库。这可能是因为所涉及的数据是来自多个数据源的数据聚合，或者是因为相应职责需要封装在其他存储库类中。

例如，负责处理用户身份验证数据的存储库**UserRepository**可以依赖于其他存储库（例如**LoginRepository**和**RegistrationRepository**），以满足其要求。

![图 2. 某个存储库与其他存储库的依赖关系图。](https://developer.android.google.cn/static/topic/libraries/architecture/images/mad-arch-data-multiple-repos.png?hl=zh-cn)

> 注意：传统上，一些开发者将依赖于其他存储库类的存储库类称为 manager，例如称为 UserManager 而非 UserRepository。如果您愿意，可以使用此命名惯例。

# 可信来源
每个存储库都只定义单个可信来源，这一点非常重要。可信来源始终包含一致、正确且最新的数据。实际上，从存储库公开的数据应始终是直接来自可信来源的数据。

可信来源可以是数据源（例如数据库），甚至可以是存储库可能包含的内存中缓存。存储库可合并不同的数据源，并解决数据源之间的所有潜在冲突，以便定期更新或因应用户输入事件更新单个可信来源。

应用中的不同存储库可以具有不同的可信来源。例如，**LoginRepository**类可以将其缓存用作可信来源，**PaymentsRepository**类则可以使用网络数据源。

为了提供离线优先支持，**建议使用本地数据源（例如数据库）作为可信来源**。

# 线程处理
调用数据源和存储库应该具有主线程安全性（即从主线程调用是安全的）。在执行长时间运行的阻塞操作时，这些类负责将其逻辑的执行移至适当的线程。例如，对于数据源，从文件读取数据应该具有主线程安全性；对于仓库，对大列表执行非常耗费资源的过滤应该具有主线程安全性。

# 生命周期
数据层中的类的实例会保留在内存中，前提是它们可以从垃圾回收根访问 - 通常是从应用中的其他对象引用。

如果某个类包含内存中的数据（例如缓存），您可能希望在特定时间段内重复使用该类的同一实例。这也称为类实例的生命周期。

如果该类的职责对于整个应用至关重要，您可以将该类的实例的作用域限定为**Application**类。这可让该实例遵循应用的生命周期。或者，如果您只需要在应用内的特定流程（例如注册流程或登录流程）中重复使用同一实例，则应将该实例的作用域限定为负责相应流程的生命周期的类。例如，您可以将包含内存中数据的**RegistrationRepository**的作用域限定为**RegistrationActivity**，或限定为注册流程的导航图。

每个实例的生命周期都是决定如何在应用内提供依赖项的关键因素。建议您遵循依赖项注入方面的最佳实践来管理依赖项，并可以将依赖项的作用域限定为依赖项容器。

# 表示业务模式
您想要从数据层公开的数据模型可能是您从不同数据源获取的信息的子集。理想情况下，不同数据源（网络数据源和本地数据源）应该只返回应用需要的信息；但通常并非如此。

例如，假设有一个 News API 服务器，它不仅返回报道信息，还会返回修改记录、用户评论和部分元数据：
```kotlin
data class ArticleApiModel(
    val id: Long,
    val title: String,
    val content: String,
    val publicationDate: Date,
    val modifications: Array<ArticleApiModel>,
    val comments: Array<CommentApiModel>,
    val lastModificationDate: Date,
    val authorId: Long,
    val authorName: String,
    val authorDateOfBirth: Date,
    val readTimeMin: Int
)
```

该应用不需要这么多关于报道的信息，因为它在屏幕上只显示报道内容，以及关于作者的基本信息。一种很好的做法是，分离模型类，并让存储库仅公开层次结构的其他层所需的数据。例如，以下代码段展示了您可以如何从网络中删减**ArticleApiModel**，以便将**Article**模型类公开给网域层和界面层：
```kotlin
data class Article(
    val id: Long,
    val title: String,
    val content: String,
    val publicationDate: Date,
    val authorName: String,
    val readTimeMin: Int
)
```

分离模型类可以带来以下好处：

- 将数据减少到只包含需要的内容，从而节省应用内存。
- 根据应用所使用的数据类型来调整外部数据类型 - 例如，应用可以使用不同的数据类型来表示日期。
- 更好地分离关注点 - 例如，如果预先定义了模型类，大型团队的成员便可以在功能的网络层和界面层单独开展工作。

您可以扩展这种做法，并可以在应用架构的其他部分（例如，在数据源类和**ViewModel**中）定义单独的模型类。不过，这需要您定义额外的类和逻辑，并且您应正确记录和测试这些类和逻辑。**至少，我们建议您在数据源接收的数据与应用其余部分所需的数据不符时，创建新模型**。

# 数据操作类型
数据层可以处理的操作类型会因操作的重要程度而异：面向界面的操作、面向应用的操作和面向业务的操作。

## 面向界面的操作
面向界面的操作仅在用户位于特定屏幕上时才相关，当用户离开相应屏幕时便会被取消。例如，显示从数据库获取的部分数据。

面向界面的操作通常由界面层触发，并且遵循调用方的生命周期，例如**ViewModel**的生命周期。

## 面向应用的操作
只要应用处于打开状态，面向应用的操作就一直相关。如果应用关闭或进程终止，这些操作将会被取消。例如，缓存网络请求结果，以便在以后需要时使用。

这些操作通常遵循**Application**类或数据层的生命周期。

## 面向业务的操作
面向业务的操作无法取消。它们应该会在进程终止后继续执行。例如，完成上传用户想要发布到其个人资料的照片。

对于面向业务的操作，建议使用**WorkManager**。

# 公开错误
与存储库和数据源的互动可能会成功，也可能会在出现故障时抛出异常。对于协程和数据流，您应使用**Kotlin**的内置错误处理机制。对于可以由挂起函数触发的错误，请根据需要使用**try/catch**块；在数据流中，请使用**catch**运算符。如果使用这种方式，界面层应负责处理在调用数据层时出现的异常。

数据层可以理解和处理不同类型的错误，并可以使用自定义异常（例如**UserNotAuthenticatedException**）公开这些错误。

> 注意：若要为与数据层的互动结果建模，另一种方法是使用 Result 类。此模式会为在处理结果时可能出现的错误和其他信号进行建模。在此模式中，数据层会返回 Result<T> 类型，而非 T，以便让界面知道在特定情况下可能发生的已知错误。对于没有适当异常处理机制的反应式编程 API（例如 LiveData）来说，必须要使用这种方法。

# 常见任务
以下几个部分举例说明了如何使用和构建数据层来执行 Android 应用中常见的特定任务。这些示例基于本指南前面提到的典型“新闻”应用。

## 发出网络请求
发出网络请求是 Android 应用可能执行的最常见任务之一。“新闻”应用需要向用户提供从网络获取的最新新闻。因此，该应用需要一个数据源类来管理网络操作：**NewsRemoteDataSource**。为了向该应用的其余部分公开信息，我们创建了一个用于处理新闻数据操作的新存储库：**NewsRepository**。

该应用需要满足的要求是，当用户打开屏幕时，该应用一律需要更新最新新闻。因此，这是一项面向界面的操作。

### 创建数据源
数据源需要公开一个用于返回最新新闻（**ArticleHeadline**实例的列表）的函数。数据源需要提供一种具有主线程安全性的方式，以便从网络获取最新新闻。为此，它需要依赖于**CoroutineDispatcher**或**Executor**来运行任务。

发出网络请求是由新的 fetchLatestNews() 方法处理的一次性调用：
```kotlin
class NewsRemoteDataSource(
  private val newsApi: NewsApi,
  private val ioDispatcher: CoroutineDispatcher
) {
    /**
     * Fetches the latest news from the network and returns the result.
     * This executes on an IO-optimized thread pool, the function is main-safe.
     */
    suspend fun fetchLatestNews(): List<ArticleHeadline> =
        // Move the execution to an IO-optimized thread since the ApiService
        // doesn't support coroutines and makes synchronous requests.
        withContext(ioDispatcher) {
            newsApi.fetchLatestNews()
        }
    }

// Makes news-related network synchronous requests.
interface NewsApi {
    fun fetchLatestNews(): List<ArticleHeadline>
}
```

**NewsApi**接口会隐藏网络 API 客户端的实现；接口是由 Retrofit 还是由 HttpURLConnection 提供支持，并没有区别。依赖于接口能够使 API 实现在应用中可交换。
> 要点：依赖于接口能够使 API 实现在应用中可交换。除了提供可扩缩性并可让您更轻松地替换依赖项之外，这还有利于进行测试，因为您可以在测试时注入虚构的数据源实现。

### 创建存储库
存储库类中不需要任何额外的逻辑，即可执行此任务，因此**NewsRepository**可充当网络数据源的代理。内存中缓存部分介绍了添加这一额外抽象层的好处。
```kotlin
// NewsRepository is consumed from other layers of the hierarchy.
class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource
) {
    suspend fun fetchLatestNews(): List<ArticleHeadline> =
        newsRemoteDataSource.fetchLatestNews()
}
```

## 实现内存中数据缓存
假设为“新闻”应用引入了一项新的要求：当用户打开屏幕时，如果用户之前已发出请求，那么该应用必须向用户显示缓存的新闻。否则，该应用应发出网络请求以获取最新新闻。

鉴于这项新的要求，当用户已打开该应用时，该应用必须在内存中保留最新新闻。因此，这是一项面向应用的操作。

### 缓存
通过添加内存中数据缓存，您可以在用户位于您的应用中时保留数据。缓存旨在使一些信息在内存中保存特定的时间长度，在此示例中，只要用户位于该应用中，就一直保存相应信息。缓存实现可以采用不同的形式。从简单的可变变量，到更为复杂、可以防止在多个线程上进行读/写操作的类，不一而足。可以在存储库中实现缓存，也可以在数据源类中实现缓存，具体取决于用例。

### 缓存网络请求结果
为了简单起见，**NewsRepository**使用可变变量来缓存最新新闻。为了保护来自不同线程的读写操作，使用了**Mutex**。

以下实现会将最新新闻信息缓存到存储库中的一个变量，该变量由**Mutex**提供写保护。如果网络请求结果是成功，数据将分配给**latestNews**变量。
```kotlin
class NewsRepository(
  private val newsRemoteDataSource: NewsRemoteDataSource
) {
    // Mutex to make writes to cached values thread-safe.
    private val latestNewsMutex = Mutex()

    // Cache of the latest news got from the network.
    private var latestNews: List<ArticleHeadline> = emptyList()

    suspend fun getLatestNews(refresh: Boolean = false): List<ArticleHeadline> {
        if (refresh || latestNews.isEmpty()) {
            val networkResult = newsRemoteDataSource.fetchLatestNews()
            // Thread-safe write to latestNews
            latestNewsMutex.withLock {
                this.latestNews = networkResult
            }
        }

        return latestNewsMutex.withLock { this.latestNews }
    }
}
```

### 让操作拥有比屏幕更长的生命周期
如果用户在网络请求正在进行时离开屏幕，系统将取消该请求，并且不会缓存结果。**NewsRepository**不应使用调用方的**CoroutineScope**来执行此逻辑。**NewsRepository**应使用附加到其生命周期的**CoroutineScope**。获取最新新闻必须是面向应用的操作。

为了遵循依赖项注入方面的最佳实践，**NewsRepository**应在其构造函数中接收一个作用域作为参数，而不是创建自己的**CoroutineScope**。由于存储库应在后台线程中执行大部分工作，因此您应使用**Dispatchers.Default**或您自己的线程池来配置**CoroutineScope**。
```kotlin
class NewsRepository(
    ...,
    // This could be CoroutineScope(SupervisorJob() + Dispatchers.Default).
    private val externalScope: CoroutineScope
) { ... }
```

由于**NewsRepository**已准备好使用外部**CoroutineScope**来执行面向应用的操作，因此它必须调用数据源，并使用由相应作用域启动的新协程来保存其结果：
```kotlin
class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource,
    private val externalScope: CoroutineScope
) {
    /* ... */

    suspend fun getLatestNews(refresh: Boolean = false): List<ArticleHeadline> {
        return if (refresh) {
            externalScope.async {
                newsRemoteDataSource.fetchLatestNews().also { networkResult ->
                    // Thread-safe write to latestNews.
                    latestNewsMutex.withLock {
                        latestNews = networkResult
                    }
                }
            }.await()
        } else {
            return latestNewsMutex.withLock { this.latestNews }
        } 
    }
}
```

**async**用于在外部作用域内启动协程。**await**在新的协程上调用，以便在网络请求返回结果并且结果保存到缓存中之前，一直保持挂起状态。如果届时用户仍位于屏幕上，就会看到最新新闻；如果用户已离开屏幕，**await**将被取消，但**async**内部的逻辑将继续执行。

## 将数据保存到磁盘以及从磁盘检索数据
假设您要保存一些数据，例如添加了书签的新闻和用户偏好设置。这种类型的数据需要在进程终止后继续保留，并且即使用户未连接到网络，也必须可供用户访问。

如果您处理的数据需要在进程终止后继续保留，则您需要通过以下方式之一将其存储在磁盘上：

- 对于需要查询、需要实现引用完整性或需要部分更新的大型数据集，请将数据保存在 Room 数据库中。在“新闻”应用示例中，新闻报道或作者信息可以保存在该数据库中。
- 对于只需要检索和设置（不需要查询，也不需要部分更新）的小型数据集，请使用 DataStore。在“新闻”应用示例中，用户的首选日期格式或其他显示偏好设置可以保存在 DataStore 中。
- 对于数据块（例如 JSON 对象），可以使用文件。

如可信来源部分所述，每个数据源都只能处理一个来源，并且与特定的数据类型（例如**News**、**Authors**、**NewsAndAuthors**或**UserPreferences**）相对应。使用数据源的类应该不知道数据是如何保存的，例如是保存在数据库中，还是保存在文件中。

### 使用 Room 作为数据源
由于每个数据源都应只负责处理一种特定类型的数据的一个数据源，因此 Room 数据源会接收数据访问对象 (DAO) 或数据库本身作为参数。例如，**NewsLocalDataSource**可以接收**NewsDao**的实例作为参数，**AuthorsLocalDataSource**则可以接收**AuthorsDao**的实例。

在某些情况下，如果不需要额外的逻辑，您可以直接将 DAO 注入存储库，因为 DAO 是一种可以在测试中轻松替换的接口。

### 使用 DataStore 作为数据源
DataStore 非常适合存储键值对，例如用户设置，具体示例可能包括时间格式、通知偏好设置，以及是显示还是隐藏用户已阅读的新闻报道。DataStore 还可以使用协议缓冲区来存储类型化对象。

与任何其他对象一样，由 DataStore 提供支持的数据源应包含与特定类型相对应或与应用的特定部分相对应的数据。对于 DataStore 来说更是如此，因为 DataStore 读取操作会作为一个每次值更新后都会发出的数据流进行公开。因此，您应将相关偏好设置存储在同一个 DataStore 中。

例如，您可以创建一个仅处理通知相关偏好设置的**NotificationsDataStore**，并创建一个仅处理新闻屏幕相关偏好设置的**NewsPreferencesDataStore**。这样，您就可以更好地限定更新作用域，因为只有当与相应屏幕相关的偏好设置发生变化时，**newsScreenPreferencesDataStore.data**流才会发出。这也意味着，该对象的生命周期可以更短，因为它只能在新闻屏幕显示时存在。

### 使用文件作为数据源

处理大型对象（例如 JSON 对象或位图）时，您需要使用**File**对象并处理线程切换。

## 使用 WorkManager 调度任务
假设为“新闻”应用引入了一项新的要求：只要设备正在充电并且已连接到不按流量计费的网络，该应用就必须为用户提供用于选择定期自动获取最新新闻的选项。这会使此操作成为一项面向业务的操作。如果该应用实现了这一要求，那么在用户打开该应用时，即使设备没有连接到网络，用户仍然可以看到最近的新闻。

借助 WorkManager，可以轻松调度异步的可靠工作，并可以负责管理约束条件。我们建议使用该库执行持久性工作。为了执行上面定义的任务，我们创建了一个 Worker 类：**RefreshLatestNewsWorker**。此类以**NewsRepository**作为依赖项，以便获取最新新闻并将其缓存到磁盘中。
```kotlin
class RefreshLatestNewsWorker(
    private val newsRepository: NewsRepository,
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result = try {
        newsRepository.refreshLatestNews()
        Result.success()
    } catch (error: Throwable) {
        Result.failure()
    }
}
```

此类任务的业务逻辑应封装在其自己的类中，并且应被视为单独的数据源。这样一来，WorkManager 将仅负责确保工作会在所有约束条件都得到满足时在后台线程中执行。通过遵循此模式，您可以根据需要在不同环境中快速交换实现。

在此示例中，必须从**NewsRepository**调用这个与新闻相关的任务，前者会将一个新的数据源作为依赖项：**NewsTasksDataSource**。实现方式如下：
```kotlin
private const val REFRESH_RATE_HOURS = 4L
private const val FETCH_LATEST_NEWS_TASK = "FetchLatestNewsTask"
private const val TAG_FETCH_LATEST_NEWS = "FetchLatestNewsTaskTag"

class NewsTasksDataSource(
    private val workManager: WorkManager
) {
    fun fetchNewsPeriodically() {
        val fetchNewsRequest = PeriodicWorkRequestBuilder<RefreshLatestNewsWorker>(
            REFRESH_RATE_HOURS, TimeUnit.HOURS
        ).setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.TEMPORARILY_UNMETERED)
                .setRequiresCharging(true)
                .build()
        )
            .addTag(TAG_FETCH_LATEST_NEWS)

        workManager.enqueueUniquePeriodicWork(
            FETCH_LATEST_NEWS_TASK,
            ExistingPeriodicWorkPolicy.KEEP,
            fetchNewsRequest.build()
        )
    }

    fun cancelFetchingNewsPeriodically() {
        workManager.cancelAllWorkByTag(TAG_FETCH_LATEST_NEWS)
    }
}
```

这些类型的类以其负责的数据命名，例如**NewsTasksDataSource**或**PaymentsTasksDataSource**。与特定类型的数据相关的所有任务都应封装在同一个类中。

如果任务需要在应用启动时触发，建议使用从**Initializer**调用存储库的 App Startup 库触发 WorkManager 请求。