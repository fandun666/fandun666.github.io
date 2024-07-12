---
title: Android Developers 应用架构指南[构建离线优先应用]
date: 2024-07-11 23:00:00
tags: [Android]
categories: [应用架构]
---
离线优先应用是无需访问互联网就能执行全部核心功能或一部分关键核心功能的应用。也就是说，它可以离线执行部分或全部业务逻辑。

构建离线优先应用首先要考虑数据层，它提供对应用数据和业务逻辑的访问。应用可能需要不时刷新这些来自设备外部来源的数据。为此，它可能需要调用网络资源来保持更新。

网络的可用性并不一定总是能得到保证。设备往往免不了会遇到网络连接不稳定或速度缓慢的问题。用户也可能会遇到以下情况：

- 互联网带宽有限。
- 连接短暂中断，例如在电梯或隧道中。
- 偶尔才能访问数据。例如，使用的平板电脑仅支持 Wi-Fi 连接。

不管原因如何，应用通常可以在上述情况下正常运行。为了确保应用可在离线状态下正常运行，它应该具备以下能力：

- 在没有可靠网络连接的情况下仍可使用。
- 立即向用户提供本地数据，而不是等待第一次网络调用完成或失败。
- 提取数据的方式考虑到电池和数据状态。例如，仅在理想情况下（例如充电或有 Wi-Fi 连接时）请求提取数据。

满足上述标准的应用通常称为离线优先应用。

# 设计离线优先应用
在设计离线优先应用时，首先应该设计数据层以及您可以对应用数据执行的以下两项主要操作：

- **读取**：检索数据以供应用的其他部分使用，例如向用户显示信息。
- **写入**：持久存储用户输入供日后检索之用。

数据层中的存储库负责组合数据源以提供应用数据。在离线优先应用中，必须至少有一个数据源无需访问网络即可执行其最关键的任务。其中一项关键任务是读取数据。
> 注意：离线优先应用至少应能在不访问网络的情况下执行读取操作。

# 模型数据
对于需要使用网络资源的每个存储库，离线优先应用至少有 2 个数据源：

- 本地数据源
- 网络数据源

![图 1：离线优先存储库](https://developer.android.google.cn/static/images/topic/architecture/data-layer/data-layer.png?hl=zh-cn)

> 注意：离线优先应用中需要访问网络数据源的存储库应当始终具有本地数据源。

## 本地数据源
本地数据源是应用的规范可信来源。应用的较高层读取任何数据，都应将其作为专属来源。这样可在处于两次连接之间的状态时确保数据一致性。本地数据源通常由存储空间提供支持并持久存储到磁盘。下面是将数据持久存储到磁盘的一些常用方法：

- 结构化数据源，例如 Room 等关系型数据库。
- 非结构化数据源。例如，Datastore 的协议缓冲区。
- 简单文件

## 网络数据源
网络数据源是应用的实际状态。最好将本地数据源与网络数据源同步。本地数据源也有可能滞后于网络数据源，在这种情况下，应用需要在重新联网后进行更新。相反，网络数据源可以滞后于本地数据源，待连接恢复后，应用便可对其进行更新。应用的网域层和界面层绝不应直接与网络层通信，而应由托管**repository**负责与其通信并用其更新本地数据源。

## 公开资源
应用对本地数据源和网络数据源执行读写操作的方式存在根本差异。查询本地数据源既快速又灵活，例如在使用 SQL 查询时。相反，查询网络数据源可能又慢又受到限制，例如在通过 ID 逐步访问 RESTful 资源时。这导致每种数据源对其提供的数据通常需要采用自己的表示形式。因此，本地数据源和网络数据源可能有自己的模型。

下面的目录结构直观体现了这一概念。**AuthorEntity**表示从应用的本地数据库读取的作者，而**NetworkAuthor**表示通过网络序列化的作者：
```kotlin
data/
├─ local/
│ ├─ entities/
│ │ ├─ AuthorEntity
│ ├─ dao/
│ ├─ NiADatabase
├─ network/
│ ├─ NiANetwork
│ ├─ models/
│ │ ├─ NetworkAuthor
├─ model/
│ ├─ Author
├─ repository/
```

接下来是**AuthorEntity**和**NetworkAuthor**的详细信息：
```kotlin
/**
 * Network representation of [Author]
 */
@Serializable
data class NetworkAuthor(
    val id: String,
    val name: String,
    val imageUrl: String,
    val twitter: String,
    val mediumPage: String,
    val bio: String,
)

/**
 * Defines an author for either an [EpisodeEntity] or [NewsResourceEntity].
 * It has a many-to-many relationship with both entities
 */
@Entity(tableName = "authors")
data class AuthorEntity(
    @PrimaryKey
    val id: String,
    val name: String,
    @ColumnInfo(name = "image_url")
    val imageUrl: String,
    @ColumnInfo(defaultValue = "")
    val twitter: String,
    @ColumnInfo(name = "medium_page", defaultValue = "")
    val mediumPage: String,
    @ColumnInfo(defaultValue = "")
    val bio: String,
)
```

最好将**AuthorEntity**和**NetworkAuthor**都留在数据层内部，公开第三种类型供外部层使用。这可以保护外部层免受本地数据源和网络数据源中不会从根本上改变应用行为的细微更改影响。这种做法如以下代码段所示：
```kotlin
/**
 * External data layer representation of a "Now in Android" Author
 */
data class Author(
    val id: String,
    val name: String,
    val imageUrl: String,
    val twitter: String,
    val mediumPage: String,
    val bio: String,
)
```

然后，网络模型可定义一种用于将其转换为本地模型的扩展方法，本地模型同样也可定义一种用于将其转换为外部表示形式的扩展方法，如下所示：
```kotlin
/**
 * Converts the network model to the local model for persisting
 * by the local data source
 */
fun NetworkAuthor.asEntity() = AuthorEntity(
    id = id,
    name = name,
    imageUrl = imageUrl,
    twitter = twitter,
    mediumPage = mediumPage,
    bio = bio,
)

/**
 * Converts the local model to the external model for use
 * by layers external to the data layer
 */
fun AuthorEntity.asExternalModel() = Author(
    id = id,
    name = name,
    imageUrl = imageUrl,
    twitter = twitter,
    mediumPage = mediumPage,
    bio = bio,
)
```

> 注意：如上所示的映射器通常在不同模块中定义的模型之间进行映射。因此，在使用这些映射器的模块中对其进行定义以免模块紧密耦合通常是一种有益的做法。

# 读取
读取是离线优先应用中应用数据的基本操作。因此，您必须确保您的应用可以读取数据，并确保一旦有新数据可用，应用便可以显示这些数据。能够做到这一点的应用属于响应式应用，因为它们会公开具有可观察类型的读取 API。

在下面的代码段中，**OfflineFirstTopicRepository**会为其所有读取 API 返回**Flows**。这样，当它收到来自网络数据源的更新时，便可以更新自己的读取器。换言之，它允许**OfflineFirstTopicRepository**在本地数据源失效时推送更改。因此，**OfflineFirstTopicRepository**的每个读取器都必须准备好在应用恢复网络连接时处理可能触发的数据更改。此外，**OfflineFirstTopicRepository**还会直接从本地数据源读取数据。它只能先更新本地数据源，通过这种更新将数据更改通知读取器。
```kotlin
class OfflineFirstTopicsRepository(
    private val topicDao: TopicDao,
    private val network: NiaNetworkDataSource,
) : TopicsRepository {

    override fun getTopicsStream(): Flow<List<Topic>> =
        topicDao.getTopicEntitiesStream()
            .map { it.map(TopicEntity::asExternalModel) }
}
```

> 注意：从离线优先应用中的存储库读取数据应直接从本地数据源读取。所有更新均应先写入本地数据源，本地数据源会更新其使用方，因为它可观察。

## 本地数据源
从本地数据源读取数据时遇到的错误应当极少。为防止读取器出错，请对读取器从中收集数据的**Flows**使用**catch**操作符。

在**ViewModel**中使用**catch**操作符的代码如下所示：
```kotlin
class AuthorViewModel(
    authorsRepository: AuthorsRepository,
    ...
) : ViewModel() {
   private val authorId: String = ...

   // Observe author information
    private val authorStream: Flow<Author> =
        authorsRepository.getAuthorStream(
            id = authorId
        )
        .catch { emit(Author.empty()) }
}
```

> 注意：catch 操作符只能防止因异常而导致应用崩溃，后备 Flow 仍会终止。如需在发生异常后恢复从该数据流收集数据，请考虑使用 retry 方法。

## 网络数据源
从网络数据源读取数据时如果发生错误，应用需要采用启发法来重试提取数据。常见的启发法包括：

### 指数退避算法
在指数退避算法中，应用不断尝试从网络数据源读取数据，两次尝试间的时间间隔也不断增加，直到读取成功或其他条件决定其应停止读取为止。

![图 2：使用指数退避算法读取数据](https://developer.android.google.cn/static/images/topic/architecture/data-layer/read-backoff.png?hl=zh-cn)

评估应用是否应继续退避的标准包括：

- 网络数据源指出的错误类型。例如，如果网络调用返回的错误指出没有连接，就应该重试该网络调用。反之，如果 HTTP 请求未获授权，那么在获得正确的凭据之前，就不应重试该 HTTP 请求。
- 允许的最大重试次数。

### 网络连接监控
在此方法中，在应用确定可以连接到网络数据源之前，系统会将读取请求加入队列。连接建立后，系统会将读取请求移出队列，读取数据并更新本地数据源。在 Android 上，可使用 Room 数据库维护此队列，并使用 WorkManager 将其作为持久性工作排空。

![图 3：使用网络监控的读取队列](https://developer.android.google.cn/static/images/topic/architecture/data-layer/read-queue.png?hl=zh-cn)

# 写入
读取离线优先应用中数据的建议方式是使用可观察类型，而写入 API 的等效方式是异步 API，例如挂起函数。这可以避免阻塞界面线程，并且有助于处理错误，因为离线优先应用中的写入操作可能会在跨越网络边界时失败。
```kotlin
interface UserDataRepository {
    /**
     * Updates the bookmarked status for a news resource
     */
    suspend fun updateNewsResourceBookmark(newsResourceId: String, bookmarked: Boolean)
}
```

在上面的代码段中，所选的异步 API 是协程，因为上述方法挂起。

在离线优先应用中写入数据时，可以考虑采取三种策略。具体选择哪种策略取决于要写入的数据类型以及应用的要求：

## 仅在线写入
尝试跨网络边界写入数据。如果成功，就更新本地数据源，否则抛出异常并留待调用方进行适当响应。

![图 4：仅在线写入](https://developer.android.google.cn/static/images/topic/architecture/data-layer/write-online-only.png?hl=zh-cn)

此策略通常用于必须近乎实时地在线执行的写入事务。例如，银行转账。由于写入可能会失败，因此通常有必要告知用户写入失败，或者从一开始就阻止用户尝试写入数据。在此类情况下，您可以采取的策略可能包括：

- 如果应用需要访问互联网才能写入数据，可以选择不向用户显示可供用户写入数据的界面，或至少也要停用该界面。
- 您可以使用一个用户无法关闭的弹出式消息或一个短暂提示来通知用户他们处于离线状态。

## 加入队列的写入
如果您有想要写入的对象，请将其插入队列。当应用恢复在线状态时，继续使用指数退避算法排空队列。在 Android 上，排空离线队列是一项持久性工作，通常委托给**WorkManager**。

![图 5：使用重试的写入队列](https://developer.android.google.cn/static/images/topic/architecture/data-layer/write-backoff.png?hl=zh-cn)

此方法适合以下情况：

- 将数据写入网络并非必不可少。
- 事务对时效的要求不高。
- 如果操作失败，并非一定要通知用户。

适合此方法的用例包括分析事件和日志记录。

## 延迟写入
先写入本地数据源，然后将写入请求加入队列，以便尽快通知网络数据源。这一点非常重要，因为当应用恢复在线状态时，网络数据源与本地数据源之间可能会存在冲突。下一部分将详细介绍如何解决冲突。

![图 6：延迟写入](https://developer.android.google.cn/static/images/topic/architecture/data-layer/write-queue.png?hl=zh-cn)

当数据对应用至关重要时，此方法是正确的选择。例如，在待办事项列表离线优先应用中，用户离线添加的任何任务都必须存储在本地，以避免数据丢失的风险。
> 注意：由于存在潜在冲突，在离线优先应用中写入数据通常比读取数据需要考虑更多方面。离线优先应用想要被视为离线优先，并不一定需要在离线状态下能够写入数据。

# 同步和解决冲突
离线优先应用恢复连接时，需要使本地数据源中的数据与网络数据源中的数据一致。此过程称为同步。应用与网络数据源同步主要有两种方式：

- 基于拉取的同步
- 基于推送的同步

## 基于拉取的同步
在基于拉取的同步中，应用在需要的时候连接到网络数据源读取最新应用数据。此方法的一种常用启发法基于用户导航，采用这种方法时，应用仅在向用户提供数据之前提取数据。

当应用预计短时间到中等长度的时间内没有网络连接时，最适合使用此方法。这是因为数据刷新需要伺机而为，长时间没有网络连接会提高用户尝试使用过时缓存或空缓存访问应用目的地的几率。

![图 7：基于拉取的同步：设备 A 只为屏幕 A 和屏幕 B 访问资源，而设备 B 只为屏幕 B、屏幕 C 和屏幕 D 访问资源](https://developer.android.google.cn/static/images/topic/architecture/data-layer/pull-sync.png?hl=zh-cn)

假设在一个应用中，使用页面令牌为某个特定屏幕提取无限滚动列表中的项目。该应用的实现可延迟连接到网络数据源，将数据持久存储到本地数据源，然后从本地数据源读取数据以向用户显示信息。在没有网络连接的情况下，存储库可以只向本地数据源请求数据。以下是 Jetpack Paging 库通过其 RemoteMediator API 使用的模式。
```kotlin
class FeedRepository(...) {

    fun feedPagingSource(): PagingSource<FeedItem> { ... }
}

class FeedViewModel(
    private val repository: FeedRepository
) : ViewModel() {
    private val pager = Pager(
        config = PagingConfig(
            pageSize = NETWORK_PAGE_SIZE,
            enablePlaceholders = false
        ),
        remoteMediator = FeedRemoteMediator(...),
        pagingSourceFactory = feedRepository::feedPagingSource
    )

    val feedPagingData = pager.flow
}
```

下表总结了基于拉取的同步的优缺点：

|  优点   | 缺点  |
|  ----  | ----  |
| 实现起来相对容易。  | 容易消耗大量流量。这是因为重复访问导航目的地会触发不必要的操作，重新提取未更改的信息。您可以通过适当的缓存来减少此问题。若要使用缓存，可在界面层使用 cachedIn 操作符或在网络层使用 HTTP 缓存。 |
| 绝不会提取不需要的数据。  | 不能使用关系型数据很好地扩展，因为拉取的模型需要自给自足。如果待同步的模型依赖于需要提取的其他模型来填充自己，那么上面提到的消耗大量流量的问题将变得更加严重。此外，它还可能导致父模型的存储库与嵌套模型的存储库之间存在依赖关系。 |

## 基于推送的同步
在基于推送的同步中，本地数据源会尽力尝试模拟网络数据源的副本集。它会在首次启动时主动提取适当数量的数据来设置基准，之后依靠来自服务器的通知提醒自己数据何时过时。

![图 8：基于推送的同步：网络数据源在数据更改时通知应用，应用通过提取经过更改的数据做出响应](https://developer.android.google.cn/static/images/topic/architecture/data-layer/push-sync.png?hl=zh-cn)

收到过时通知后，应用连接到网络数据源，只更新标记为过时的数据。这项工作将委托给**Repository**，由其连接到网络数据源，并将提取的数据持久存储到本地数据源。由于存储库通过可观测类型公开其数据，因此读取器将收到所有更改的通知。
```kotlin
class UserDataRepository(...) {

    suspend fun synchronize() {
        val userData = networkDataSource.fetchUserData()
        localDataSource.saveUserData(userData)
    }
}
```

在此方法中，应用对网络数据源的依赖要低得多，而且长时间无法使用网络数据源也能正常运行。它可以在离线状态下提供读写访问，因为系统假定本地存储着来自网络数据源的最新信息。

下表总结了基于推送的同步的优缺点：

|  优点   | 缺点  |
|  ----  | ----  |
| 应用可以无限期离线使用。  | 为了解决冲突，对数据进行版本控制非常重要。 |
| 可将流量消耗降到最低。应用仅提取经过更改的数据。  | 需要考虑同步期间的写入问题。 |
| 非常适合关系型数据。每个存储库只负责为其支持的模型提取数据。  | 网络数据源需要支持同步。 |

## 混合同步
某些应用采用混合方法，具体基于拉取还是基于推送根据数据而定。例如，某个社交媒体应用可能会使用基于拉取的同步按需提取用户的关注 Feed，因为 Feed 更新的频率较高。然而，同一应用可能会选择使用基于推送的同步来提取已登录用户的相关数据，包括其用户名、个人资料照片等。

最终，离线优先同步的选择取决于产品要求和可用的技术基础架构。
> 注意：应用的同步方法取决于应用的需求以及支持本地数据源和网络数据源的基础架构的限制。

# 冲突解决
如果应用处于离线状态时在本地写入的数据与网络数据源的数据不一致，说明存在冲突，必须解决冲突后才能进行同步。

解决冲突问题通常需要借助版本控制。应用需要通过一些簿记来跟踪发生更改的时间。这样，它就能将元数据传递给网络数据源。然后，由网络数据源负责提供绝对可信来源。根据应用的需求，可以考虑的冲突解决策略还有很多。对于移动应用，常见的方法是“最后写入内容生效”。

在此方法中，设备将时间戳元数据附加到其写入网络数据源的数据中。网络数据源在收到这些数据后，会舍弃比当前状态旧的所有数据而接受比当前状态新的数据。

![图 9：“最后写入内容生效”：数据的可信来源由最后一个写入数据的实体决定](https://developer.android.google.cn/static/images/topic/architecture/data-layer/last-write-wins.png?hl=zh-cn)

在上图中，两部设备都处于离线状态，并且最初都与网络数据源同步。离线时，它们都在本地写入数据并跟踪自己写入数据的时间。当二者恢复在线状态并与网络数据源同步时，网络数据源通过持久存储来自设备 B 的数据来解决冲突，因为设备 B 写入数据的时间更晚。

# 离线优先应用中的 WorkManager
在前面介绍的读取和写入策略中，有两个常用的实用程序：

- 队列
    - 读取：用于将读取操作推迟到网络连接可用时。
    - 写入：用于将写入操作推迟到网络连接可用时，并将写入操作重新加入队列进行重试。
- 网络连接监视器
    - 读取：在应用连接时用作排空读取队列的信号，也用于同步
    - 写入：在应用连接时用作排空写入队列的信号，也用于同步

这两种情况都是 WorkManager 擅长的持久性工作的例子。例如，在 Now in Android 示例应用中，同步本地数据源时将 WorkManager 用作读取队列监视器和网络监视器。在启动时，该应用会执行以下操作：

1. 将读取同步工作加入队列，以确保本地数据源和网络数据源相同。
2. 排空读取同步队列，并在应用处于在线状态时开始同步。
3. 使用指数退避算法执行从网络数据源读取数据的操作。
4. 将读取结果持久存储到本地数据源中，解决可能发生的任何冲突。
5. 公开本地数据源中的数据，供应用的其他层使用。

上述过程如下图所示：

![图 10：Now in Android 应用中的数据同步](https://developer.android.google.cn/static/images/topic/architecture/data-layer/nia-sync.png?hl=zh-cn)

使用 WorkManager 将同步工作加入队列后，使用**ExistingWorkPolicy.KEEP**将其指定为唯一工作：
```kotlin
class SyncInitializer : Initializer<Sync> {
   override fun create(context: Context): Sync {
       WorkManager.getInstance(context).apply {
           // Queue sync on app startup and ensure only one
           // sync worker runs at any time
           enqueueUniqueWork(
               SyncWorkName,
               ExistingWorkPolicy.KEEP,
               SyncWorker.startUpSyncWork()
           )
       }
       return Sync
   }
}
```

> 注意：“Now in Android”中的读取队列非常简单，使用 enqueueUniqueWork API 来表示就足矣。为了进一步保证队列的排空顺序，需要使用 Room 或 Datastore 等数据持久化 API 实现更可靠的队列实现。然后，您可以设置一个 Worker 来按顺序排空此队列。

其中，SyncWorker.startupSyncWork() 的定义如下：
```kotlin

/**
 Create a WorkRequest to call the SyncWorker using a DelegatingWorker.
 This allows for dependency injection into the SyncWorker in a different
 module than the app module without having to create a custom WorkManager
 configuration.
*/
fun startUpSyncWork() = OneTimeWorkRequestBuilder<DelegatingWorker>()
    // Run sync as expedited work if the app is able to.
    // If not, it runs as regular work.
   .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
   .setConstraints(SyncConstraints)
    // Delegate to the SyncWorker.
   .setInputData(SyncWorker::class.delegatedData())
   .build()

val SyncConstraints
   get() = Constraints.Builder()
       .setRequiredNetworkType(NetworkType.CONNECTED)
       .build()
```

具体而言，由**SyncConstraints**定义的**Constraints**要求**NetworkType**为**NetworkType.CONNECTED**。也就是说，它会等到网络可用后再运行。

当网络可用后，工作器将**SyncWorkName**指定的唯一工作队列委托给适当的**Repository**实例来排空该队列。如果同步失败，doWork() 方法会返回 Result.retry()。WorkManager 将采用指数退避算法自动重试同步。否则，返回 Result.success() 完成同步。
```kotlin
class SyncWorker(...) : CoroutineWorker(appContext, workerParams), Synchronizer {

    override suspend fun doWork(): Result = withContext(ioDispatcher) {
        // First sync the repositories in parallel
        val syncedSuccessfully = awaitAll(
            async { topicRepository.sync() },
            async { authorsRepository.sync() },
            async { newsRepository.sync() },
        ).all { it }

        if (syncedSuccessfully) Result.success()
        else Result.retry()
    }
}
```

