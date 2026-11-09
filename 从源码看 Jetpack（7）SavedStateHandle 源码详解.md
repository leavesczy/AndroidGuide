> Google Jetpack 自从推出以后，极大地改变了 Android 开发者们的开发模式，并降低了开发难度。这也要求我们对当中一些子组件的实现原理具有一定的了解，所以我就打算来写一系列 Jetpack 源码解析的文章，希望对你有所帮助

我们知道，Activity 意外销毁的情况可以分为两种：

1. 由于屏幕旋转等配置更改的原因导致 Activity 被销毁
2. 由于系统资源限制导致 Activity 被销毁

对于这两种情况，我们当然希望 Activity 重建后之前 **加载的数据** 以及 **用户状态** 都能够得到恢复，每种情况目前有着不同的恢复方法

- 对于第一种情况，Jetpack 提供了 ViewModel 来解决这个问题。ViewModel 可以在配置更改后继续存留，适合用于在内存中存储比较复杂或者量比较大的数据，例如，用 RecyclerView 加载的多个列表项对应的 Data。**但当第二种情况发生时 ViewModel 是无法被保留下来的，Activity 重建后也只会得到一个新的 ViewModel 实例，并且之前已经加载的数据也会丢失**。关于 ViewModel 的源码详解可以看我的另一篇文章：[从源码看 Jetpack（6）- ViewModel 源码详解](https://juejin.im/post/6873356946896846856)
- 对于第二种情况，需要依赖于 Activity 原生提供的数据保存及恢复机制，即依赖以下两类方法来实现数据保存和数据恢复
  - onSaveInstanceState(Bundle)。通过向 Bundle 插入键值对来保存数据，数据在上述两种情况发生时都会被保留下来，但该方法也有着存储容量和存取效率的限制。Bundle 有着容量限制，不适合用于存储大量数据，而且是通过将数据序列化到磁盘来进行保存的，所以如果要保存的数据很复杂或者很大，序列化就会消耗大量的内存和时间。因此 `onSaveInstanceState` 方法仅适合用于存储少量简单类型的数据
  - onCreate(Bundle) 或者 onRestoreInstanceState(Bundle)。用于从 Bundle 中取出数据进行状态恢复

Google 官方也对这两种情况进行了对比：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00b6ddc7920f4a84bbf2c5ec0e0175e8~tplv-k3u1fbpfcp-zoom-1.image)

对于第二种情况，数据的保存和恢复流程被限制在了 Activity 的特定方法里，我们无法直接在 ViewModel 中决定哪些数据需要被保留，也无法直接拿到恢复后的数据，使得整个重建流程和 ViewModel 分裂开了

为了解决这个问题，Jetpack 提供了 SavedStateHandle 这么一个组件，可以看做是对 ViewModel 的功能扩展，使得开发者可以直接在 ViewModel 中直接操作整个数据的重建过程，本文要介绍的就是 SavedStateHandle 的使用方式和实现原理

本文内容基于以下版本来进行讲解

```kotlin
compileSdkVersion 30
implementation 'androidx.appcompat:appcompat:1.3.0-beta01'
implementation "androidx.lifecycle:lifecycle-viewmodel-savedstate:2.3.0"
implementation "androidx.savedstate:savedstate:1.1.0"
```
# 一、使用示例

SavedStateHandle 的引入使得开发者无需直接使用 `onSaveInstanceState(Bundle)` 等方法来完成数据的保存和重建，而只需要在 ViewModel 里来完成即可

其使用基本流程可以总结为：

- 将 SavedStateHandle 作为 ViewModel 的构造参数
- ViewModel 内部通过 `SavedStateHandle.getLiveData` 方法来生成一个 LiveData 对象，LiveData 中的数据即我们想要持久化保存的数据。如果是全新启动 Activity，LiveData 中保存的值为 null；如果是重建后的 Activity，LiveData 中保存的值则为重建前其自身的值
- 传给 `getLiveData` 方法的 String 参数是一个唯一 Key，最终保存到 Bundle 中的键值对就以该值作为 Key，以 LiveData 的值作为 value

```kotlin
class SavedStateViewModel(savedStateHandle: SavedStateHandle) : ViewModel() {

    companion object {

        private const val KEY_NAME = "keyName"

    }

    val nameLiveData = savedStateHandle.getLiveData<String>(KEY_NAME)

    val blogLiveData = MutableLiveData<String>()

}

class MainActivity : AppCompatActivity() {

    private val savedStateViewModel by lazy {
        ViewModelProvider(this).get(SavedStateViewModel::class.java)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        log("savedStateViewModel: $savedStateViewModel")
        log("savedStateViewModel.name: ${savedStateViewModel.nameLiveData.value}")
        log("savedStateViewModel.blog: ${savedStateViewModel.blogLiveData.value}")
        log("onCreate")
        btn_test.setOnClickListener {
            savedStateViewModel.nameLiveData.value = "业志陈"
            savedStateViewModel.blogLiveData.value = "https://juejin.cn/user/923245496518439/posts"
        }
    }

    private fun log(log: String) {
        Log.e("MainActivity", log)
    }

}
```
打开开发者模式中"不保留活动"的选项，以此来模拟 Activity 由于系统内存不足被销毁的情况

当 MainActivity 第一次启动时，两个 LiveData 中的值都是为 null

```kotlin
E/MainActivity: savedStateViewModel: github.leavesc.demo.SavedStateViewModel@df3fa77
E/MainActivity: savedStateViewModel.name: null
E/MainActivity: savedStateViewModel.blog: null
E/MainActivity: onCreate
```
点击按钮为这两个 LiveData 进行赋值，按 Home 键退出应用，此时 MainActivity 在后台就会被销毁。重新打开应用，此时就可以看到 ViewModel 其实已经是新的一个实例了，但通过 SavedStateHandle 构建的 `nameLiveData` 中还保留着之前的值，而 `blogLiveData` 中就还是默认值 null

```kotlin
E/MainActivity: savedStateViewModel: github.leavesc.demo.SavedStateViewModel@f5fa30c
E/MainActivity: savedStateViewModel.name: 业志陈
E/MainActivity: savedStateViewModel.blog: null
E/MainActivity: onCreate
```
以上例子就展示了 SavedStateHandle 在 Activity 被意外杀死时也可以保留数据的能力，使得我们可以直接在 ViewModel 里完成整个数据的重建逻辑。**此外，再强调一次，如果 Activity 是由于系统资源限制导致被销毁重建的话，ViewModel 实例是不会被保留下来的，所以在以上例子中第二次得到的是一个新的 ViewModel 实例，此时只能依赖 Activity 原生的数据恢复机制来保存少量简单的数据**

而 SavedStateHandle 其实也是通过封装 `onSaveInstanceState(Bundle)` 和 `onCreate(Bundle)` 两个方法来实现的，SavedStateHandle 会在 Activity 被销毁时通过 `onSaveInstanceState(Bundle)` 方法将数据保存在 Bundle 中，在重建时又将数据从 `onCreate(Bundle?)` 中取出，开发者只负责向 SavedStateHandle 存取数据即可，并不需要和 Activity 直接做交互，从而简化了整个开发流程

SavedStateHandle 整个数据重建流程主要涉及以下几个类和接口：

1. SavedStateRegistryOwner
2. SavedStateRegistryController
3. SavedStateRegistry
4. SavedStateHandle

下面就来详细介绍下其内部具体的实现原理

# 二、SavedStateRegistryOwner

SavedStateRegistryOwner 是一个接口，用于标记其实现类（Activity/Fragment）拥有着数据重建的能力。`androidx.activity.ComponentActivity` 和 `androidx.fragment.app.Fragment` 就继承了 SavedStateRegistryOwner 接口，相当于所有子类都拥有一个 SavedStateRegistry 对象

```java
public interface SavedStateRegistryOwner extends LifecycleOwner {
    @NonNull
    SavedStateRegistry getSavedStateRegistry();
}
```
# 三、SavedStateRegistryController

ComponentActivity 将数据的**保存和恢复**逻辑都转发给了 SavedStateRegistryController 来处理，在 `onSaveInstanceState` 方法里通过调用 `performSave` 方法来保存数据，在 `onCreate` 方法里通过调用 `performRestore` 方法来恢复数据

SavedStateRegistryController 又会将逻辑转交由 SavedStateRegistry 的同名方法来完成

```java
public class ComponentActivity extends androidx.core.app.ComponentActivity
        implements ContextAware,
                LifecycleOwner,
                ViewModelStoreOwner,
                HasDefaultViewModelProviderFactory,
                SavedStateRegistryOwner,
                OnBackPressedDispatcherOwner,
                ActivityResultRegistryOwner,
                ActivityResultCaller {

    final SavedStateRegistryController mSavedStateRegistryController =
            SavedStateRegistryController.create(this);

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        // Restore the Saved State first so that it is available to
        // OnContextAvailableListener instances
        // 恢复数据
        mSavedStateRegistryController.performRestore(savedInstanceState);
        ···
        super.onCreate(savedInstanceState);
        ···
    }

    @CallSuper
    @Override
    protected void onSaveInstanceState(@NonNull Bundle outState) {
        ···
        super.onSaveInstanceState(outState);
        // 保存数据
        mSavedStateRegistryController.performSave(outState);
        ···
    }

    @NonNull
    @Override
    public final SavedStateRegistry getSavedStateRegistry() {
        return mSavedStateRegistryController.getSavedStateRegistry();
    }
}

public final class SavedStateRegistryController {
    private final SavedStateRegistryOwner mOwner;
    private final SavedStateRegistry mRegistry;

    private SavedStateRegistryController(SavedStateRegistryOwner owner) {
        mOwner = owner;
        mRegistry = new SavedStateRegistry();
    }

    @NonNull
    public SavedStateRegistry getSavedStateRegistry() {
        return mRegistry;
    }

    @MainThread
    public void performRestore(@Nullable Bundle savedState) {
        Lifecycle lifecycle = mOwner.getLifecycle();
        // 必须在 Activity 的 onCreate 方法调用结束前进行数据恢复
        if (lifecycle.getCurrentState() != Lifecycle.State.INITIALIZED) {
            throw new IllegalStateException(
                    "Restarter must be created only during " + "owner's initialization stage");
        }
        lifecycle.addObserver(new Recreator(mOwner));
        mRegistry.performRestore(lifecycle, savedState);
    }

    @MainThread
    public void performSave(@NonNull Bundle outBundle) {
        mRegistry.performSave(outBundle);
    }

    @NonNull
    public static SavedStateRegistryController create(@NonNull SavedStateRegistryOwner owner) {
        return new SavedStateRegistryController(owner);
    }
}
```
# 四、SavedStateRegistry

## 拿数据的入口

SavedStateRegistry 是实际进行保存和恢复数据的地方，那么很自然地，SavedStateRegistry 就需要有一个入口可以从外部（例如，ViewModel ）取数据，这个入口就是 `registerSavedStateProvider` 方法

外部需要实现 SavedStateProvider 接口，在 `saveState()` 返回想要保存的数据，然后调用 `registerSavedStateProvider` 方法将 SavedStateProvider 对象提交给 SavedStateRegistry。因为并不是所有 Activity 被销毁的情况都需要进行数据的保存和恢复操作，例如用户按返回键退出 Activity 的时候就不需要保存数据，所以 `saveState()` 方法仅会在需要的时候才会被调用

```java
private SafeIterableMap<String, SavedStateProvider> mComponents = new SafeIterableMap<>();

// 外部通过一个唯一标识 key 来和要保存的数据 Bundle 相对应，后续也通过这个 key 来恢复数据
@MainThread
public void registerSavedStateProvider(
        @NonNull String key, @NonNull SavedStateProvider provider) {
    SavedStateProvider previous = mComponents.putIfAbsent(key, provider);
    if (previous != null) {
        throw new IllegalArgumentException(
                "SavedStateProvider with the given key is" + " already registered");
    }
}

public interface SavedStateProvider {
    @NonNull
    Bundle saveState();
}
```
## 保存数据

既然已经指定了拿数据的入口，那么就来看下 `performSave` 方法是如何保存数据的，其主要逻辑是：

1. 如果在上一次重建 Activity 时保存下来的数据还未消费完，那么再次重建 Activity 时就将未消费完的数据也保存给 components
2. 遍历 `mComponents` ，将所有需要保存的数据都保存到 components 中
3. 将 components 保存到 `onSaveInstanceState` 方法传来的 Bundle 对象里，从而完成数据的保存操作

```java
private static final String SAVED_COMPONENTS_KEY =
        "androidx.lifecycle.BundlableSavedStateRegistry.key";

@Nullable private Bundle mRestoredState;

@MainThread
void performSave(@NonNull Bundle outBundle) {
    Bundle components = new Bundle();
    if (mRestoredState != null) {
        // 步骤1
        components.putAll(mRestoredState);
    }
    // 步骤2
    for (Iterator<Map.Entry<String, SavedStateProvider>> it =
                    mComponents.iteratorWithAdditions();
            it.hasNext(); ) {
        Map.Entry<String, SavedStateProvider> entry1 = it.next();
        components.putBundle(entry1.getKey(), entry1.getValue().saveState());
    }
    // 步骤3
    outBundle.putBundle(SAVED_COMPONENTS_KEY, components);
}
```
## 恢复数据

再来看下 `performRestore` 是如何恢复数据的，其主要逻辑是：

1. 拿到 `performSave` 方法保存到 Bundle 里的数据，将数据存到 `mRestoredState` 中
2. 通过监听 Lifecycle 来确定当前是否处于可以恢复数据的生命周期阶段，用一个布尔变量 `mAllowingSavingState` 来标记

```java
private boolean mRestored;

boolean mAllowingSavingState = true;

@Nullable private Bundle mRestoredState;

@MainThread
void performRestore(@NonNull Lifecycle lifecycle, @Nullable Bundle savedState) {
    if (mRestored) {
        // 不应该重复恢复数据
        throw new IllegalStateException("SavedStateRegistry was already restored.");
    }
    if (savedState != null) {
        // 步骤1
        mRestoredState = savedState.getBundle(SAVED_COMPONENTS_KEY);
    }

    // 步骤2
    lifecycle.addObserver(
            new GenericLifecycleObserver() {
                @Override
                public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
                    if (event == Lifecycle.Event.ON_START) {
                        mAllowingSavingState = true;
                    } else if (event == Lifecycle.Event.ON_STOP) {
                        mAllowingSavingState = false;
                    }
                }
            });

    mRestored = true;
}
```
## 消费数据

数据被恢复了并不意味着 Activity 已经恢复到了被销毁前的状态，被恢复的数据还存在 Bundle 里，此时还需要开发者通过取**键值对**的方式来消费数据，将**用户数据或者 UI 状态**恢复到销毁前的状态

消费数据的入口就是 `consumeRestoredStateForKey` 方法，外部通过使用和传给 `registerSavedStateProvider` 方法时一样的 key 来取数据，并在取了之后将数据从 `mRestoredState` 中移除。如果所有数据都被消费了的话，那么就将 `mRestoredState` 置为 null，标记着所有数据都已经被消费完了

```java
@MainThread
@Nullable
public Bundle consumeRestoredStateForKey(@NonNull String key) {
    if (!mRestored) {
        throw new IllegalStateException(
                "You can consumeRestoredStateForKey "
                        + "only after super.onCreate of corresponding component");
    }
    if (mRestoredState != null) {
        Bundle result = mRestoredState.getBundle(key);
        mRestoredState.remove(key);
        if (mRestoredState.isEmpty()) {
            mRestoredState = null;
        }
        return result;
    }
    return null;
}
```
## 联系

可以看到，SavedStateRegistry 已经代理了 Activity 的 `onCreate(Bundle)` 和 `onSaveInstanceState(Bundle)` 这两个方法，已经串联起了整个流程，后面我们只需要看是谁向 SavedStateRegistry 提供了数据，又是被谁消费了数据即可，即主要就看是谁调用了 `registerSavedStateProvider` 和 `consumeRestoredStateForKey` 这两个方法

# 五、SavedStateHandle

SavedStateHandle 包含两个构造函数，`initialState` 中保存的即是 Activity 重建时保留下来的的键值对数据，`mRegular` 中保存的即是最终要持久化保存的键值对数据。在最开始展示的例子里，SavedStateHandle 是作为 ViewModel 的构造参数而存在的，而我们自己并没有来显式调用其构造函数，SavedStateHandle 的初始化都交由组件内部来自动完成了。如果最终调用的是有参构造函数，则代表着此次初始化是 Activity 被销毁重建的情况，如果调用的是无参构造函数，则代表着此次初始化是 Activity 全新启动的情况

```java
public final class SavedStateHandle {

    final Map<String, Object> mRegular;

    public SavedStateHandle(@NonNull Map<String, Object> initialState) {
        mRegular = new HashMap<>(initialState);
    }

    public SavedStateHandle() {
        mRegular = new HashMap<>();
    }

    @MainThread
    public boolean contains(@NonNull String key) {
        return mRegular.containsKey(key);
    }

    @SuppressWarnings({"unchecked", "TypeParameterUnusedInFormals"})
    @MainThread
    @Nullable
    public <T> T get(@NonNull String key) {
        return (T) mRegular.get(key);
    }

    ···

}
```
`getLiveData` 方法会返回一个和 `key` 还有 `mRegular` 关联的 LiveData 对象，LiveData 对象的初始默认值会从 `mRegular` 和 `initialValue` 两个之间选取，每次生成的 LiveData 对象也都会被保存在 `mLiveDatas` 中，以便后续复用

```java
private final Map<String, SavingStateLiveData<?>> mLiveDatas = new HashMap<>();

@MainThread
@NonNull
public <T> MutableLiveData<T> getLiveData(@NonNull String key) {
    return getLiveDataInternal(key, false, null);
}

@MainThread
@NonNull
public <T> MutableLiveData<T> getLiveData(
        @NonNull String key, @SuppressLint("UnknownNullness") T initialValue) {
    return getLiveDataInternal(key, true, initialValue);
}

@SuppressWarnings("unchecked")
@NonNull
private <T> MutableLiveData<T> getLiveDataInternal(
        @NonNull String key, boolean hasInitialValue, @Nullable T initialValue) {
    MutableLiveData<T> liveData = (MutableLiveData<T>) mLiveDatas.get(key);
    if (liveData != null) {
        return liveData;
    }
    SavingStateLiveData<T> mutableLd;
    // double hashing but null is valid value
    if (mRegular.containsKey(key)) {
        mutableLd = new SavingStateLiveData<>(this, key, (T) mRegular.get(key));
    } else if (hasInitialValue) {
        mutableLd = new SavingStateLiveData<>(this, key, initialValue);
    } else {
        mutableLd = new SavingStateLiveData<>(this, key);
    }
    mLiveDatas.put(key, mutableLd);
    return mutableLd;
}
```
当外部对 LiveData 进行值更新操作时，SavedStateHandle 需要拿到最新值，因为最终持久化保存的肯定也需要是最新值。所以 `getLiveDataInternal` 方法返回的 SavingStateLiveData 对象就会在 `setValue` 方法被调用后，同步更新 `mRegular` 中的键值对数据

```java
static class SavingStateLiveData<T> extends MutableLiveData<T> {
    private String mKey;
    private SavedStateHandle mHandle;

    SavingStateLiveData(SavedStateHandle handle, String key, T value) {
        super(value);
        mKey = key;
        mHandle = handle;
    }

    SavingStateLiveData(SavedStateHandle handle, String key) {
        super();
        mKey = key;
        mHandle = handle;
    }

    @Override
    public void setValue(T value) {
        if (mHandle != null) {
            mHandle.mRegular.put(mKey, value);
        }
        super.setValue(value);
    }

    void detach() {
        mHandle = null;
    }
}
```
SavedStateHandle 也提供了另外一种声明需要缓存的键值对数据的方法。SavedStateHandle 开放了一个 `setSavedStateProvider` 方法交由外部来传入 SavedStateProvider 对象，外部负责实现 `saveState()` 方法来返回想要持久化缓存的 Bundle 对象，由 SavedStateHandle 来负责调用该方法

```java
public interface SavedStateProvider {
    @NonNull
    Bundle saveState();
}

final Map<String, SavedStateProvider> mSavedStateProviders = new HashMap<>();

@MainThread
public void setSavedStateProvider(@NonNull String key, @NonNull SavedStateProvider provider) {
    mSavedStateProviders.put(key, provider);
}
```
我们在 ViewModel 层通过向 `mRegular` 存取值，就是在决定一旦 Activity 被意外销毁重建时需要恢复的数据有哪些，所以最终 `mRegular` 还是要被存到 Bundle 里，这个过程就由 `mSavedStateProvider` 来实现，其内部会遍历 `mSavedStateProviders` 和 `mRegular`，将 key 和 value 按照对应关系顺序存入两个不同的 ArrayList 里，最后将两个 ArrayList 都保存到 Bundle 里

```java
private final SavedStateProvider mSavedStateProvider =
        new SavedStateProvider() {
            @SuppressWarnings("unchecked")
            @NonNull
            @Override
            public Bundle saveState() {
                // Get the saved state from each SavedStateProvider registered with this
                // SavedStateHandle, iterating through a copy to avoid re-entrance
                Map<String, SavedStateProvider> map = new HashMap<>(mSavedStateProviders);
                for (Map.Entry<String, SavedStateProvider> entry : map.entrySet()) {
                    Bundle savedState = entry.getValue().saveState();
                    set(entry.getKey(), savedState);
                }
                // Convert the Map of current values into a Bundle
                Set<String> keySet = mRegular.keySet();
                ArrayList keys = new ArrayList(keySet.size());
                ArrayList value = new ArrayList(keys.size());
                for (String key : keySet) {
                    keys.add(key);
                    value.add(mRegular.get(key));
                }

                Bundle res = new Bundle();
                // "parcelable" arraylists - lol
                res.putParcelableArrayList("keys", keys);
                res.putParcelableArrayList("values", value);
                return res;
            }
        };

@NonNull
SavedStateProvider savedStateProvider() {
    return mSavedStateProvider;
}
```
# 六、关联上

这里来做个小小的总结

- SavedStateHandle 的构造函数由组件内部来自动调用，外部不需要来手动调用。如果调用的是无参构造函数，则对应的是 Activity 全新启动的情况，此时就没有需要恢复的数据。如果调用的是有参构造函数，则对应的是 Activity 重建启动的情况，传入的 Map 保存的即 Activity 重建时被保留下来的键值对数据
- 所以说，SavedStateHandle 本身在初始化时已经包含了所有被缓存下来的数据（如果有的话），在使用过程中我们也会不断更新该键值对，当后续 Activity 被意外销毁时，外部就又通过 mSavedStateProvider 拿到所有需要缓存的键值对数据，mSavedStateProvider 的 saveState() 方法就会将所有需要持久化保留的数据都打包成一个 Bundle 对象并返回
- ViewModel 中想要进行持久化保存的数据需要通过 `savedStateHandle.getLiveData` 的方式来进行取值和赋值。当 Activity 第一次被启动时，LiveData 中肯定也不包含初始值，在后续过程中我们才会向其赋值。当 Activity 被销毁重建时，此时获取到的 LiveData 才会拥有初始值，即 Activity 第一次启动过程中向 LiveData 赋予的值都会被保留到此次
- SavedStateRegistry 已经代理了 Activity 的 `onCreate(Bundle)` 和 `onSaveInstanceState(Bundle)` 这两个方法，已经串联起了整个流程，后面我们只需要看是谁向 SavedStateRegistry 提供了数据，又是被谁消费了数据即可，即主要就看是谁调用了 `registerSavedStateProvider` 和 `consumeRestoredStateForKey` 这两个方法
- SavedStateHandle 的 `mSavedStateProvider` 最终需要提交给 SavedStateRegistry 的 `registerSavedStateProvider` 方法，由 SavedStateRegistry 来取出 SavedStateHandle 中所有需要保留的键值对数据。而 SavedStateRegistry 的 `consumeRestoredStateForKey` 方法返回的 Bundle 数据，最终又需要被解包为一个 Map 对象，该 Map 对象就作为构造参数来初始化 SavedStateHandle

可以看出来，SavedStateHandle 和 SavedStateRegistry 的职责点各不相同。SavedStateHandle 负责接收 Activity 重建时缓存的数据，并将缓存的数据以 LiveData 的方式暴露给开发者，也向外部提供了获取最新键值对的入口。SavedStateRegistry 则负责将 Activity 原生的数据缓存机制串联起来，向外部暴露了提交数据和消费数据的入口。SavedStateHandle 缓存的数据就需要提交给 SavedStateRegistry，SavedStateRegistry 缓存的数据最终也需要交由 SavedStateHandle 来消费。所以说，目前还欠缺的只是将 SavedStateHandle 和 SavedStateRegistry 给关联起来，这个联系点就需要看 SavedStateHandle 到底是如何初始化的

看了我上一篇关于 ViewModel 的讲解文章后，读者应该已经知道 ViewModel 的初始化逻辑是需要交由 `ViewModelProvider.Factory` 来进行声明的，而在文章开头给出的例子里我之所以可以不实现任何自定义的 `ViewModelProvider.Factory` ，是因为 SavedStateHandle 已经给出来了一个默认的 Factory 实现类，即 SavedStateViewModelFactory。SavedStateViewModelFactory 会获取到 ComponentActivity 包含的 SavedStateRegistry 对象，以此来初始化 SavedStateHandle。由于这一部分并不是重点内容，本身也没有多复杂，我就不多说了，读者重点了解 SavedStateHandle 和 SavedStateRegistry 两者的职责和关联关系即可