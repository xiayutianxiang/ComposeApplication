# Android 开发高频知识点 01：状态管理、保存与恢复

## 1. `ViewModel`、`rememberSaveable` 和本地持久化分别解决什么问题？

### 标准回答

三者的核心区别是**状态的生命周期和数据规模**：

| 方案 | 适合保存 | 配置变更后 | 系统因内存压力杀进程后 | 用户主动结束页面后 |
| --- | --- | --- | --- | --- |
| `ViewModel` | 页面级状态、业务状态、网络请求结果 | 保留 | 不保留 | 不保留 |
| `rememberSaveable` | 少量、轻量的 Compose UI 状态 | 保留 | 通常可恢复 | 不保留 |
| 本地持久化（Room、DataStore、文件等） | 业务数据、草稿、用户偏好等 | 保留 | 保留 | 保留 |

可以这样记：


- `ViewModel` 主要解决旋转屏幕等配置变更导致的页面重建，避免重复加载和丢失页面状态。
- `rememberSaveable` 主要保存轻量 UI 状态，例如输入框内容、筛选条件、滚动位置。
- 本地持久化用于真正需要长期保存的数据，不能把大量对象塞进 `Bundle` 或 saved state。

### Compose 示例

```kotlin
@Composable
fun SearchScreen(
    viewModel: SearchViewModel = viewModel()
) {
    // 轻量 UI 状态：Activity 重建后仍可恢复
    var keyword by rememberSaveable { mutableStateOf("") }

    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    SearchContent(
        keyword = keyword,
        onKeywordChange = {
            keyword = it
            viewModel.search(it)
        },
        uiState = uiState
    )
}
```

### 面试易错点

1. `ViewModel` 不是永久存储。系统杀死应用进程后，`ViewModel` 实例和其中的内存数据都会消失。
2. `onSaveInstanceState`、`rememberSaveable` 和 `SavedStateHandle` 适合保存少量、可序列化的瞬时状态，不适合保存完整列表、Bitmap 或大对象。
3. 用户按返回键退出页面或调用 `finish()` 时，系统不会把它当作需要恢复的页面；这时应由业务决定是否写入本地存储。

## 2. 系统杀死应用进程后，如何恢复 `ViewModel` 中的关键状态？

### 标准回答

使用 `SavedStateHandle` 作为 `ViewModel` 的状态备份：

- 普通业务状态仍放在 `ViewModel` 中管理。
- 需要跨系统进程死亡恢复的少量关键字段，同时写入 `SavedStateHandle`。
- 页面重新创建时，系统通过 saved state 将这些字段传回新的 `ViewModel`。
- 大量或复杂数据不要依赖 saved state，应从 Room、DataStore 或网络重新加载。

```kotlin
class SearchViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val query = savedStateHandle.getStateFlow("query", "")

    val uiState: StateFlow<SearchUiState> = query
        .flatMapLatest { keyword ->
            repository.search(keyword)
        }
        .map { results -> SearchUiState.Success(results) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = SearchUiState.Empty
        )

    fun search(keyword: String) {
        savedStateHandle["query"] = keyword
    }
}
```

### 什么时候选哪种恢复方式？

| 场景 | 推荐方式 |
| --- | --- |
| 输入框当前文本、简单开关、滚动位置 | `rememberSaveable` |
| 页面筛选条件、当前详情 ID、分页位置 | `SavedStateHandle` + `ViewModel` |
| 用户草稿、登录信息、缓存列表 | Room、DataStore 或其他持久化方案 |
| 页面展示的大量网络数据 | `ViewModel` 管理状态，进程重启后重新请求或从本地缓存加载 |

### 一句话总结

`ViewModel` 负责“活着时管理页面状态”，`SavedStateHandle` 负责“进程被系统杀掉后恢复关键线索”，本地数据库负责“长期保存业务数据”。

### 参考

- [Activity 生命周期与界面状态保存](https://developer.android.com/guide/components/activities/activity-lifecycle)
- [保存界面状态](https://developer.android.com/topic/libraries/architecture/saving-states)
- [ViewModel 的 SavedState 模块](https://developer.android.com/topic/libraries/architecture/viewmodel/viewmodel-savedstate)

## 3. Compose 中为什么要做状态提升？如何实现单向数据流？

### 标准回答

状态提升（State Hoisting）是把状态从子 Composable 移到共同的父级，由父级持有状态并通过参数下发、通过回调接收事件。这样可以让 UI 组件保持无状态、易复用和易测试，并明确“状态向下、事件向上”的单向数据流。

```kotlin
@Composable
fun SearchRoute(viewModel: SearchViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    SearchContent(
        keyword = uiState.keyword,
        onKeywordChange = viewModel::onKeywordChange
    )
}

@Composable
fun SearchContent(
    keyword: String,
    onKeywordChange: (String) -> Unit
) {
    TextField(value = keyword, onValueChange = onKeywordChange)
}
```

### 面试易错点

1. 不要为了“无状态”把所有状态都塞进 `ViewModel`；纯 UI 瞬时状态（例如输入框展开状态）可以由合适的 Composable 层持有。
2. `remember` 只在重组之间保留值，配置变更后会丢失；需要跨配置变更保存的轻量值使用 `rememberSaveable`。
3. `derivedStateOf` 适合输入变化频繁但输出不需要同频更新的派生状态，不应对每个普通计算都包一层。

## 4. `StateFlow` 和 `SharedFlow` 在 ViewModel 中如何选择？

### 标准回答

`StateFlow` 表示“当前状态”：必须有初始值，始终保存最新值，新订阅者会立即收到当前值，适合页面 UI 状态。

`SharedFlow` 表示“事件流”：可以配置 replay 和缓冲策略，适合一次性事件，例如显示 Snackbar、导航或打开系统页面。事件是否需要重放、没有订阅者时是否允许丢失，应由业务语义决定。

```kotlin
class DetailViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(DetailUiState())
    val uiState: StateFlow<DetailUiState> = _uiState.asStateFlow()

    private val _events = MutableSharedFlow<DetailEvent>()
    val events: SharedFlow<DetailEvent> = _events.asSharedFlow()

    fun onSave() {
        viewModelScope.launch {
            repository.save()
            _events.emit(DetailEvent.ShowSavedMessage)
        }
    }
}
```

### 面试易错点

1. 不要把一次性导航事件直接放进 `StateFlow`；页面重建后可能重复消费，或者因为状态一直保留而重复导航。
2. UI 应在生命周期感知的收集方式中订阅，例如 Compose 使用 `collectAsStateWithLifecycle()`；普通协程收集应使用 `repeatOnLifecycle`。
3. `SharedFlow` 默认 `replay = 0`，没有活跃订阅者时发出的事件可能无法被之后的订阅者收到；需要可靠投递时应重新设计事件模型或配合持久化状态。
