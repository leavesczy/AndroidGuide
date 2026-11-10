历经两年的打磨，在 2021 年七月底的时候，Google 终于发布了 Jetpack Compose 的正式版本。我对 Jetpack Compose 的很多特性都很感兴趣，也觉得其很可能会成为以后 Android 原生应用开发时的首选技术方案，所以在正式版发布之前就开始了一次实战演练，花了两个月时间断断续续写了一个完全用 Jetpack Compose 实现的 IM APP，实现了基本的聊天功能，同时也写了一篇文章对其进行介绍：[学不动也要学，Jetpack Compose 写一个 IM APP（一）](https://juejin.cn/post/6991429231821684773)

在文章中，我以 [compose_chat](https://github.com/leavesczy/compose_chat) 为例，详细地介绍了 Jetpack Compose 的很多重要概念和基础特性，包括：

- 命令式与声明式
- 可组合函数
- 状态
- 状态提升
- 纯函数
- 副作用
- 布局
- 动画 & 手势操作
- 主题

当时主要介绍的是 Jetpack Compose 这种新兴的声明式开发和传统 View 体系的命令式开发之间的巨大差异性，以及这种差异性随之带来的各种开发理念的改变。在我看来，该文章对于打算学习 Jetpack Compose 的同学是一份挺好的入门指南，compose_chat 也有助于同学们快速掌握开发套路

到现在又过了有几个月时间，在这段时间里我对 compose_chat 进行了大量优化，并且也新增了一些功能，最大的一个功能点就是当前已经支持 **群聊** 功能了，compose_chat 中预留了三个聊天群，有兴趣的同学可以去体验下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/adbeccad79d94d0dbff85bc02d7ccf04~tplv-k3u1fbpfcp-zoom-1.image)

上一篇文章中我主要讲解的是概念，对于具体的代码介绍得比较少，此篇文章就再来补充具体的代码细节，以及一些可以和 Jetpack Compose 无缝结合使用的 Jetpack 库和三方开源库。由于现在关于 Jetpack Compose 的学习资料还比较少，有些地方也许我也理解错了，如有错误希望读者能指出来，也希望对你有所帮助

# 一、MutableState

Jetpack Compose 通过嵌套多个可组合函数来描述整个屏幕状态并以此来绘制屏幕视图，而更新视图的唯一途径就是**生成新的入参参数并再次调用可组合函数**，新的入参参数即我们希望更新后的视图状态 State，每当 State 更新时就会触发可组合函数进行重组，从而实现 UI 刷新

为了让系统能够察觉到 State 被更新了，Compose 提供了 `MutableState<T>` 这种可观察类型，当其 value 发生了变化，系统就会重组所有依赖于该值的可组合函数

```kotlin
@Stable
interface MutableState<T> : State<T> {
    override var value: T
    operator fun component1(): T
    operator fun component2(): (T) -> Unit
}
```

我们可以通过 `mutableStateOf` 方法来创建一个 MutableState

```kotlin
var nickname = mutableStateOf("")
```

`MutableState<T>` 并不是用于存储状态的唯一一种方式，像 LiveData、Flow、RxJava 等可观察类型都是受支持的，而为了能够在状态发生变化时自动触发重组，就需要将这些类型转化为 `State<T>` 类型。Jetpack Compose 提供了相应的转换扩展函数，例如 LiveData 对应的就是 `observeAsState()` 方法，Flow 对应的是 `collectAsState()` 方法，它们都会在值发生变化的时候就自动调用 `state.value`

```kotlin
@Composable
fun <T> LiveData<T>.observeAsState(): State<T?> = observeAsState(value)

@Composable
fun <T> StateFlow<T>.collectAsState(
    context: CoroutineContext = EmptyCoroutineContext
): State<T> = collectAsState(value, context)
```

# 二、remember

可组合函数的**执行顺序、执行线程、甚至执行次数**都具有很大的不确定性。例如，在执行动画时一个可组合函数可能会在一秒内被执行 N 多次，这就会导致我们声明在函数内的对象在不断地在被重新初始化，赋予的值也会丢失

为了在组合期间能够记住某个对象值，此时就需要使用到 `remember` 方法了。系统会在初始组合期间将由 remember 指向的值存储在组合中，并在重组期间返回该值，从而保证值不会丢失。remember 既可用于存储可变对象，又可用于存储不可变对象

```kotlin
var nickname by remember() {
    mutableStateOf("")
}
```

以 compose_chat 为例，个人资料修改页面 HomeDrawerScreen 是通过 OutlinedTextField 来输入内容的，此时就需要同时使用到 MutableState 和 remember

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d86190406c0547c58795e75e71e23be9~tplv-k3u1fbpfcp-zoom-1.image)

当用户输入了新内容时，就会在 `onValueChange` 回调里改变 nickname，从而触发 DrawerFrontLayer 重组，而为了能够在重组期间保证 nickname 的值不会丢失，就需要使用到 remember 了

```kotlin
@Composable
private fun DrawerFrontLayer(
    backdropScaffoldState: BackdropScaffoldState,
    homeScreenDrawerState: HomeScreenDrawerState
) {
    Column() {
        var nickname by remember {
            mutableStateOf(
                homeScreenDrawerState.userProfile.nickname
            )
        }
        CommonOutlinedTextField(
            value = nickname,
            onValueChange = {
                if (it.length > 16) {
                    return@CommonOutlinedTextField
                }
                nickname = it
            },
            label = "nickname"
        )
    }
}
```

**remember 能够保证值不会丢失，但有时候我们也希望该值能够主动更新自身**

例如，当用户输入了新内容后，nickname 就被改变了，视图上就会一直显示着修改后的值。但此时用户并没有保存设置，所以就会一直显示着非真实昵称，就像下图所示

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb0570ab625b4bba83b25611d4cbd579~tplv-k3u1fbpfcp-zoom-1.image)

而我希望达到的效果是，当该浮层被关闭后，nickname 就能够被重置为用户的真实昵称，而非**脏数据**。此时就可以通过为 remember 方法添加 key 参数来达到该效果了，remember 方法支持传入多个 key 参数，当某个 key 发生变化时，就会重置自身重新初始化

例如，以下的 nickname 就以浮层的显示状态 `backdropScaffoldState.isRevealed` 作为了 key 之一，当浮层被关闭时，key 从 true 变成了 false，就会触发 nickname 重新初始化，从而丢弃之前输入的值

```kotlin
@Composable
private fun DrawerFrontLayer(
    backdropScaffoldState: BackdropScaffoldState,
    homeScreenDrawerState: HomeScreenDrawerState
) {
    Column() {
        var nickname by remember(
            key1 = backdropScaffoldState.isRevealed,
            key2 = homeScreenDrawerState
        ) {
            mutableStateOf(
                homeScreenDrawerState.userProfile.nickname
            )
        }
        CommonOutlinedTextField(
            value = nickname,
            onValueChange = {
                if (it.length > 16) {
                    return@CommonOutlinedTextField
                }
                nickname = it
            },
            label = "nickname"
        )
    }
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3eed7eed8174de9b273d823264267f5~tplv-k3u1fbpfcp-zoom-1.image)

# 三、CompositionLocal

通常情况下，数据是以传递参数的形式在整个视图树的可组合函数之间进行流转的

例如，为了设定 TextLabel 显示的文本内容和文本颜色，就需要 ChatApp 主动向其传递明确的参数值。这种数据传递形式也符合 Jetpack Compose 的编码原则：**一个合格的可组合函数就应该属于纯函数，幂等且没有副作用**

```kotlin
@Composable
fun ChatApp() {
    val colors = lightColors()
    TextLabel(labelText = "compose_chat", color = colors.primary)
}

@Composable
fun TextLabel(labelText: String, color: Color) {
    Text(
        text = labelText,
        color = color
    )
}
```

但这种方式在应对某些情况时就会显得很麻烦

例如，一个应用的主题都是有着明确的规范的，包括文本大小、文本颜色、阴影效果、圆角弧度等，这类属性值全局统一且发生变化的概率很低，如果调用每个可组合函数都需要显式传递这些属性值的话，那么就会使得函数的调用成本非常高

Jetpack Compose 提供了 CompositionLocal 来应对这种情况，通过组合来隐式向下传递数据。CompositionLocal 元素在视图树的某个节点以值的形式向内提供，供所有内嵌的可组合函数隐式使用，而无需在可组合函数中将 CompositionLocal 显式声明为参数，且每个可组合函数都会使用离自己最近的 CompositionLocal 元素

以官方提供的代码为例。LocalContentAlpha 用于定义内嵌内容的 Alpha 值，Column 就通过 CompositionLocal 划分为了三种不同的 Alpha 层次：默认透明度、medium、disabled

```kotlin
@Composable
fun CompositionLocalExample() {
    MaterialTheme {
        Column {
            Text("Uses MaterialTheme's provided alpha")
            CompositionLocalProvider(LocalContentAlpha provides ContentAlpha.medium) {
                Text("Medium value provided for LocalContentAlpha")
                Text("This Text also uses the medium value")
                CompositionLocalProvider(LocalContentAlpha provides ContentAlpha.disabled) {
                    Text("This Text uses the disabled alpha now")
                }
            }
        }
    }
}
```

最终的文本内容就会包含三种不同的透明度

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70d0c0cb17d147be9da344931a7404c6~tplv-k3u1fbpfcp-zoom-1.image)

Jetpack Compose 的主题也是通过 CompositionLocal 来实现的。以 compose_chat 为例，其主题由 ChatTheme 来定义，包含三种基本的主题属性，即：颜色 colors、排版 typography、形状 shapes，这些属性会自动反映在所有用来构建应用的组件中，可组合函数也可以主动读取上层视图赋予的属性

而整个应用都包裹在 ChatTheme 内部中，内部的可组合函数可以通过 `MaterialTheme.shapes.large` 的方式来直接获取 ChatTheme 中定义的属性值，避免了显式的参数传递

```kotlin
@Composable
fun ChatTheme(
    appTheme: AppTheme,
    content: @Composable () -> Unit
) {
    val colors = ...
    val typography = ... 
    val shapes = ...
    MaterialTheme(
        colors = colors,
        typography = typography,
        shapes = shapes,
        content = content
    )
}

ChatTheme {
    Text(
        text = "compose_chat",
        modifier = Modifier.clip(shape = MaterialTheme.shapes.large),
        color = MaterialTheme.colors.secondary,
        style = MaterialTheme.typography.body1,
    )
}
```

通过查看 MaterialTheme 方法的源码可以知道，我们传递的 colors 都会被保存在 LocalColors 中，而 MaterialTheme.colors 引用的正是 LocalColors。通过将代表整个应用的最高层级可组合函数包裹在 MaterialTheme 中，所有内嵌的可组合函数就都隐式包含了一整套统一的主题，即避免了显式的参数传递，也可以很方便地来调整整个应用的主题配置

```kotlin
@Composable
fun MaterialTheme(
    colors: Colors = MaterialTheme.colors,
    typography: Typography = MaterialTheme.typography,
    shapes: Shapes = MaterialTheme.shapes,
    content: @Composable () -> Unit
) {
    val rememberedColors = remember {
        // Explicitly creating a new object here so we don't mutate the initial [colors]
        // provided, and overwrite the values set in it.
        colors.copy()
    }.apply { updateColorsFrom(colors) }
    val rippleIndication = rememberRipple()
    val selectionColors = rememberTextSelectionColors(rememberedColors)
    CompositionLocalProvider(
        LocalColors provides rememberedColors,
        LocalContentAlpha provides ContentAlpha.high,
        LocalIndication provides rippleIndication,
        LocalRippleTheme provides MaterialRippleTheme,
        LocalShapes provides shapes,
        LocalTextSelectionColors provides selectionColors,
        LocalTypography provides typography
    ) {
        ProvideTextStyle(value = typography.body1) {
            PlatformMaterialTheme(content)
        }
    }
}

internal val LocalColors = staticCompositionLocalOf { lightColors() }

object MaterialTheme {

    val colors: Colors
        @Composable
        @ReadOnlyComposable
        get() = LocalColors.current

    val typography: Typography
        @Composable
        @ReadOnlyComposable
        get() = LocalTypography.current

    val shapes: Shapes
        @Composable
        @ReadOnlyComposable
        get() = LocalShapes.current
}
```

在项目中，我们可以仿照 MaterialTheme 的做法，为自定义的可组合函数提供获取某些系统资源的入口，例如可以用来获取系统键盘，即以下的 LocalInputMethodManager

```kotlin
val LocalInputMethodManager = staticCompositionLocalOf<InputMethodManager> {
    error("CompositionLocal InputMethodManager not present")
}

@Composable
fun ProvideInputMethodManager(content: @Composable () -> Unit) {
    val context = LocalContext.current
    val inputMethodManager = remember {
        context.getSystemService(
            Context.INPUT_METHOD_SERVICE
        ) as InputMethodManager
    }
    CompositionLocalProvider(LocalInputMethodManager provides inputMethodManager) {
        content()
    }
}

@Composable
fun ChatScreen() {
    ProvideInputMethodManager {
        val inputMethodManager = LocalInputMethodManager.current
        //TODO
    }
}
```

# 四、ViewModel

在 Jetpack Compose 中，我们的业务处理逻辑应该都是放在 ViewModel 中的，可组合函数不应该包含任何业务处理逻辑，而应该只根据入参参数的变化来进行视图更新

为了能够在可组合函数中**创建**或者是**拿到已有的** ViewModel 实例，一个必要的前提就是当前可组合函数要拥有 ViewModelStoreOwner 实例，两者的关系可以看我的另一篇文章：[从源码看 Jetpack（六）- ViewModel 源码详解](https://juejin.cn/post/6873356946896846856)

简单来说就是 ViewModel 需要依靠 ViewModelStoreOwner 来创建和存储，所有 ViewModel 实例都需要保存在 ViewModelStoreOwner 中，详情可以去看我的源码解读文章，这里不再赘述

Compose 包中提供了一个关于 ViewModel 的扩展函数 `viewModel`，可以很方便地在可组合函数中来获取 ViewModel 实例，就像以下这样

```kotlin
class LoginViewModel() : ViewModel()

@Composable
fun LoginScreen(navController: NavHostController) {
    val loginViewModel = viewModel<LoginViewModel>()
}
```

从源码可以看出 `viewModel` 方法也是依靠 CompositionLocal 来获取 ViewModelStoreOwner 实例的。Compose 在 Activity 创建时就自动为 LocalViewModelStoreOwner 赋值了，默认值就是和 Activity 相关联的 ViewModelStoreOwner

```kotlin
@Composable
public inline fun <reified VM : ViewModel> viewModel(
    viewModelStoreOwner: ViewModelStoreOwner = checkNotNull(LocalViewModelStoreOwner.current) {
        "No ViewModelStoreOwner was provided via LocalViewModelStoreOwner"
    },
    key: String? = null,
    factory: ViewModelProvider.Factory? = null
): VM = viewModel(VM::class.java, viewModelStoreOwner, key, factory)

@Composable
public fun <VM : ViewModel> viewModel(
    modelClass: Class<VM>,
    viewModelStoreOwner: ViewModelStoreOwner = checkNotNull(LocalViewModelStoreOwner.current) {
        "No ViewModelStoreOwner was provided via LocalViewModelStoreOwner"
    },
    key: String? = null,
    factory: ViewModelProvider.Factory? = null
): VM = viewModelStoreOwner.get(modelClass, key, factory)
```

此外，如果我们要获取的是包含构造参数的 ViewModel 实例的话，那就需要主动传入 `ViewModelProvider.Factory` 参数了，就像以下这样

```kotlin
class ChatViewModel(private val chat: Chat) : ViewModel()

@Composable
fun ChatScreen(chat: Chat) {
    val chatViewModel = viewModel<ChatViewModel>(factory = object :
        ViewModelProvider.NewInstanceFactory() {
        override fun <T : ViewModel> create(modelClass: Class<T>): T {
            return ChatViewModel(chat = chat) as T
        }
    })
}
```

此方法完全可行，但也存在模板代码，因为大多数情况下我们使用的都是 NewInstanceFactory，且都需要进行类型强转，区别只在于实例化 ViewModel 的过程，我们可以在 `viewModel` 方法的基础上再封装一个扩展函数，简化代码

```kotlin
@Composable
inline fun <reified VM : ViewModel> viewModelInstance(crossinline create: () -> VM): VM =
    viewModel(factory = object : ViewModelProvider.NewInstanceFactory() {
        override fun <T : ViewModel> create(modelClass: Class<T>): T {
            return create() as T
        }
    })
```

最终仅需要声明实例化 ViewModel 的过程即可，节省了很多代码量

```kotlin
@Composable
fun ChatScreen(
    navController: NavHostController,
    listState: LazyListState,
    chat: Chat
) {
    val chatViewModel1 = viewModelInstance {
        ChatViewModel(chat = chat)
    }
}
```

# 五、OnBackPressedDispatcher

OnBackPressedDispatcher 是 `androidx.activity:activity` 包中提供的一套用于实现拦截返回键事件的机制，在 Jetpack Compose 中也有着应用

以 compose_chat 为例，在主界面 HomeScreen 中提供了一个用于**添加好友和入群**的悬浮弹窗 HomeScreenSheetContent，当悬浮弹窗处于可见状态时，由于 HomeScreen 和 HomeScreenSheetContent 是一体的，当点击返回键时，整个 Activity 就会直接退出

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e335193a241344d6b2a11721fec7bedb~tplv-k3u1fbpfcp-zoom-1.image)

而我想要的效果是：当悬浮弹窗处于可见状态时，点击返回键只会让悬浮弹窗隐藏，在不可见状态下点击返回键才允许直接退出 Activity

此时就需要依靠 OnBackPressedDispatcher 来主动拦截返回事件了，Compose 包中对应的就是 BackHandler 方法

当 HomeScreenSheetContent 处于可见状态，即 `modalBottomSheetState.isVisible` 为 true 时就会对返回事件进行拦截，在 `onBack` 回调里将 HomeScreenSheetContent 一步步置为了 Hidden 状态，当 HomeScreenSheetContent 被隐藏后也会同时取消拦截

```kotlin
@Composable
fun HomeScreenSheetContent(
    homeScreenSheetContentState: HomeScreenSheetContentState,
) {
    val coroutineScope = rememberCoroutineScope()
    
    fun expandSheetContent(targetValue: ModalBottomSheetValue) {
        coroutineScope.launch {
            homeScreenSheetContentState.modalBottomSheetState.animateTo(targetValue = targetValue)
        }
    }

    BackHandler(enabled = homeScreenSheetContentState.modalBottomSheetState.isVisible, onBack = {
        when (homeScreenSheetContentState.modalBottomSheetState.currentValue) {
            ModalBottomSheetValue.Hidden -> {

            }
            ModalBottomSheetValue.Expanded -> {
                expandSheetContent(targetValue = ModalBottomSheetValue.HalfExpanded)
            }
            ModalBottomSheetValue.HalfExpanded -> {
                expandSheetContent(targetValue = ModalBottomSheetValue.Hidden)
            }
        }
    })
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8ab7ba234104373b3787efb1d2d348d~tplv-k3u1fbpfcp-zoom-1.image)

从源码可以看出来 BackHandler 方法其实就是对 OnBackPressedDispatcherOwner 注册了一个回调通知，在 enabled 为 true 的时候就负责调用我们传入的 `onBack` 方法

```kotlin
@Composable
public fun BackHandler(enabled: Boolean = true, onBack: () -> Unit) {
    // Safely update the current `onBack` lambda when a new one is provided
    val currentOnBack by rememberUpdatedState(onBack)
    // Remember in Composition a back callback that calls the `onBack` lambda
    val backCallback = remember {
        object : OnBackPressedCallback(enabled) {
            override fun handleOnBackPressed() {
                currentOnBack()
            }
        }
    }
    // On every successful composition, update the callback with the `enabled` value
    SideEffect {
        backCallback.isEnabled = enabled
    }
    val backDispatcher = checkNotNull(LocalOnBackPressedDispatcherOwner.current) {
        "No OnBackPressedDispatcherOwner was provided via LocalOnBackPressedDispatcherOwner"
    }.onBackPressedDispatcher
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner, backDispatcher) {
        // Add callback to the backDispatcher
        backDispatcher.addCallback(lifecycleOwner, backCallback)
        // When the effect leaves the Composition, remove the callback
        onDispose {
            backCallback.remove()
        }
    }
}
```

# 六、Effect API

在很多讲解关于程序设计最佳实践的文章或者书籍里，都会推荐一个编码原则：**尽可能使用 val、不可变对象及纯函数来设计程序**。这个原则在 Jetpack Compose 中也同样需要遵守，因为一个合格的可组合函数就应该属于**纯函数**，幂等且没有副作用

何谓纯函数？假如一个函数使用相同的入参参数重复执行时，总是能获得相同的运行结果，且运行结果不会影响任何外部状态，也不用担心重复执行会对外部造成改变，那么这个函数就称为纯函数

纯函数不具备副作用，具有引用透明性。副作用就是指修改了某处的某些东西，比如：

- 引用或修改了外部变量的值
- 执行了 IO 操作，比如读写了 SharedPreferences
- 执行了 UI 操作，比如修改了一个按钮的可操作状态

以下例子就不属于纯函数，由于受到外部变量的影响，使用相同入参参数多次执行 count 函数的结果并不全部相同，且每次执行都会影响到外部环境（使用到了 println 函数），这些都属于副作用

```kotlin
var count = 1

fun count(x: Int): Int {
    count += x
    println(count)
    return count
}
```

使用 Jetpack Compose 时需要注意：

- 可组合函数可以按任何顺序执行
- 可组合函数可以并行执行
- 重组会跳过尽可能多的可组合函数和 lambda
- 重组是乐观的操作，可能会被取消
- 可组合函数可能会像动画的每一帧一样非常频繁地运行

但在某些情况下，可组合函数无法做到完全无副作用。例如，我们在切换应用主题的时候希望系统状态栏也能跟着一起改变背景色，此时就说可组合函数产生了副作用。为了处理这种情况，Jetpack Compose 提供了 Effect API，以便以可预测的方式执行这些附带效应，避免附带效应以不安全的形式被过度执行

下面就来介绍一些比较常见的 Effect API

## LaunchedEffect

LaunchedEffect 用于在可组合函数中安全地调用协程。当 LaunchedEffect 进入组合时，它会通过 CoroutineScope.launch 启动一个协程，并将开发者声明的代码块作为参数传递给该协程进行执行。当 LaunchedEffect 退出组合时，启动的协程也会被取消，从而保证了安全性。同时 LaunchedEffect 也支持和多个 key 进行绑定，当 key 发生变化时就会取消现有协程并重新启动

compose_chat 在多个地方中都使用到了 LaunchedEffect，例如在 HomeActivity 中就通过 LaunchedEffect 启动了一个协程用来监测当前账号的状态 serverConnectState，当账号被挤下线时就会自动跳转到登陆页面

```kotlin
class HomeActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        WindowCompat.setDecorFitsSystemWindows(window, false)
        setContent {
            val appViewModel = viewModel<AppViewModel>()
            val navController = rememberAnimatedNavController()
            LaunchedEffect(key1 = Unit) {
                withContext(Dispatchers.Main) {
                    appViewModel.serverConnectState.collect {
                        when (it) {
                            ServerState.KickedOffline -> {
                                showToast("本账号已在其它客户端登陆，请重新登陆")
                                AccountCache.onUserLogout()
                                navController.navToLogin()
                            }
                            ServerState.Logout -> {
                                navController.navToLogin()
                            }
                            else -> {
                                showToast("Connect State Changed : $it")
                            }
                        }
                    }
                }
            }
        }
    }
    
}
```

## rememberCoroutineScope

由于 LaunchedEffect 是可组合函数，所以我们没法在可组合函数之外来使用它。为了能够在可组合函数之外安全地使用协程，此时就需要用到 `rememberCoroutineScope` 方法了，该方法会返回一个 CoroutineScope，该作用域存在生命周期的限制，当可组合函数退出组合时就会取消所有启动的协程，保证了安全性

例如，compose_chat 在群资料页 GroupProfileScreen 中就使用到了 `rememberCoroutineScope` 方法，`quitGroup` 回调属于一个普通的 lambda 方法，无法直接使用 LaunchedEffect，因此回调中就通过 coroutineScope 来调用 GroupProfileViewModel 中的 suspend 方法

```kotlin
@Composable
fun GroupProfileScreen(
    navController: NavHostController,
    groupId: String
) {
    val groupProfileViewModel = viewModelInstance {
        GroupProfileViewModel(groupId = groupId)
    }
    val coroutineScope = rememberCoroutineScope()
    Scaffold(
        modifier = Modifier
            .fillMaxSize(),
    ) {
        Box {
            GroupProfileScreenTopBar(quitGroup = {
                coroutineScope.launch {
                    when (val result = groupProfileViewModel.quitGroup()) {
                        ActionResult.Success -> {
                            showToast("已退出群聊")
                            navController.navToHomeScreen()
                        }
                        is ActionResult.Failed -> {
                            showToast(result.reason)
                        }
                    }
                }
            })
        }
    }
}
```

## snapshotFlow

snapshotFlow 用于将任意包含返回值的代码块转换为冷 flow，当判断到值发生变化时，flow 就会向收集器发出新值

例如，compose_chat 的侧边栏 HomeScreenDrawer 就通过 snapshotFlow 来自动关闭修改用户资料的悬浮窗 DrawerFrontLayer 。当判断到 HomeScreenDrawer 被关闭了且 DrawerFrontLayer 处于显示状态时，就会主动隐藏 DrawerFrontLayer

```kotlin
@Composable
fun HomeScreenDrawer(homeScreenDrawerState: HomeScreenDrawerState) {
    val drawerState = homeScreenDrawerState.drawerState
    val scaffoldState = rememberBackdropScaffoldState(initialValue = BackdropValue.Revealed)

    LaunchedEffect(key1 = Unit) {
        snapshotFlow {
            !drawerState.isOpen && scaffoldState.isConcealed
        }.collect {
            scaffoldState.reveal()
        }
    }
    
    BackdropScaffold(
        scaffoldState = scaffoldState,
        frontLayerContent = {
            DrawerFrontLayer(
                backdropScaffoldState = scaffoldState,
                homeScreenDrawerState = homeScreenDrawerState
            )
        }
    )
}
```

## rememberUpdatedState

## SideEffect

## DisposableEffect

- rememberUpdatedState。如果希望在效应中捕获某个值，且即使该值发生了变化也不会导致效应重启，可以使用 rememberUpdatedState 来实现
- SideEffect。如果需要与非 Compose 管理的对象共享 Compose 状态，可以使用 SideEffect 来实现，因为每次成功重组时都会调用该可组合项
- DisposableEffect。如果附带效应需要在 key 发生变化或者是可组合项退出组合后进行清理，可以使用 DisposableEffect 来实现

Compose 提供的 BackHandler 方法就同时使用到了以上三者，其意义分别是：

- 外部传入的 onBack 方法仅需要在拦截了返回事件时进行调用即可，效应仅需要拿到最新值即可，即使该值发生了变化，也不影响当前的效应执行过程，因此 就通过 rememberUpdatedState 来捕获 onBack 回调，从而避免去重启效应。而如果不使用 rememberUpdatedState 的话，就需要将 onBack 当做附带效应的 key，这就会导致 onBack 一发生变化就需要重启效应
- 外部在调用 BackHandler 方法时传入的 enabled 参数可能是基于某个逻辑表达式来传入的，随时可能发生变化，而 SideEffect 在每次重组时都会被执行，因此 BackHandler 就通过 SideEffect 来保证 OnBackPressedCallback 能实时指向当前最新的启用状态
- DisposableEffect 提供了 onDispose 回调，当 key 发生变化或者是可组合项退出组合时就会被调用，因此就通过 DisposableEffect 来自动移除对 OnBackPressed 事件的监听。需要注意的是，DisposableEffect 必须添加 onDispose 语句，且必须放在最后面，否则编译器将报错

```kotlin
@Composable
public fun BackHandler(enabled: Boolean = true, onBack: () -> Unit) {
    // Safely update the current `onBack` lambda when a new one is provided
    val currentOnBack by rememberUpdatedState(onBack)
    // Remember in Composition a back callback that calls the `onBack` lambda
    val backCallback = remember {
        object : OnBackPressedCallback(enabled) {
            override fun handleOnBackPressed() {
                currentOnBack()
            }
        }
    }
    // On every successful composition, update the callback with the `enabled` value
    SideEffect {
        backCallback.isEnabled = enabled
    }
    val backDispatcher = checkNotNull(LocalOnBackPressedDispatcherOwner.current) {
        "No OnBackPressedDispatcherOwner was provided via LocalOnBackPressedDispatcherOwner"
    }.onBackPressedDispatcher
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner, backDispatcher) {
        // Add callback to the backDispatcher
        backDispatcher.addCallback(lifecycleOwner, backCallback)
        // When the effect leaves the Composition, remove the callback
        onDispose {
            backCallback.remove()
        }
    }
}
```

# 七、Coroutines

Kotlin 协程在 compose_chat 中用得也比较多，比较重要的一个点就是用于发送消息时

当用于点击发送消息后，就会构造出一条文本消息 TextMessage，而 TextMessage 所处的状态先后有三种可能：发送中、发送成功、发送失败。为了避免上层业务逻辑需要通过多次 Callback 的方式来异步拿到底层 IM SDK 的处理结果，compose_chat 选择通过 Channel 来同步地完成整个过程

IM SDK 的具体消息发送逻辑对应的是 MessageProvider，在正式发送消息前会先将 text 构建为一条 Sending 状态的 TextMessage，上层视图就可以先拿到该消息用于 UI 展示。然后再来正式向 IM SDK 发送消息，并将消息的最终状态（成功或者失败）再通过 Channel 回调出去

```kotlin
class MessageProvider : IMessageProvider, Converters {

    override suspend fun send(chat: Chat, text: String, channel: Channel<Message>) {
        val id = chat.id
        val messageId = RandomUtils.generateMessageId()
            msgId = messageId, state = MessageState.Sending,
            timestamp = V2TIMManager.getInstance().serverTime, msg = text,
            sender = AppConst.personProfile.value,
        )
        channel.send(sendingMessage)
        val callback = object : V2TIMValueCallback<V2TIMMessage> {
            override fun onSuccess(t: V2TIMMessage) {
                coroutineScope.launch {
                    val msg = convertMessage(t) as? TextMessage
                    //向 channel 发送最终结果
                    if (msg == null) {
                        channel.send(sendingMessage.copy(state = MessageState.SendFailed))
                    } else {
                        channel.send(msg)
                    }
                    //关闭 channel，代表回调结束
                    channel.close()
                }
            }

            override fun onError(code: Int, desc: String?) {
                coroutineScope.launch {
                    //向 channel 发送最终结果
                    channel.send(
                        sendingMessage.copy(state = MessageState.SendFailed).apply {
                            tag = "code: $code desc: $desc"
                        })
                    //关闭 channel，代表回调结束
                    channel.close()
                }
            }
        }
        when (chat) {
            is Chat.C2C -> {
                V2TIMManager.getInstance().sendC2CTextMessage(text, id, callback)
            }
            is Chat.Group -> {
                V2TIMManager.getInstance()
                    .sendGroupTextMessage(text, id, V2TIMMessage.V2TIM_PRIORITY_HIGH, callback)
            }
        }
    }

}
```

ChatViewModel 则仅需要向 MessageProvider 传递 text 即可，后续 TextMessage 的构建过程和发送逻辑都无需关心，仅需同步地向 messageChannel 取值进行 UI 状态刷新即可

```kotlin
class ChatViewModel(private val chat: Chat) : ViewModel() {

    fun sendMessage(text: String) {
        viewModelScope.launch {
            val messageChannel = Channel<Message>()
            launch {
                ComposeChat.messageProvider.send(
                    chat = chat,
                    text = text,
                    channel = messageChannel
                )
            }
            var sendingMessage: Message? = null
            for (message in messageChannel) {
                when (message.state) {
                    MessageState.Sending -> {
                        sendingMessage = message
                        attachNewMessage(newMessage = message, mushScrollToBottom = true)
                    }
                    MessageState.Completed, MessageState.SendFailed -> {
                        val sending = sendingMessage ?: return@launch
                        val sendingMessageIndex =
                            allMessage.indexOfFirst { it.msgId == sending.msgId }
                        if (sendingMessageIndex > -1) {
                            allMessage[sendingMessageIndex] = message
                            refreshViewState()
                        }
                        if (message.state == MessageState.SendFailed) {
                            (message.tag as? String)?.let {
                                showToast(it)
                            }
                        }
                    }
                }
            }
        }
    }

}
```

关于 Kotlin 协程的入门教程可以看我的另一篇文章：[一文快速入门 Kotlin 协程](https://juejin.cn/post/6908271959381901325)

# 八、Coil

对于一个应用来说，加载图片的功能必不可少，Glide 也应该是大部分开发者接触的最多的图片加载框架了，但对于 Jetpack Compose 来说 Glide 就不适用了，Glide 似乎暂时也没有支持 Jetpack Compose 的打算：[Jetpack Compose Support · Issue #4459 · bumptech/glide (github.com)](https://github.com/bumptech/glide/issues/4459)

为了在 Jetpack Compose 中能够加载网络图片，并支持必要的内存缓存和磁盘缓存功能，我们可以选择另外一个图片加载框架：[Coil](https://github.com/coil-kt/coil)

Coil 能实现 Glide 的所有功能，并且也同时支持 Jetpack Compose。Coil 相对 Glide 来说还比较“新”，1.0 正式版是在 2020 年 10 月份发布的，如果你的项目中已经大面积使用到了 Jetpack、Kotlin Coroutines、OkHttp 的话，那么 Coil 会更加契合你的项目

对 Coil 和 Glide 做下简单的对比：

1. 实现语言
   - Glide 全盘使用 Java 语言来实现，对于 Java 和 Kotlin 语言的友好程度差不多
   - Coil 全盘使用 Kotlin 语言来实现，为 ImageView 声明了多个用于加载图片的扩展函数，对 Kotlin 语言的友好程度会更高很多
2. 网络请求
   - Glide 默认是使用 HttpURLConnection，但也提供了更换网络请求实现途径的入口
   - Coil 默认是使用 OkHttp，但也提供了更换网络请求实现途径的入口
3. 生命周期监听
   - Glide 通过向 Activity 或者 Fragment 注入一个无 UI 界面的 Fragment 来实现监听
   - Coil 直接通过 Lifecycle 来实现监听
4. 内存缓存
   - Glide 的内存缓存分为 ActiveResources 和 MemoryCache 两级
   - Coil 的内存缓存分为 WeakMemoryCache 和 StrongMemoryCache 两级，本质上和 Glide 一样
5. 磁盘缓存
   - Glide 在加载到图片后通过 DiskLruCache 来进行磁盘缓存，且提供了**是否缓存、是否缓存原始图片、是否缓存转换过后的图片**等多个选择
   - Coil 通过 OkHttp 的网络请求缓存机制来实现磁盘缓存，且磁盘缓存只对通过网络请求加载到的原始图片生效，不缓存其它来源的图片和转换过后的图片
6. 网络缓存
   - Glide 不存在这个概念
   - Coil 相比 Glide 多出了一层网络缓存，可用于实现**不进行网络加载，而是强制使用本地缓存**（当然，如果本地缓存不存在的话就会报错）
7. 线程框架
   - Glide 使用原生的 ThreadPoolExecutor 来完成后台任务，通过 Handler 来实现线程切换
   - Coil 使用 Coroutines 来完成后台任务及线程切换

使用 Coil 来加载图片也十分简单，仅需要传入必要的 data 即可，该 data 的类型可以是 HttpUrl、Uri、File 等多种类型

```kotlin
@Composable
fun CoilImage(
    modifier: Modifier = Modifier,
    data: Any,
    contentScale: ContentScale = ContentScale.Crop,
    builder: ImageRequest.Builder.() -> Unit = {},
) {
    val imagePainter = rememberImagePainter(data = data, builder = builder)
    Image(
        modifier = modifier,
        painter = imagePainter,
        contentDescription = null,
        contentScale = contentScale,
    )
}
```

关于 Coil 的详细源码介绍可以看我的另一篇文章，当时 Coil 刚发布 1.0 正式版本不久我就写了这篇文章：[Android 主流开源库源码分析（十三）- 可能是全网第一篇 Coil 源码分析文章](https://juejin.cn/post/6897872882051842061)

# 九、结尾

关于 Jetpack Compose 和 compose_chat 的介绍就到这里了，先后看完我两篇文章的话，读者应该都能快速上手开发了

最后再放一下 compose_chat 的 GitHub 地址：[用 Jetpack Compose 实现一个 IM APP](https://github.com/leavesczy/compose_chat)

Apk 下载尝鲜：[用 Jetpack Compose 实现一个 IM APP](https://github.com/leavesczy/compose_chat/releases)

腾讯云 IM SDK 免费版最多只能注册一百个账号，1.0 版本的 compose_chat 用户数已经满了，因此 2.0 版本我更换了新的 AppId。如果发现无法注册，读者可以使用以下几个我预先注册好的账号，但多设备同时登陆的话会互相挤掉线 ~~

- Google
- Android
- Jetpack
- Compose
- Flutter
- Kotlin
- Java
- Dart
- ViewModel
- LiveData