Android 系统中，Window 在代码层次上是一个抽象类，在概念上表示的是一个窗口。Android 中所有的视图都是通过 Window 来呈现的，例如 Activity、Dialog 和 Toast 等，它们实际上都是挂载在 Window 上的。大部分情况下应用层开发者很少需要来和 Window 打交道，Activity 已经隐藏了 Window 的具体实现逻辑了，但我觉得来了解 Window 机制的一个比较大的好处是**可以加深我们对 View 绘制流程以及事件分发机制的了解**，这两个操作就涉及到我们的日常开发了，实现自定义 View 和解决 View 的滑动冲突时都需要我们掌握这方面的知识点，而这两个操作和 Window 机制有很大的关联。视图树只有被挂载到 Window 后才会触发视图树的绘制流程，之后视图树才有机会接收到用户的触摸事件。也就是说，视图树被挂载到了 Window 上是 Activity 和 Dialog 能够展示到屏幕上且和用户做交互的前置条件

本文就以 Activity 为例子，展开讲解 Activity 是如何挂载到 Window 上的，基于 Android API 30 进行分析，希望对你有所帮助

# 一、Window

Window 存在的意义是什么呢？

大部分情况下，用户都是在和应用的 Activity 做交互，应用在 Activity 上接收用户的输入并在 Activity 上向用户做出交互反馈。例如，在 Activity 中显示了一个 Button，当用户点击后就会触发 OnClickListener，这个过程中用户就是在和 Activity 中的视图树做交互，此时还没有什么问题。可是，当需要在 Activity 上弹出 Dialog 时，系统需要确保 Dialog 是会覆盖在 Activity 之上的，有触摸事件时也需要确保 Dialog 是先于 Activity 接收到的；当启动一个新的 Activity 时又需要覆盖住上一个 Activity 以及其显示的 Dialog；在弹出 Toast 时，又需要确保 Toast 是覆盖在 Activity 和 Dialog 之上的

这种种要求就涉及到了一个层次管理问题，系统需要对当前屏幕上显示的多个视图树进行统一管理，这样才能来**决定不同视图树的显示层次以及在接收触摸事件时的优先级**。系统就通过 Window 这个概念来实现上述目的

想要在屏幕上显示一个 Window 并不算多复杂，代码大致如下所示

```kotlin
private val windowManager by lazy {
    context.getSystemService(Context.WINDOW_SERVICE) as WindowManager
}

private val floatBallView by lazy {
    FloatBallView(context)
}

private val floatBallWindowParams: WindowManager.LayoutParams by lazy {
    WindowManager.LayoutParams().apply {
        width = FloatBallView.VIEW_WIDTH
        height = FloatBallView.VIEW_HEIGHT
        gravity = Gravity.START or Gravity.CENTER_VERTICAL
        flags =
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
        type = if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) {
            WindowManager.LayoutParams.TYPE_SYSTEM_ALERT
        } else {
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
        }
    }
}

fun showFloatBall() {
    windowManager.addView(floatBallView, floatBallWindowParams)
}
```

显示一个 Window 最基本的操作流程有：

1. 声明希望显示的 View，即本例子中的 floatBallView，其承载了我们希望用户看到的视图界面
2. 声明 View 的位置参数和交互逻辑，即本例子中的 floatBallWindowParams，其规定了 floatBallView 在屏幕上的位置以及和用户的交互逻辑
3. 通过 WindowManager 来添加 floatBallView，从而将 floatBallView 挂载到 Window 上，WindowManager 是外界访问 Window 的入口

当中，WindowManager.LayoutParams 的 flags 属性就用于控制 Window 的显示特性和交互逻辑，常见的有以下几个：

1. WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE。表示当前 Window 不需要获取焦点，也不需要接收各种按键输入事件，按键事件会直接传递给下层具有焦点的 Window

2. WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL。表示当前 Window 区域的单击事件希望自己处理，其它区域的事件则传递给其它 Window

3. WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED。表示当前 Window 希望显示在锁屏界面

此外，WindowManager.LayoutParams 的 type 属性就用于表示 Window 的类型。Window 有三种类型：应用 Window、子 Window、系统 Window。应用类Window 对应 Activity。子 Window 具有依赖关系，不能单独存在，需要附属在特定的父 Window 之中，比如 Dialog 就是一个子 Window。系统 Window 是需要权限才能创建的 Window，比如 Toast 和 statusBar 都是系统 Window

从这也可以看出，系统 Window 是处于最顶层的，所以说 type 属性也用于控制 Window 的显示层级，显示层级高的 Window 就会覆盖在显示层级低的 Window 之上。应用 Window 的层级范围是 1～99，子 Window 的层级范围是 1000～1999，系统 Window 的层级范围是 2000～2999。如果想要让我们创建的 Window 位于其它 Window 之上，那么就需要使用比较大的层级值了，但想要显示自定义的系统级 Window 的话就必须向系统申请权限了

WindowManager.LayoutParams 内就声明了这些层级值，我们可以择需选取。例如，系统状态栏本身也是一个 Window，其 type 值就是 TYPE_STATUS_BAR

```java
public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {

    public int type;

    // 应用 Window 的开始值
    public static final int FIRST_APPLICATION_WINDOW = 1;
    // 应用 Window 的结束值
    public static final int LAST_APPLICATION_WINDOW = 99;

    // 子 Window 的开始值
    public static final int FIRST_SUB_WINDOW = 1000;
    // 子 Window 的结束值
    public static final int LAST_SUB_WINDOW = 1999;

    // 系统 Window 的开始值
    public static final int FIRST_SYSTEM_WINDOW = 2000;
    // 系统状态栏
    public static final int TYPE_STATUS_BAR = FIRST_SYSTEM_WINDOW;
    // 系统 Window 的结束值
    public static final int LAST_SYSTEM_WINDOW = 2999;
}
```

# 二、WindowManager

每个 Window 都会关联一个 View，想要显示 Window 也离不开 WindowManager，WindowManager 就提供了对 View 进行操作的能力。WindowManager 本身是一个接口，其又继承了另一个接口 ViewManager，WindowManager 最基本的三种操作行为就由 ViewManager 来定义，即**添加 View、更新 View、移除 View**

```java
public interface ViewManager {
    public void addView(View view, ViewGroup.LayoutParams params);

    public void updateViewLayout(View view, ViewGroup.LayoutParams params);

    public void removeView(View view);
}
```

WindowManager 的实现类是 WindowManagerImpl，其三种基本的操作行为都交由了 WindowManagerGlobal 去实现，这里使用到了桥接模式

```java
public final class WindowManagerImpl implements WindowManager {

    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(
                view, params, mContext.getDisplayNoVerify(), mParentWindow, mContext.getUserId());
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }

    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
}
```

这里主要看下 WindowManagerGlobal 是如何实现 `addView` 方法的即可

首先，WindowManagerGlobal 会对入参参数进行校验，并对 LayoutParams 做下参数调整。例如，如果当前要显示的是子 Window 的话，那么就需要使其 LayoutParams 遵循父 Window 的要求才行

```java
public void addView(
        View view,
        ViewGroup.LayoutParams params,
        Display display,
        Window parentWindow,
        int userId) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        // If there's no parent, then hardware acceleration for this view is
        // set from the application's hardware acceleration setting.
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                                & ApplicationInfo.FLAG_HARDWARE_ACCELERATED)
                        != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }
    ···
}
```

之后就会为当前的视图树（即 view）构建一个关联的 ViewRootImpl 对象，通过 ViewRootImpl 来绘制视图树并完成 Window 的添加过程。ViewRootImpl 的 `setView` 方法会触发启动整个视图树的绘制流程，即完成视图树的 Measure、Layout、Draw 流程，具体流程可以看我的另一篇文章：[一文读懂 View 的 Measure、Layout、Draw 流程](https://juejin.cn/post/6939540905581887502)

```java
public void addView(
        View view,
        ViewGroup.LayoutParams params,
        Display display,
        Window parentWindow,
        int userId) {
    ···

    ViewRootImpl root;
    View panelParentView = null;

    ···

    root = new ViewRootImpl(view.getContext(), display);

    view.setLayoutParams(wparams);

    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);

    // do this last because it fires off messages to start doing things
    try {
        // 启动和 view 关联的整个视图树的绘制流程
        root.setView(view, wparams, panelParentView, userId);
    } catch (RuntimeException e) {
        // BadTokenException or InvalidDisplayException, clean up.
        if (index >= 0) {
            removeViewLocked(index, true);
        }
        throw e;
    }
}
```

ViewRootImpl 内部最终会通过 WindowSession 来完成 Window 的添加过程，`mWindowSession` 是一个 Binder 对象，真正的实现类是 Session，也就是说，Window 的添加过程涉及到了 IPC 调用。后面就比较复杂了，能力有限就不继续看下去了

```java
mOrigWindowType = mWindowAttributes.type;
mAttachInfo.mRecomputeGlobalAttributes = true;
collectViewAttributes();
adjustLayoutParamsForCompatibility(mWindowAttributes);
res = mWindowSession.addToDisplayAsUser(
    mWindow, mSeq, mWindowAttributes,
    getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
    mAttachInfo.mDisplayCutout, inputChannel,
    mTempInsets, mTempControls
);
setFrame(mTmpFrame);
```

需要注意的是，这里所讲的视图树代表的是很多种不同的视图形式。在启动一个 Activity 或者显示一个 Dialog 的时候，我们都需要为它们指定一个布局文件，布局文件会通过 LayoutInflater 加载映射为一个具体的 View 对象，即最终 Activity 和 Dialog 都会被映射为一个 View 类型的视图树，它们都会通过 WindowManager 的 `addView` 方法来显示到屏幕上，WindowManager 对于 Activity 和 Dialog 来说具有统一的操作行为入口

# 三、Activity & Window

这里就以 Activity 为例子来展开讲解 Window 相关的知识点，所以也需要先对 Activity 的组成结构做个大致的介绍。Activity 和 Window 之间的关系可以用以下图片来表示

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a513336e6d04478cb5eb32789f4a9d72~tplv-k3u1fbpfcp-zoom-1.image)

1. 每个 Activity 均包含一个 Window 对象，即 Activity 和 Window 是一对一的关系

2. Window 是一个抽象类，其唯一的实现类是 PhoneWindow

3. PhoneWindow 内部包含一个 DecorView，DecorView 是 FrameLayout 的子类，其内部包含一个 LinearLayout，LinearLayout 中又包含两个自上而下的 childView，即 ActionBar 和 ContentParent。我们平时在 Activity 中调用的 `setContentView` 方法实际上就是在向 ContentParent 执行 `addView` 操作

Window 这个抽象类里定义了多个和 UI 操作相关的方法，我们平时在 Activity 中调用的 `setContentView` 和 `findViewById` 方法都会被转交由 Window 来实现，Window 是 Activity 和视图树系统交互的入口。例如，其 `getDecorView()` 方法就用于获取内嵌的 DecorView，`findViewById()` 方法就会将具体逻辑转交由 DecorView 来实现，因为 DecorView 才是真正包含 `contentView` 的容器类

```java
public abstract class Window {

    public Window(Context context) {
        mContext = context;
        mFeatures = mLocalFeatures = getDefaultFeatures(context);
    }

    public abstract void setContentView(@LayoutRes int layoutResID);

    @Nullable
    public <T extends View> T findViewById(@IdRes int id) {
        return getDecorView().findViewById(id);
    }

    public abstract void setTitle(CharSequence title);

    public abstract @NonNull View getDecorView();

    ···

}
```

# 四、Activity # setContentView

每个 Activity 内部都包含一个 Window 对象 `mWindow`，在 `attach` 方法中完成初始化，这说明 Activity 和 Window 是一对一的关系。`mWindow` 对象对应的是 PhoneWindow 类，这也是 Window 的唯一实现类

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
                Window.Callback,
                KeyEvent.Callback,
                OnCreateContextMenuListener,
                ComponentCallbacks2,
                Window.OnWindowDismissedCallback,
                AutofillManager.AutofillClient,
                ContentCaptureManager.ContentCaptureClient {

    @UnsupportedAppUsage private Window mWindow;

    @UnsupportedAppUsage private WindowManager mWindowManager;

    @UnsupportedAppUsage
    final void attach(
            Context context,
            ActivityThread aThread,
            Instrumentation instr,
            IBinder token,
            int ident,
            Application application,
            Intent intent,
            ActivityInfo info,
            CharSequence title,
            Activity parent,
            String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config,
            String referrer,
            IVoiceInteractor voiceInteractor,
            Window window,
            ActivityConfigCallback activityConfigCallback,
            IBinder assistToken) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        // 初始化 mWindow
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(mWindowControllerCallback);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        ···
    }

    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
}
```

Activity 的 `attach` 方法又是在 ActivityThread 的 `performLaunchActivity` 方法中被调用的，在通过反射生成 Activity 实例后就会调用 `attach` 方法，且可以看到该方法的调用时机是早于 Activity 的 `onCreate` 方法的。所以说，在生成 Activity 实例后不久其 Window 对象就已经被初始化了，而且早于各个生命周期回调函数

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ···
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                    "Unable to instantiate activity " + component + ": " + e.toString(), e);
        }
    }

    ···

    activity.attach(
            appContext,
            this,
            getInstrumentation(),
            r.token,
            r.ident,
            app,
            r.intent,
            r.activityInfo,
            title,
            r.parent,
            r.embeddedID,
            r.lastNonConfigurationInstances,
            config,
            r.referrer,
            r.voiceInteractor,
            window,
            r.configCallback,
            r.assistToken);

    ···

    if (r.isPersistable()) {
        mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
    } else {
        mInstrumentation.callActivityOnCreate(activity, r.state);
    }
    return activity;
}
```

此外，从 Activity 的 `setContentView` 的方法签名来看，具体逻辑都交由了 Window 的同名方法来实现，传入的 `layoutResID` 就是我们希望在屏幕上呈现的布局，那么 PhoneWindow 自然就需要去加载该布局文件生成对应的 View。而为了能够有一个对 View 进行统一管理的入口，View 应该要包含在一个指定的 ViewGroup 中才行，该 ViewGroup 指的就是 DecorView

# 五、PhoneWindow # setContentView

PhoneWindow 的 `setContentView` 方法的逻辑可以总结为：

1. PhoneWindow 内部包含一个 DecorView 对象 `mDecor`。DecorView 是 FrameLayout 的子类，其内部包含两个我们经常会接触到的 childView：actionBar 和 contentParent，actionBar 即 Activity 的标题栏，contentParent 即 Activity 的视图内容容器
2. 如果 `mContentParent` 为 null 的话则调用 `installDecor()` 方法来初始化 DecorView，从而同时初始化 `mContentParent`；不为 null 的话则移除 `mContentParent` 的所有 `childView`，为 `layoutResID` 腾出位置（不考虑转场动画，实际上最终的操作都一样）
3. 通过 `LayoutInflater.inflate` 生成 `layoutResID` 对应的 View，并将其添加到 `mContentParent` 中，从而将我们的目标视图挂载到一个统一的容器中（不考虑转场动画，实际上最终的操作都一样）
4. 当 ContentView 添加完毕后会回调 `Callback.onContentChanged` 方法，我们可以通过重写 Activity 的该方法从而得到布局内容改变的通知

所以说，Activity 的 `setContentView` 方法实际上就是在向 DecorView 的 `mContentParent` 执行 `addView` 操作，所以该方法才叫 `setContentView` 而非 `setView`

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {

    private DecorView mDecor;

    ViewGroup mContentParent;

    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene =
                    Scene.getSceneForLayout(mContentParent, layoutResID, getContext());
            transitionTo(newScene);
        } else {
            // 将 layoutResID 对应的 View 添加到 mContentParent 中
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }

        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            // 回调通知 contentView 发生变化了
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }

    private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

            // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
            mDecor.makeFrameworkOptionalFitsSystemWindows();

            final DecorContentParent decorContentParent =
                    (DecorContentParent) mDecor.findViewById(R.id.decor_content_parent);

            if (decorContentParent != null) {
                mDecorContentParent = decorContentParent;
                ···
            } else {
                ···
            }
            ···
        }
    }
}
```

`mContentParent` 通过 `generateLayout` 方法来完成初始化，该方法的逻辑可以分为三步：

1. 读取我们为 Activity 设置的 theme 属性，以此配置基础的 UI 风格。例如，如果我们设置了 `<item name="windowNoTitle">true</item>` 的话，那么就会执行 `requestFeature(FEATURE_NO_TITLE)` 来隐藏标题栏
2. 根据 features 来选择合适的布局文件，得到 `layoutResource`。之所以会有多种布局文件，是因为不同的 Activity 会有不同的显示要求，有的要求显示 title，有的要求显示 leftIcon，而有的可能全都不需要，为了避免控件冗余就需要来选择合适的布局文件。而虽然每种布局文件结构上略有不同，但均会包含一个 ID 名为 `content` 的 FrameLayout，`mContentParent` 就对应该 FrameLayout
3. DecorView 会拿到 `layoutResource` 生成对应的 View 对象并添加为自己的 childView，对应 DecorView 中的 `mContentRoot`，后续执行的 `findViewById(ID_ANDROID_CONTENT)` 操作就都是交由 DecorView 来实现的了，而正常来说每种 `layoutResource` 都会包含一个 ID 为 `content` 的 FrameLayout，如果发现找不到的话就直接抛出异常，否则就成功返回拿到 `mContentParent`

```java
protected ViewGroup generateLayout(DecorView decor) {
    // Apply data from current theme.

    TypedArray a = getWindowStyle();

    ···

    // 第一步
    if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
        requestFeature(FEATURE_NO_TITLE);
    } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
        // Don't allow an action bar if there is no title.
        requestFeature(FEATURE_ACTION_BAR);
    }

    ···

    // 第二步
    int layoutResource;
    ···
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

    // 第三步
    ViewGroup contentParent = (ViewGroup) findViewById(ID_ANDROID_CONTENT);
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }

    ···
    return contentParent;
}
```

# 六、DecorView

DecorView 是 FrameLayout 的子类，其 `onResourcesLoaded` 方法在拿到 PhoneWindow 传递过来的 `layoutResource` 后，就会生成对应的 View 并添加为自己的 childView，就像普通的 ViewGroup 执行 `addView` 方法一样，该 childView 就对应 `mContentRoot`，我们可以在 Activity 中通过 `(window.decorView as ViewGroup).getChildAt(0)` 来获取到 `mContentRoot`

所以 DecorView 可以看做是 Activity 中整个视图树的根布局

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {

    @UnsupportedAppUsage private PhoneWindow mWindow;

    ViewGroup mContentRoot;

    DecorView(
            Context context, int featureId, PhoneWindow window, WindowManager.LayoutParams params) {
        ···
    }

    void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
        if (mBackdropFrameRenderer != null) {
            loadBackgroundDrawablesIfNeeded();
            mBackdropFrameRenderer.onResourcesLoaded(
                    this,
                    mResizingBackgroundDrawable,
                    mCaptionBackgroundDrawable,
                    mUserCaptionBackgroundDrawable,
                    getCurrentColor(mStatusColorViewState),
                    getCurrentColor(mNavigationColorViewState));
        }
        mDecorCaptionView = createDecorCaptionView(inflater);
        final View root = inflater.inflate(layoutResource, null);
        if (mDecorCaptionView != null) {
            if (mDecorCaptionView.getParent() == null) {
                addView(mDecorCaptionView, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
            }
            mDecorCaptionView.addView(
                    root, new ViewGroup.MarginLayoutParams(MATCH_PARENT, MATCH_PARENT));
        } else {
            // Put it below the color views.
            addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        }
        mContentRoot = (ViewGroup) root;
        initializeElevation();
    }
}
```

# 七、ActivityThread

完成以上步骤后，此时其实还只是完成了 Activity 整个视图树的加载工作，虽然 Activity 的 `attach` 方法已经创建了 Window 对象，但还需要将 DecorView 提交给 WindowManager 后才能正式将视图树展示到屏幕上

DecorView 具体的提交时机还需要看 ActivityThread 的 `handleResumeActivity` 方法，该方法用于回调 Activity 的 `onResume` 方法，里面还会回调到 Activity 的 `makeVisible` 方法，从方法名可以猜出来 `makeVisible` 方法就用于令 Activity 变为可见状态

```java
@Override
public void handleResumeActivity(
        IBinder token, boolean finalStateRequest, boolean isForward, String reason) {
    ···
    r.activity.makeVisible();
    ···
}
```

`makeVisible` 方法会判断当前 Activity 是否已经将 DecorView 提交给 WindowManager 了，如果还没的话就进行提交，最后将 DecorView 的可见状态设为 VISIBLE，至此才建立起 Activity 和 WindowManager 之间的关联关系，之后 Activity 才正式对用户可见

```java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

# 八、做下总结

对以上流程做下总结

1. 每个 Activity 内部都包含一个 Window 对象，Activity 的 `setContentView`、`findViewById` 等操作都会交由 Window 来实现，Window 是 Activity 和整个 View 系统交互的入口
2. PhoneWindow 是 Window 这个抽象类的的唯一实现类，Activity 和 Dialog 内部都是通过 PhoneWindow 来加载视图树，将具体的视图树处理逻辑交由 PhoneWindow 实现，并通过 PhoneWindow 和视图树进行交互，因此 PhoneWindow 成为了上层类和视图树系统之间的交互入口，从而也将 Activity 和 Dialog 共同的视图逻辑给抽象出来了，减轻了上层类的负担，这也是 Window 机制存在的好处之一
3. PhoneWindow 根据 theme 和 features 得知 Activity 的基本视图属性，由此来选择合适的根布局文件 `layoutResource`，每种 `layoutResource` 虽然在布局结构上略有不同，但是均会包含一个 ID 名为 `content` 的 FrameLayout，`contentParent` 即该 FrameLayout。我们可以通过 `Window.ID_ANDROID_CONTENT` 来拿到该 ID，也可以在 Activity 中通过 `findViewById<View>(Window.ID_ANDROID_CONTENT)` 来获取到 `contentParent`
4. PhoneWindow 并不直接管理视图树，而是交由 DecorView 去管理。DecorView 会根据 `layoutResource` 来生成对应的 rootView 并将开发者指定的 contentView 添加为 `contentParent` 的 childView，所以可以将 DecorView 看做是视图树的根布局。正因为如此，Activity 的 `findViewById` 操作实际上会先交由 Window，Window 再交由 DecorView 去完成，因为 DecorView 才是实际持有 contentView 的容器类
5. View 和 ViewGroup 共同组成一个具体的视图树，视图树的根布局则是 DecorView，DecorView 的存在使得视图树有了一个统一的容器，有利于统一系统的主题样式并对所有 childView 进行统一管理
6. Activity 的 DecorView 是在 `makeVisible` 方法里提交给 WindowManager 的，之后 WindowManagerImpl 会通过 ViewRootImpl 来完成整个视图树的绘制流程，之后 Activity 才正式对用户可见

# 九、一个 Demo

这里我也提供一个自定义 Window 的 Demo，实现了基本的拖拽移动和点击事件，代码点击这里：[AndroidOpenSourceDemo](https://github.com/leavesczy/AndroidOpenSourceDemo)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8222b030014d4aa99f39e673f60e9ea7~tplv-k3u1fbpfcp-zoom-1.image)

# 十、一文系列

最近比较倾向于只用一篇文章来写一个知识点，也懒得总是想文章标题，就一直沿用一开始用的**一文读懂XXX**，写着写着也攒了蛮多篇文章了，之前也已经写了几篇关于 View 系统的文章，希望对你有所帮助

- [一文读懂 View 的 Measure、Layout、Draw 流程](https://juejin.cn/post/6939540905581887502)
- [一文读懂 View Post 的原理及缺陷](https://juejin.cn/post/6939763855216082974)
- [一文读懂 View 事件分发机制](https://juejin.cn/post/6931914294980411406)
- [一文读懂 Handler 机制](https://juejin.cn/post/6901682664617705485)
- [一文读懂 SharedPreferences 的缺陷及一点点思考](https://juejin.cn/post/6932277268110639112)
- [一文读懂 Java 和 Kotlin 的泛型难点](https://juejin.cn/post/6935322686943920159)
- [一文快速入门 Kotlin 协程](https://juejin.cn/post/6908271959381901325)
- [一文快速入门 ConstraintLayout](https://juejin.cn/post/6911710012750430215)