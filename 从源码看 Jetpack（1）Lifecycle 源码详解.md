> Google Jetpack 自从推出以后，极大地改变了 Android 开发者们的开发模式，并降低了开发难度。这也要求我们对当中一些子组件的实现原理具有一定的了解，所以我就打算来写一系列 Jetpack 源码解析的文章，希望对你有所帮助

Lifecycle 是 Jetpack 整个家族体系内最为基础的组件之一，正是因为有了 Lifecycle 的存在，使得如今开发者搭建依赖于生命周期变化的业务逻辑变得简单高效了许多，使得我们可以用一种统一的方式来监听 Activity、Fragment、Service、甚至是 Process 的生命周期变化，且大大减少了业务代码发生内存泄漏和 NPE 的风险

本文就来对 Lifecycle 进行一次全面的源码解读，基于以下版本来进行讲解

```groovy
implementation 'androidx.appcompat:appcompat:1.2.0'
implementation "androidx.lifecycle:lifecycle-common:2.2.0"
implementation "androidx.lifecycle:lifecycle-common-java8:2.2.0"
implementation "androidx.lifecycle:lifecycle-runtime:2.2.0"
```

# 一、Lifecycle

现如今，如果我们想要根据 Activity 的生命周期状态的变化来管理我们的业务逻辑的话，那么可以很方便的使用如下方式来进行监听。以基于回调接口方法的形式来进行事件通知，每当 Activity 的生命周期方法被触发时，该接口的相应同名方法就会在**之前或者之后被调用**

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    lifecycle.addObserver(object : DefaultLifecycleObserver {
        override fun onCreate(owner: LifecycleOwner) {

        }

        override fun onResume(owner: LifecycleOwner) {

        }

        override fun onDestroy(owner: LifecycleOwner) {

        }
    })
}
```

此外还有一种基于 `@OnLifecycleEvent` 注解来进行回调的方式，这种方式不对方法名做要求，但是对方法的**入参类型、入参顺序、入参个数**有特定要求，这个在后续章节会有介绍。这种方式面向的是基于 Java 7 作为编译版本的平台，但在以后会被逐步废弃，Google 官方也建议开发者尽量使用接口回调的形式

```kotlin
lifecycle.addObserver(object : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun onCreateEvent(lifecycleOwner: LifecycleOwner) {

    }

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    fun onDestroyEvent(lifecycleOwner: LifecycleOwner) {

    }

    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
    fun onAnyEvent(lifecycleOwner: LifecycleOwner, event: Lifecycle.Event) {

    }

})
```

Lifecycle 是一个抽象类，其本身的逻辑比较简单，在大多数时候我们会接触到的是其子类 LifecycleRegistry。Lifecycle 内部仅包含一个全局变量，三个抽象方法、两个枚举类

- `mInternalScopeRef` 用于在引入了 `lifecycle-common-ktx` 包的情况，即只有在使用 kotlin 协程库的时候才有用，在这里无需理会
- 三个抽象方法则分别用于添加 LifecycleObserver 、移除 LifecycleObserver、获取当前 Lifecycle 所处的状态值
- Event 类用于抽象 Activity/Fragment 的生命周期事件发生变化时所对应的事件。例如，当 Activity 的 `onCreate` 方法被回调时就会被抽象为 `ON_CREATE` 事件
- State 类用于标记 Lifecycle 的当前生命周期状态。例如，当 Activity 即将回调 `onDestory` 方法时则处于 `DESTROYED` 状态

```java
public abstract class Lifecycle {

    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    @NonNull
    AtomicReference<Object> mInternalScopeRef = new AtomicReference<>();

    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);

    @MainThread
    @NonNull
    public abstract State getCurrentState();

    public enum Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY
    }

    @SuppressWarnings("WeakerAccess")
    public enum State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;

        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}
```

# 二、相关的接口

在 Lifecycle 体系中，很多事件回调和类型定义都是通过接口的形式来实现的，这里再来罗列下开发者经常会使用到的几个接口及其作用

## LifecycleOwner

LifecycleOwner 接口用于标记其实现类具备 Lifecycle 对象，实现了该接口即意味着实现类具有生命周期。我们日常使用的 `androidx.appcompat.app.AppCompatActivity` 和 `androidx.fragment.app.Fragment` 均实现了该接口

```java
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```

## LifecycleObserver

LifecycleObserver 是一个空接口，大部分情况下真正具有使用意义的是它的子接口 ，LifecycleObserver 可以说仅是用于类型标记

```java
public interface LifecycleObserver {}
```

## LifecycleEventObserver

LifecycleEventObserver 用于监听 Lifecycle 的生命周期变化，可以获取到生命周期事件发生的具体变化

```java
public interface LifecycleEventObserver extends LifecycleObserver {

    void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event);
}
```

## FullLifecycleObserver

FullLifecycleObserver 根据 Activity 的生命周期回调方法扩展了几个同名的抽象方法，可以看成是对 LifecycleEventObserver 进行更加具体的事件拆分

```java
interface FullLifecycleObserver extends LifecycleObserver {

    void onCreate(LifecycleOwner owner);

    void onStart(LifecycleOwner owner);

    void onResume(LifecycleOwner owner);

    void onPause(LifecycleOwner owner);

    void onStop(LifecycleOwner owner);

    void onDestroy(LifecycleOwner owner);
}
```

## DefaultLifecycleObserver

DefaultLifecycleObserver 接口继承于 FullLifecycleObserver，DefaultLifecycleObserver 对父接口的所有方法都进行了默认实现。大多数情况下我们只需要一两种生命周期事件的通知，DefaultLifecycleObserver 的存在就使得我们可以按需重写父接口的方法而无需实现所有抽象方法。而**接口可以声明默认方法**这一特性是从 Java 8 开始才有的，所以只有当你的项目是以 Java 8 作为目标编译版本时才可以使用 DefaultLifecycleObserver。如果使用 FullLifecycleObserver 的话我们就必须实现所有抽象方法

```java
public interface DefaultLifecycleObserver extends FullLifecycleObserver {
    @Override
    default void onCreate(@NonNull LifecycleOwner owner) {}

    @Override
    default void onStart(@NonNull LifecycleOwner owner) {}

    @Override
    default void onResume(@NonNull LifecycleOwner owner) {}

    @Override
    default void onPause(@NonNull LifecycleOwner owner) {}

    @Override
    default void onStop(@NonNull LifecycleOwner owner) {}

    @Override
    default void onDestroy(@NonNull LifecycleOwner owner) {}
}
```

> DefaultLifecycleObserver 包含于 `androidx.lifecycle:lifecycle-common-java8:xxx` 这个依赖库内，该依赖库也只包含该接口，从依赖库的命名也可以看出它是用于 Java 8 平台的。Google 官方也建议开发者尽量使用 DefaultLifecycleObserver ，因为 Java 8 最终是会成为 Android 开发的主流，而 Java 7 平台下通过注解来实现生命周期回调的方式最终会被废弃

## FullLifecycleObserverAdapter

FullLifecycleObserverAdapter 实现了 LifecycleEventObserver 接口，用于在收到 Lifecycle 生命周期事件状态变化时对 FullLifecycleObserver、LifecycleEventObserver 进行事件转发

```java
class FullLifecycleObserverAdapter implements LifecycleEventObserver {

    private final FullLifecycleObserver mFullLifecycleObserver;
    private final LifecycleEventObserver mLifecycleEventObserver;

    FullLifecycleObserverAdapter(
            FullLifecycleObserver fullLifecycleObserver,
            LifecycleEventObserver lifecycleEventObserver) {
        mFullLifecycleObserver = fullLifecycleObserver;
        mLifecycleEventObserver = lifecycleEventObserver;
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
        switch (event) {
            case ON_CREATE:
                mFullLifecycleObserver.onCreate(source);
                break;
            case ON_START:
                mFullLifecycleObserver.onStart(source);
                break;
            case ON_RESUME:
                mFullLifecycleObserver.onResume(source);
                break;
            case ON_PAUSE:
                mFullLifecycleObserver.onPause(source);
                break;
            case ON_STOP:
                mFullLifecycleObserver.onStop(source);
                break;
            case ON_DESTROY:
                mFullLifecycleObserver.onDestroy(source);
                break;
            case ON_ANY:
                throw new IllegalArgumentException("ON_ANY must not been send by anybody");
        }
        if (mLifecycleEventObserver != null) {
            mLifecycleEventObserver.onStateChanged(source, event);
        }
    }
}
```

# 三、ReportFragment

再来回顾下我们如今是如何通过 Lifecycle 来实现 Activity 的生命周期监听的

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    lifecycle.addObserver(object : LifecycleEventObserver {
        override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
            when (event) {
                Lifecycle.Event.ON_CREATE -> TODO()
                Lifecycle.Event.ON_START -> TODO()
                Lifecycle.Event.ON_RESUME -> TODO()
                Lifecycle.Event.ON_PAUSE -> TODO()
                Lifecycle.Event.ON_STOP -> TODO()
                Lifecycle.Event.ON_DESTROY -> TODO()
                Lifecycle.Event.ON_ANY -> TODO()
            }
        }
    })
}
```

用是这样就能用了，但深究起来，此时一个很显而易见的问题就是，LifecycleEventObserver 是如何取得各个生命周期状态变化的事件 Lifecycle.Event 呢？或者说，是谁回调了 `onStateChanged` 方法呢？

现在我们在日常开发中使用的 Activity 往往都是继承于 `androidx.appcompat.appcompat:xxx` 这个包内的 AppCompatActivity，而 AppCompatActivity 最终会继承于 `androidx.core.app.ComponentActivity`， ComponentActivity 的 `onCreate` 方法是这样的：

```kotlin
@SuppressLint("RestrictedApi")
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
}
```

而正是通过 ReportFragment 使得 LifecycleEventObserver 可以接收到 Activity 所有的的 Lifecycle.Event。这里就来详细看看 ReportFragment 的内部源码，一步步了解其实现逻辑

`injectIfNeededIn()` 是一个静态方法，以 `android.app.Activity` 对象作为入参参数

```java
public static void injectIfNeededIn(Activity activity) {
    if (Build.VERSION.SDK_INT >= 29) {
        // On API 29+, we can register for the correct Lifecycle callbacks directly
        // 直接向 android.app.Activity 注册生命周期回调
        activity.registerActivityLifecycleCallbacks(new LifecycleCallbacks());
    }
    // Prior to API 29 and to maintain compatibility with older versions of
    // ProcessLifecycleOwner (which may not be updated when lifecycle-runtime is updated and
    // need to support activities that don't extend from FragmentActivity from support lib),
    // use a framework fragment to get the correct timing of Lifecycle events
    // 向 activity 添加一个不可见的 fragment
    android.app.FragmentManager manager = activity.getFragmentManager();
    if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
        manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
        // Hopefully, we are the first to make a transaction.
        manager.executePendingTransactions();
    }
}
```

`injectIfNeededIn()` 方法会根据两种情况来进行事件分发：

- 运行设备的系统版本号小于 29。此情况会通过向 Activity 添加一个无 UI 界面的 Fragment（即 ReportFragment），间接获得 Activity 的各个生命周期事件的回调通知
- 运行设备的系统版本号大于等于29。此情况会向 Activity 注册一个 LifecycleCallbacks ，以此来直接获得各个生命周期事件的回调通知。这里应该还牵扯到对旧版本 ProcessLifecycleOwner 和 support 库的兼容，所以此时也会同时执行第一种情况的操作

之所以会进行这两种情况区分，是因为 `registerActivityLifecycleCallbacks` 中的 `onActivityPostXXX` 和 `onActivityPreXXX` 等方法是 SDK 29 时新添加的方法。当版本小于 29 时，就还是需要通过 ReportFragment 来间接取得事件通知

## SDK >= 29

先来看下 LifecycleCallbacks，其作用就是会在 Activity 的 `onCreate、onStart、onResume` 等方法**被调用后**通过 `dispatch(activity, Lifecycle.Event.ON_XXX)` 方法发送相应的 Event 值，并在 `onPause、onStop、onDestroy` 等方法**被调用前**发送相应的 Event 值

```java
static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {
    @Override
    public void onActivityCreated(@NonNull Activity activity, @Nullable Bundle bundle) {}

    @Override
    public void onActivityPostCreated(
            @NonNull Activity activity, @Nullable Bundle savedInstanceState) {
        dispatch(activity, Lifecycle.Event.ON_CREATE);
    }

    ···

    @Override
    public void onActivityPreDestroyed(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_DESTROY);
    }

    @Override
    public void onActivityDestroyed(@NonNull Activity activity) {}
}
```

`dispatch()` 方法拿到 Event 值后，就会先通过 Activity 拿到 Lifecycle 对象，再通过类型判断拿到 LifecycleRegistry 对象，最终通过调用 `handleLifecycleEvent()` 方法将 Event 值传递出去，从而使得外部得到各个生命周期事件的通知

```java
@SuppressWarnings("deprecation")
static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
    // LifecycleRegistryOwner 已被废弃，主要看 LifecycleOwner
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }

    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

从这也可以猜到，`androidx.appcompat.app.AppCompatActivity` 实现了 LifecycleOwner 接口后返回的 Lifecycle 对象就是 LifecycleRegistry，实际上 `androidx.fragment.app.Fragment` 也一样

## SDK < 29

再来看下向 Activity 添加的 ReportFragment 是如何生效的。由于 ReportFragment 是挂载在 Activity 身上的，ReportFragment 本身的生命周期方法和所在的 Activity 是相关联的，通过在 ReportFragment 相应的生命周期方法里调用 `dispatch(Lifecycle.Event.ON_XXXX)` 方法发送相应的 Event 值，以此来间接获得 Activity 的各个生命周期事件的回调通知

```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    ···
    dispatch(Lifecycle.Event.ON_CREATE);
}

@Override
public void onStart() {
    super.onStart();
    ···
    dispatch(Lifecycle.Event.ON_START);
}

@Override
public void onDestroy() {
    super.onDestroy();
    dispatch(Lifecycle.Event.ON_DESTROY);
    ···
}
```

`dispatch()` 方法内部会判断目标设备的版本号来决定是否真的分发 Event 值，避免当 SDK 版本号大于 29 时和 LifecycleCallbacks 重复发送

```java
private void dispatch(@NonNull Lifecycle.Event event) {
    if (Build.VERSION.SDK_INT < 29) {
        dispatch(getActivity(), event);
    }
}
```

这样，ReportFragment 就通过上述逻辑向外部转发了 Activity 发生的 Event 值

# 四、LifecycleRegistry

ReportFragment 最终在向外传出 Lifecycle.Event 值时，调用的都是 LifecycleRegistry 对象的 `handleLifecycleEvent(Lifecycle.Event)` 方法，既然需要的 Event 值已经拿到了，那再来看下 LifecycleRegistry 是如何将 Event 值转发给 LifecycleObserver 的

LifecycleRegistry 是整个 Lifecycle 家族内一个很重要的类，其屏蔽了生命周期持有类（Activity / Fragment 等）的具体类型，使得外部（Activity / Fragment 等）可以只负责转发生命周期事件，由 LifecycleRegistry 来实现具体的事件回调和状态管理。`androidx.activity.ComponentActivity` 和 `androidx.fragment.app.Fragment` 都使用到了 LifecycleRegistry

```java
public class ComponentActivity extends androidx.core.app.ComponentActivity
        implements LifecycleOwner,
                ViewModelStoreOwner,
                HasDefaultViewModelProviderFactory,
                SavedStateRegistryOwner,
                OnBackPressedDispatcherOwner {

    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}

public class Fragment
        implements ComponentCallbacks,
                OnCreateContextMenuListener,
                LifecycleOwner,
                ViewModelStoreOwner,
                HasDefaultViewModelProviderFactory,
                SavedStateRegistryOwner {

    LifecycleRegistry mLifecycleRegistry;

    @Override
    @NonNull
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```

在具体看 LifecycleRegistry 的实现逻辑之前，需要先对 LifecycleRegistry 的定位和必须具备的功能有一个大致的了解，可以从我们想要的效果来逆推实现这个效果所需要的步骤：

1. 不单单是 Activity 和 Fragment 可以实现 LifecycleOwner 接口，像 Service、Dialog 等具有生命周期的类一样可以实现 LifecycleOwner 接口，而不管 LifecycleOwner 的实现类是什么，其本身所需要实现的功能和逻辑都是一样的：addObserver、removeObserver、getCurrentState、遍历循环 observers 进行 Event 通知等。所以 Google 官方势必需要提供一个通用的 Lifecycle 实现类，以此来简化开发者实现 LifecycleOwner 接口的成本，最终的实现类即 LifecycleRegistry（之后假设需要实现 LifecycleOwner 接口的仅有 Activity 一种，方便读者理解）
2. LifecycleRegistry 需要持有 LifecycleOwner 对象来判断是否可以向其回调，但为了避免内存泄漏不能直接强引用 LifecycleOwner
3. LifecycleRegistry 向 Observer 发布 Event 值的触发条件有两种：
   - 新添加了一个 Observer，需要向其同步 Activity 当前的 State 值。例如，当 Activity 处于 RESUMED 状态时向其添加了一个 LifecycleEventObserver ，此时就必须向 LifecycleEventObserver 同步当前的最新状态值，所以 LifecycleEventObserver 就会先后收到 CREATED、STARTED、RESUMED 三个 Event
   - Activity 的生命周期状态发生了变化，需要向 Observer 同步最新的 State 值

有了以上的几点认知后，再来看下 LifecycleRegistry 的大致逻辑

LifecycleRegistry 自然是 Lifecycle 的子类，其构造函数需要传入 LifecycleOwner 对象

```java
public class LifecycleRegistry extends Lifecycle {

    // mState 用来标记 LifecycleOwner 对象所处的当前生命周期状态
    private State mState;

    // 持有对 LifecycleOwner 的弱引用，避免内存泄露
    private final WeakReference<LifecycleOwner> mLifecycleOwner;

    public LifecycleRegistry(@NonNull LifecycleOwner provider) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;
    }
}
```

`addObserver()` 方法的主要逻辑是：将传入的 observer 对象包装为 ObserverWithState 类型，方便将 **注解形式的 LifecycleObserver（Java 7）和接口实现的 LifecycleObserver（Java 8）** 进行状态回调时的入口统一为 `dispatchEvent()` 方法。此外，由于当添加 LifecycleObserver 时 Lifecycle 可能已经处于非 INITIALIZED 状态了，所以需要通过循环检查的方式来向 ObserverWithState 逐步下发 Event 值

```java
// Lifecycle 类中对 addObserver 方法添加了 @MainThread 注解，意思是该方法只能用于主线程调用
// 所以此处不考虑多线程的情况
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    if (previous != null) {
        // 如果 observer 之前已经传进来过了，则不重复添加，直接返回
        return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // 如果 LifecycleOwner 对象已经被回收了，则直接返回
        return;
    }

    // 如果 isReentrance 为 true，意味着当前存在重入的情况：
    // 1. mAddingObserverCounter != 0。会出现这种情况，是由于开发者先添加了一个 LifecycleObserver，
    //   当还在向其回调事件的过程中，在回调方法里又再次调用了 addObserver 方法添加了一个新的 LifecycleObserver
    // 2.mHandlingEvent 为 true。即此时正处于向外回调 Lifecycle.Event 的状态
    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;

    State targetState = calculateTargetState(observer);

    // 递增加一，标记当前正处于向新添加的 LifecycleObserver 回调 Event 值的过程
    mAddingObserverCounter++;

    // statefulObserver.mState.compareTo(targetState) < 0 成立的话说明 State 值还没遍历到目标状态
    // mObserverMap.contains(observer) 成立的话说明 observer 还没有并移除
    // 因为有可能在遍历过程中开发者主动在回调方法里将 observer 给移除掉了，所以这里每次循环都检查下
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        // 将 observer 已经遍历到的当前的状态值 mState 保存下来
        pushParentState(statefulObserver.mState);
        // 向 observer 回调进入“statefulObserver.mState”前需要收到的 Event 值
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        // 移除 mState
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}
```

向 LifecycleObserver 回调事件的过程可以用以下这张官方提供的图来展示

1. 假设当前 LifecycleRegistry 的 `mState` 处于 RESUMED 状态，此时通过 `addObserver` 方法新添加的 LifecycleObserver 会被包装为 ObserverWithState，且初始化状态为 INITIALIZED。由于 RESUMED 大于INITIALIZED，ObserverWithState 就会按照 `INITIALIZED -> CREATED -> STARTED -> RESUMED ` 这样的顺序先后收到事件通知
2. 假设当前 LifecycleRegistry 的 `mState` 处于 STARTED 状态。如果 LifecycleRegistry 收到 ON_RESUME 事件，`mState` 就需要变更为 RESUMED；如果 LifecycleRegistry 收到 ON_STOP 事件，`mState` 就需要变更为 CREATED；所以说，LifecycleRegistry 的 `mState` 会先后向不同方向迁移

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5314c16fe94b47569f16e34e5f308fdd~tplv-k3u1fbpfcp-zoom-1.image)

ObserverWithState 将外界传入的 LifecycleObserver 对象传给 Lifecycling 进行类型包装，将反射逻辑和接口回调逻辑都给汇总综合成一个新的 LifecycleEventObserver 对象，从而使得 Event 分发过程都统一为 `mLifecycleObserver.onStateChanged`

```java
static class ObserverWithState {
    State mState;
    LifecycleEventObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```

在上文提到过，ReportFragment 最终在向外传出 Lifecycle.Event 值时，调用的都是 LifecycleRegistry 对象的 `handleLifecycleEvent(Lifecycle.Event)` 方法，该方法会根据接收到的 Event 值换算出对应的 State 值，然后更新本地的 `mState`，再向所有 Observer 进行事件通知，最终还是会调用到 ObserverWithState 的 `dispatchEvent` 方法，所以后边我们再来重点关注 `dispatchEvent` 方法即可

```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}

private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}
```

需要注意的一点是，对 `androidx.fragment.app.Fragment` 生命周期事件的监听一样需要使用到 LifecycleRegistry，Fragment 内部最终也是通过调用其 `handleLifecycleEvent(Lifecycle.Event)` 方法来完成其本身的生命周期事件通知，代码较为简单，这里不再赘述

# 五、Lifecycling

上面说到了，LifecycleRegistry 会将外部传入的所有 LifecycleObserver 根据 Lifecycling 包装成 LifecycleEventObserver 对象，这里先来解释下为什么需要进行这层包装

1. LifecycleEventObserver 和 FullLifecycleObserver 都是继承于 LifecycleObserver 的接口，开发者自己实现的自定义 Observer 可能同时实现了这两个接口或者实现了任一接口，LifecycleRegistry 必须在有事件触发的时候回调存在的所有接口方法
2. 除了通过接口方法来实现回调外，Google 也提供了通过注解来实现回调的方式，此时就需要通过反射来实现回调

基于以上要求，如果在 LifecycleRegistry 中直接对外部传入的 Observer 来进行**类型判断、接口回调、反射调用**等一系列操作的话，那势必会使得 LifecycleRegistry 整个类非常的臃肿，所以 Lifecycling 的作用就是来将这一系列的逻辑给封装起来，仅仅开放一个 `onStateChanged` 方法即可让 LifecycleRegistry 完成整个事件分发，从而使得整个流程会更加清晰明了且职责分明

那现在就来看下 `lifecycleEventObserver 方法的逻辑`

```java
@NonNull
static LifecycleEventObserver lifecycleEventObserver(Object object) {
    // 对应于上述的第一点
    boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
    boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
    if (isLifecycleEventObserver && isFullLifecycleObserver) {
        // 如果 object 对象同时继承了 LifecycleEventObserver 和 FullLifecycleObserver 接口
        // 则将其包装为 FullLifecycleObserverAdapter 对象来进行事件转发
        return new FullLifecycleObserverAdapter(
                (FullLifecycleObserver) object, (LifecycleEventObserver) object);
    }
    if (isFullLifecycleObserver) {
        // 同上
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
    }
    if (isLifecycleEventObserver) {
        // object 已经是需要的目标类型了（LifecycleEventObserver），直接原样返回即可
        return (LifecycleEventObserver) object;
    }

    // 对应于上述所说的第二点，即反射操作
    final Class<?> klass = object.getClass();
    int type = getObserverConstructorType(klass);
    if (type == GENERATED_CALLBACK) {
        List<Constructor<? extends GeneratedAdapter>> constructors =
                sClassToAdapters.get(klass);
        if (constructors.size() == 1) {
            GeneratedAdapter generatedAdapter =
                    createGeneratedAdapter(constructors.get(0), object);
            return new SingleGeneratedAdapterObserver(generatedAdapter);
        }
        GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
        for (int i = 0; i < constructors.size(); i++) {
            adapters[i] = createGeneratedAdapter(constructors.get(i), object);
        }
        return new CompositeGeneratedAdaptersObserver(adapters);
    }
    return new ReflectiveGenericLifecycleObserver(object);
}
```

## 第一种情况

FullLifecycleObserver 根据 Activity / Fragment 这两个类的生命周期回调方法扩展了几个同名的抽象方法，可以看成是对 LifecycleEventObserver 进行更加具体的事件拆分，让使用者可以只处理自己关心的生命周期事件，这一般是用于 Java 8 以上的编译平台

```java
interface FullLifecycleObserver extends LifecycleObserver {

    void onCreate(LifecycleOwner owner);

    void onStart(LifecycleOwner owner);

    void onResume(LifecycleOwner owner);

    void onPause(LifecycleOwner owner);

    void onStop(LifecycleOwner owner);

    void onDestroy(LifecycleOwner owner);
}
```

FullLifecycleObserverAdapter 实现了 LifecycleEventObserver 接口，用于在收到 Lifecycle 生命周期事件状态变化时，对两个构造参数 FullLifecycleObserver、LifecycleEventObserver 进行事件转发

```java
class FullLifecycleObserverAdapter implements LifecycleEventObserver {

    private final FullLifecycleObserver mFullLifecycleObserver;
    private final LifecycleEventObserver mLifecycleEventObserver;

    FullLifecycleObserverAdapter(
            FullLifecycleObserver fullLifecycleObserver,
            LifecycleEventObserver lifecycleEventObserver) {
        mFullLifecycleObserver = fullLifecycleObserver;
        mLifecycleEventObserver = lifecycleEventObserver;
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
        switch (event) {
            case ON_CREATE:
                mFullLifecycleObserver.onCreate(source);
                break;
            case ON_START:
                mFullLifecycleObserver.onStart(source);
                break;
            case ON_RESUME:
                mFullLifecycleObserver.onResume(source);
                break;
            case ON_PAUSE:
                mFullLifecycleObserver.onPause(source);
                break;
            case ON_STOP:
                mFullLifecycleObserver.onStop(source);
                break;
            case ON_DESTROY:
                mFullLifecycleObserver.onDestroy(source);
                break;
            case ON_ANY:
                throw new IllegalArgumentException("ON_ANY must not been send by anybody");
        }
        if (mLifecycleEventObserver != null) {
            mLifecycleEventObserver.onStateChanged(source, event);
        }
    }
}
```

## 第二种情况

对于第二种情况的反射操作，其逻辑相对来说会比较复杂，需要进行一系列的**类型判断、类型缓存、反射调用**等操作，这里主要来看下 ClassesInfoCache 对于使用 OnLifecycleEvent 进行注解的方法是如何进行限制的

我们知道，Java 平台的反射操作是一个比较低效和耗费性能的行为，为了避免每次需要进行事件回调时都再来对包含 OnLifecycleEvent 注解的 class 对象进行反射解析，所以 Lifecycling 内部对 Class、Method 等进行了缓存，以便后续复用。而 Lifecycling 就将这些缓存信息都封装存放在了 ClassesInfoCache 内部。此外，被注解的方法的**入参类型、入参顺序、入参个数都有着严格的限制**，毕竟如果开发者为回调方法声明了一个 String 类型的入参参数的话，这种情况肯定是不合理的

ClassesInfoCache 内部会判断指定的 class 对象是否包含使用了 OnLifecycleEvent 进行注解的方法，并将判断结果缓存在 `mHasLifecycleMethods` 内，缓存信息会根据 `createInfo(klass, methods)` 来进行获取

```java
// 判断指定的 class 对象是否包含使用了 OnLifecycleEvent 进行注解的方法
boolean hasLifecycleMethods(Class<?> klass) {
    Boolean hasLifecycleMethods = mHasLifecycleMethods.get(klass);
    if (hasLifecycleMethods != null) {
        // 如果本地有缓存的话则直接返回缓存值
        return hasLifecycleMethods;
    }
    // 本地还没有缓存值，以下逻辑就是来通过反射判断 klass 是否包含使用 OnLifecycleEvent 进行注解的方法

    // 获取 klass 包含的所有方法
    Method[] methods = getDeclaredMethods(klass);
    for (Method method : methods) {
        OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
        if (annotation != null) {
            // Optimization for reflection, we know that this method is called
            // when there is no generated adapter. But there are methods with @OnLifecycleEvent
            // so we know that will use ReflectiveGenericLifecycleObserver,
            // so we createInfo in advance.
            // CreateInfo always initialize mHasLifecycleMethods for a class, so we don't do it
            // here.
            createInfo(klass, methods);
            return true;
        }
    }
    mHasLifecycleMethods.put(klass, false);
    return false;
}
```

而正是在 `createInfo` 方法内部对被注解方法的入参类型、入参顺序、入参个数等进行了限制，如果不符合规定，在运行时就会直接抛出异常

```java
// 以下三个整数值用于标记被注解的方法的入参参数的个数
// 不包含入参参数
private static final int CALL_TYPE_NO_ARG = 0;
// 包含一个入参参数
private static final int CALL_TYPE_PROVIDER = 1;
// 包含两个入参参数
private static final int CALL_TYPE_PROVIDER_WITH_EVENT = 2;

private CallbackInfo createInfo(Class<?> klass, @Nullable Method[] declaredMethods) {
    Class<?> superclass = klass.getSuperclass();
    Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
    if (superclass != null) {
        CallbackInfo superInfo = getInfo(superclass);
        if (superInfo != null) {
            handlerToEvent.putAll(superInfo.mHandlerToEvent);
        }
    }

    Class<?>[] interfaces = klass.getInterfaces();
    for (Class<?> intrfc : interfaces) {
        for (Map.Entry<MethodReference, Lifecycle.Event> entry :
                getInfo(intrfc).mHandlerToEvent.entrySet()) {
            verifyAndPutHandler(handlerToEvent, entry.getKey(), entry.getValue(), klass);
        }
    }

    Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
    boolean hasLifecycleMethods = false;
    for (Method method : methods) {
        // 找到包含 OnLifecycleEvent 注解的方法
        OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
        if (annotation == null) {
            continue;
        }
        hasLifecycleMethods = true;

        // 以下的所有逻辑是这样的：
        // 1. 获取 method 所对应的方法的参数个数和参数类型，即 params
        // 2. 如果参数个数为 0，则 callType = CALL_TYPE_NO_ARG，method 不包含入参参数
        // 3. 如果参数个数大于 0，则第一个参数必须是 LifecycleOwner 类型的对象，否则抛出异常
        // 3.1、如果参数个数为 1，则 callType = CALL_TYPE_PROVIDER
        // 3.2、如果参数个数为 2，则注解值 annotation 必须是 Lifecycle.Event.ON_ANY
        //     且第二个参数必须是 Lifecycle.Event 类型的对象，否则抛出异常
        //     如果一切都符合条件，则 callType = CALL_TYPE_PROVIDER_WITH_EVENT
        // 3.3、如果参数个数大于 2，则抛出异常，即要求 method 最多包含两个参数，且对参数类型和参数顺序进行了限制
        Class<?>[] params = method.getParameterTypes();
        int callType = CALL_TYPE_NO_ARG;
        if (params.length > 0) {
            callType = CALL_TYPE_PROVIDER;
            if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                throw new IllegalArgumentException(
                        "invalid parameter type. Must be one and instanceof LifecycleOwner");
            }
        }
        Lifecycle.Event event = annotation.value();

        if (params.length > 1) {
            callType = CALL_TYPE_PROVIDER_WITH_EVENT;
            if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
                throw new IllegalArgumentException(
                        "invalid parameter type. second arg must be an event");
            }
            if (event != Lifecycle.Event.ON_ANY) {
                throw new IllegalArgumentException(
                        "Second arg is supported only for ON_ANY value");
            }
        }
        if (params.length > 2) {
            throw new IllegalArgumentException("cannot have more than 2 params");
        }
        MethodReference methodReference = new MethodReference(callType, method);
        verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
    }
    CallbackInfo info = new CallbackInfo(handlerToEvent);
    mCallbackMap.put(klass, info);
    mHasLifecycleMethods.put(klass, hasLifecycleMethods);
    return info;
}
```

然后在 MethodReference 类内部的 `invokeCallback()` 方法完成最终的反射调用。MethodReference 用于缓存具有 OnLifecycleEvent 注解的方法（Method）以及该方法所具有的入参个数（知道了入参个数就知道了该如何进行反射调用），通过 `invokeCallback()` 方法来进行 Lifecycle.Event 事件通知

```java
static class MethodReference {
    final int mCallType;
    final Method mMethod;

    MethodReference(int callType, Method method) {
        mCallType = callType;
        mMethod = method;
        mMethod.setAccessible(true);
    }

    void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
        //noinspection TryWithIdenticalCatches
        // 根据入参个数来传递特定的参数并进行反射回调
        // 因此用 OnLifecycleEvent 进行注解的方法，其入参个数、入参类型、入参声明顺序都有固定的要求
        // 当不符合要求时会导致反射失败从而抛出异常
        try {
            switch (mCallType) {
                case CALL_TYPE_NO_ARG:
                    mMethod.invoke(target);
                    break;
                case CALL_TYPE_PROVIDER:
                    mMethod.invoke(target, source);
                    break;
                case CALL_TYPE_PROVIDER_WITH_EVENT:
                    mMethod.invoke(target, source, event);
                    break;
            }
        } catch (InvocationTargetException e) {
            throw new RuntimeException("Failed to call observer method", e.getCause());
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }

    // 省略无关方法
    ···
}
```

# 六、总结

Lifecycle 的整个事件流程都在上文大致讲述完毕了，这里再来做下总结

1. 我们日常使用的 `androidx.appcompat.app.AppCompatActivity` 和 `androidx.fragment.app.Fragment` 都实现了 LifecycleOwner 接口，其 `getLifecycle()` 方法返回的都是 LifecycleRegistry
2. `androidx.core.app.ComponentActivity` 默认挂载了一个无 UI 界面的 ReportFragment，ReportFragment 会根据用户手机的系统版本号来选择用不同的方式获取到 Activity 的生命周期事件，最终调用 LifecycleRegistry 的 `handleLifecycleEvent(Lifecycle.Event)` 方法将 Lifecycle.Event 传递出去。此时 LifecycleRegistry 就拿到了 Lifecycle.Event。ReportFragment 的存在意义就是完全屏蔽了系统版本差异，使得外部可以通过一个统一的方法得到回调通知
3. `androidx.fragment.app.Fragment` 会直接在内部调用 LifecycleRegistry 的 `handleLifecycleEvent(Lifecycle.Event)` 方法完成事件通知
4. LifecycleRegistry 会将外部 `addObserver` 传进来的 LifecycleObserver 对象都给包装成 LifecycleEventObserver 对象，屏蔽了外部 LifecycleObserver 的差异性（可能是接口，也可能是注解）
5. LifecycleRegistry 通过直接调用 LifecycleEventObserver 对象的 `onStateChanged` 方法来完成最终的事件回调，至此整个流程就完成了