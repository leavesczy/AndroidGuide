Activity 通过 Window 与 View 系统进行交互，Window 再通过 ViewRootImpl 与 DecorView（视图树的根布局） 进行交互。平时我们都是通过 `setContentView` 方法来指定 Activity 要承载的具体布局文件，布局文件中可能包含多个 ViewGroup 和 View，Activity 势必需要将所有 View 绘制到屏幕上，整个绘制流程就分为 measure、layout、draw 三大步骤。作为应用层的开发者，我们很多时候都需要通过实现自定义 View 来满足一些产品需求，这就要求我们对 View 的绘制流程有一个比较清晰的认知，本文就来介绍下这整个流程是如何流转的，希望对你有所帮助 🤣🤣

本文基于 Android API 30 进行分析

# 一、Measure

measure 代表的是**测量尺寸**的过程，在这个过程中 View 需要计算出自己的宽高大小

## 1、MeasureSpec

我们知道，一个 View 想要显示在屏幕上，那么其自身必然就要带有宽高属性，即**尺寸大小 size**，而 size 的设定依据可能来源于不同的约束条件：ViewGroup 允许的最大空间、布局文件中为 View 指定了特定宽高、将 View 的宽高设定为 match_parent 或者 wrap_content 等等。有了 size 后，相对应的我们就需要能够分辨出不同的约束条件，比如说 View 得到的 widthSize 是 100dp，这可能是因为父容器只有 100dp 且 View 使用了 match_parent，也可能是我们在布局文件中为该 View 直接指定了宽度就是 100dp，View 需要知道这种区别，这是 View 在测量自身尺寸的依据之一，即我们也需要拿到**测量模式 mode**

MeasureSpec 就用来封装 View 的 **size** 和 **mode** 这两个属性，它是 View 的一个静态内部类，用一个 int 类型的三十二位整数来表示这两个属性，前两位表示 mode，后三十位表示 size。通过单个整数来表示两个属性值并通过位运算来进行拆分可以更加节省内存空间。两个二进制位足够表示四种可能值，实际上 View 只用到了三种：UNSPECIFIED、EXACTLY、AT_MOST。`makeMeasureSpec` 方法就用于打包封装 size 和 mode 这两个属性值来生成 MeasureSpec

```java
public static class MeasureSpec {

    public static final int UNSPECIFIED = 0 << MODE_SHIFT;

    public static final int EXACTLY     = 1 << MODE_SHIFT;

    public static final int AT_MOST     = 2 << MODE_SHIFT;

	public static int makeMeasureSpec(int size, int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }

    ···

}
```

三种测量模式的含义：

| mode        | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| UNSPECIFIED | ViewGroup 对于 View 没有任何限制，View 可以拿到任意想要的 size |
| EXACTLY     | View 本身设定了确切大小的 size。例如，View 的宽度设置为了 match_parent 或者具体的 dp 值，match_parent 即占满父容器，对于 View 来说也属于精准值 |
| AT_MOST     | size 是 View 能够占据的最大空间，View 的最终大小不能超出该范围。对应 wrap_content，View 可以在父容器可以容纳的范围内申请空间 |

在进行自定义 View 的时候，系统会自动构造出 MeasureSpec 对象并回调给 View 的 `onMeasure(int widthMeasureSpec, int heightMeasureSpec)`方法，在此方法中我们就需要根据实际情况来计算出 View 应该且可以占有的尺寸值

## 2、LayoutParams

LayoutParams 是 ViewGroup 的一个静态内部类，包含了 View 的两个最基础属性： width 和 height，默认只会解析我们在布局文件中设置的 `layout_width` 和 `layout_height`这两个属性

```java
public static class LayoutParams {

    @Deprecated
    public static final int FILL_PARENT = -1;

    public static final int MATCH_PARENT = -1;

    public static final int WRAP_CONTENT = -2;

    public int width;

    public int height;

    //从布局文件中解析宽高值
    public LayoutParams(Context c, AttributeSet attrs) {
        TypedArray a = c.obtainStyledAttributes(attrs, R.styleable.ViewGroup_Layout);
        setBaseAttributes(a,
                R.styleable.ViewGroup_Layout_layout_width,
                R.styleable.ViewGroup_Layout_layout_height);
        a.recycle();
    }

    //外部直接指定宽高值
    public LayoutParams(int width, int height) {
        this.width = width;
        this.height = height;
    }

    ···

}
```

View 的 MeasureSpec 就由其父容器 ViewGroup 的 MeasureSpec 和 View 自身的 LayoutParams 来共同决定

View 能够占据的尺寸大小肯定是要受其父容器 ViewGroup 的影响，一般情况下 View 的尺寸是不会超出 ViewGroup 本身可以容纳的范围的，毕竟超出的话 View 也会显示不全，除非说 ViewGroup 本身是支持滑动的，例如 ScrollView

LayoutParams 则代表的是 View 本身的尺寸属性和布局属性，例如 width、height、margin 等，我们在布局文件中为 View 设置的 `layout_width="match_parent"` 和 `layout_marginStart="@dimen/DIMEN_32PX"` 等最终就都会转换为 View 内的 LayoutParams 对象，这也是 MeasureSpec 的生成依据之一

## 3、ViewRootImpl  &  View

我们知道，DecorView 是整个视图树的根布局，而 DecorView 是 FrameLayout 的子类，所以说平时我们在 Activity 中 `setContentView` 其实就是在向 DecorView 执行 `addView` 操作。很自然地，整个视图树的测量过程就是要从 DecorView 开始，从上到下从外到内进行，DecorView 的尺寸大小就是整个视图树所能占据的最大空间，而 DecorView 的宽高默认都是 `match_parent`，即占据整个屏幕空间

View 的整个绘制流程的启动入口可以从 ViewRootImpl 的 `performTraversals` 方法开始看，`performTraversals` 方法逻辑挺复杂的，主要就用于为 DecorView 生成 MeasureSpec，我们只看其主干逻辑即可。`mWidth` 和 `mHeight` 即屏幕的宽高，`lp.width` 和 `lp.height` 即 DecorView 的宽高，由于可见最终 `childWidthMeasureSpec` 和 `childHeightMeasureSpec` 的 mode 都将是 EXACTLY。最后又会调用 `performMeasure` 方法来启动整个视图树的测量流程，当中的 mView 代表的即是 DecorView

```java
private void performTraversals() {
    ···
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ···
}

private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        //启动测量流程
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

`measure` 方法是 View 类中的一个 final 方法，这说明不管是 ViewGroup 还是 View 的其它子类都无法重写该方法，`measure` 方法在完成一些通用逻辑后就会去调用 `onMeasure` 方法。`onMeasure` 方法就是我们在进行自定义 View 时需要重写的方法之一，我们需要在这里为自定义 View 完成自身尺寸的测量逻辑，对于 ViewGroup 来说除了需要测量自身外还需要测量所有 childView

在默认情况下，`onMeasure`方法会综合考虑 View 自身是否设置了 minWidth 和 background，取这两者的最大宽高值 maxSize 作为 View 最终尺寸的参考依据之一。如果 specMode 是 UNSPECIFIED，则最终值为 maxSize。如果 specMode 是 AT_MOST 或者 EXACTLY，则最终值为 specSize。这说明在默认情况下 View 并没有区分处理 **`match_parent`、`wrap_content` 和指定了明确尺寸**这三种情况，在自定义 View 的时候就需要我们来主动区分处理不同情况

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}

public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

`setMeasuredDimension`也是一个 final 方法，不管是 View 还是 ViewGroup，**在将宽高测量出来后都需要将测量结果传递给该方法**。需要注意的是，**此时 measuredWidth 和 measuredHeight 并不一定就等于 View 最终的 width 和 height**，View 的 width 和 height 还需要在 layout 阶段才能最终确定下来。而且从 `setMeasuredDimension` 方法的逻辑就可以看出来，此时只是将测量结果保存到了 `mMeasuredWidth` 和 `mMeasuredHeight` 上，即平时我们调用 `getMeasuredWidth()` 和 `getMeasuredHeight()`方法只是拿到了 View 在 measure 阶段的测量结果，并不一定就等于 View 的最终宽高

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;
        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;
    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

## 4、ViewGroup

DecorView 是 FrameLayout 的子类，实际上就是一个 ViewGroup，所以说平时我们在 Activity 中 `setContentView` 其实就是在向 ViewGroup 执行 `addView`。这里就再来分析下 ViewGroup 的 measure 过程

上文说了，ViewRootImpl 通过调用 DecorView 的`measure`方法来启动整个视图树的测量流程，之后又会调用`onMeasure`方法。对于 FrameLayout 来说，其自身可能会包含多个 childView，那么在 measure 阶段就需要在进行自身的测量操作之前先完成所有 childView 的测量操作。而 DecorView 直接继承于 FrameLayout  并重写了 `onMeasure` 方法，DecorView 增加了一些修正操作，当判断到 widthMode 和 heightMode 为 AT_MOST 时，就会尝试去将 mode 修正为 EXACTLY 并修改 size 大小，生成新的 `widthMeasureSpec` 和 `heightMeasureSpec`，并调用 `super.onMeasure` 将实际的测量操作交由 FrameLayout 去完成

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    final DisplayMetrics metrics = getContext().getResources().getDisplayMetrics();
    final boolean isPortrait =
            getResources().getConfiguration().orientation == ORIENTATION_PORTRAIT;

    final int widthMode = getMode(widthMeasureSpec);
    final int heightMode = getMode(heightMeasureSpec);

    boolean fixedWidth = false;
    mApplyFloatingHorizontalInsets = false;
    if (widthMode == AT_MOST) {
        final TypedValue tvw = isPortrait ? mWindow.mFixedWidthMinor : mWindow.mFixedWidthMajor;
        if (tvw != null && tvw.type != TypedValue.TYPE_NULL) {
            final int w;
            if (tvw.type == TypedValue.TYPE_DIMENSION) {
                w = (int) tvw.getDimension(metrics);
            } else if (tvw.type == TypedValue.TYPE_FRACTION) {
                w = (int) tvw.getFraction(metrics.widthPixels, metrics.widthPixels);
            } else {
                w = 0;
            }
            if (DEBUG_MEASURE) Log.d(mLogTag, "Fixed width: " + w);
            final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
            if (w > 0) {
                widthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        Math.min(w, widthSize), EXACTLY);
                fixedWidth = true;
            } else {
                widthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        widthSize - mFloatingInsets.left - mFloatingInsets.right,
                        AT_MOST);
                mApplyFloatingHorizontalInsets = true;
            }
        }
    }

    mApplyFloatingVerticalInsets = false;
    if (heightMode == AT_MOST) {
        //省略和 widthMode 相同操作的代码
    }

    //将实际的测量操作交由 FrameLayout 去完成
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    int width = getMeasuredWidth();
    boolean measure = false;

    widthMeasureSpec = MeasureSpec.makeMeasureSpec(width, EXACTLY);

    if (!fixedWidth && widthMode == AT_MOST) {
        final TypedValue tv = isPortrait ? mWindow.mMinWidthMinor : mWindow.mMinWidthMajor;
        if (tv.type != TypedValue.TYPE_NULL) {
            final int min;
            if (tv.type == TypedValue.TYPE_DIMENSION) {
                min = (int)tv.getDimension(metrics);
            } else if (tv.type == TypedValue.TYPE_FRACTION) {
                min = (int)tv.getFraction(mAvailableWidth, mAvailableWidth);
            } else {
                min = 0;
            }
            if (DEBUG_MEASURE) Log.d(mLogTag, "Adjust for min width: " + min + ", value::"
                    + tv.coerceToString() + ", mAvailableWidth=" + mAvailableWidth);

            if (width < min) {
                widthMeasureSpec = MeasureSpec.makeMeasureSpec(min, EXACTLY);
                measure = true;
            }
        }
    }

    // TODO: Support height?

    if (measure) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }
}
```

FrameLayout 自然是会重写 View 的 `onMeasure`方法，不同 ViewGroup 有不同的布局效果，因此宽高属性的计算规则肯定也是不一样的，这个差异就体现在`onMeasure`方法中 。FrameLayout 的布局特点就是会将所有的 childView 进行叠加覆盖显示，因此 FrameLayout 的宽高主要是受尺寸最大的 childView 影响，其它因素还包括 padding、minimumWidth、minimumHeight、foreground 等。此外，如果我们为 FrameLayout 直接设定了准确的宽高值，例如 match_parent 或者 200dp 这类值的话，那么不管 childView 尺寸多大，FrameLayout 也只能以该精准值来完成测量过程了

来具体看下 FrameLayout 的`onMeasure`方法，逻辑上可以分为两步

第一步 FrameLayout 会去遍历所有 childView，触发其 measure 操作，获取每个 childView  的 measuredWidth 和 measuredHeight 的最大值，同时还需要考虑 childViewMargin、FrameLayout 自身 padding、minimumSzie 等多个参数的限制，从而得到 maxWidth 和 maxHeight，最终还需要考虑这两个值是否符合 MeasureSpec 的限制规则，因为很有可能存在一种情况：FrameLayout 自身只有 100 x 100 的空间，而 childView 要求的是 200 x 200，那么此时 FrameLayout 也只能按照 100 x 100 的规格来调用 `setMeasuredDimension`方法

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();

    //FrameLayout 的 layout_width 或 layout_height 是否设置了 wrap_content
    final boolean measureMatchParentChildren =
            MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
            MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;

    //遍历 measure 所有 childView
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        //mMeasureAllChildren 默认为 false，不予理会
        //childView 只有不为 GONE 才需要 measure
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            //去完成 childView 的 measure 操作
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            //保存所有 childView 中的最大宽高值
            maxWidth = Math.max(maxWidth,
                    child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                    child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                //第二步逻辑需要用到
                if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);
                }
            }
        }
    }

    // Account for padding too
    //需要考虑 padding 和 ForegroundPadding
    maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
    maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

    // Check against our minimum height and width
    //需要考虑是否达到了设定的 Minimum 要求
    maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    // Check against our foreground's minimum height and width
    //需要考虑是否达到了 Foreground 的 Minimum 要求
    final Drawable drawable = getForeground();
    if (drawable != null) {
        maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
        maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
    }

    //resolveSizeAndState 会根据 specMode 的类型来决定是选择 maxSize 还是 specSize
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));
    ···
}
```

第二步 FrameLayout 还需要考虑一种比较特殊的情况：假设 FrameLayout 的 `layout_width` 设置为 `wrap_content`，而某个 childView 的 `layout_width` 设置为 `match_parent`。此时对于 FrameLayout 来说其宽度并没有确定值，需要依靠所有 childView 来决定。对于该 childView 来说，其希望的是宽度占满整个 FrameLayout。所以此时该 childView 的`widthSpecSize`就应该是 FrameLayout 当前的 `widthMeasureSize`，`widthSpecMode` 应该是 `EXACTLY`才对。而 FrameLayout 也只有在完成所有 childView 的 measure 操作后才能得到自己的`widthMeasureSize`，所以第二步逻辑就是来进行补救措施，判断是否需要让 childView 进行第二次 measure

此外，这里也只需要在有大于一个的 childView 存在上述情况时才需要处理，因为在执行第一步逻辑的时候该 childView 的`widthSpecSize`就已经等于 FrameLayout 的`widthSpecSize`了，虽然`widthSpecMode`是 AT_MOST，但此时该 childView 已经可以拿到其能占据的最大空间了。而如果存在多个 childView 的话就需要统一再来给它们一个相同的 `widthMeasureSize`

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();

    //FrameLayout 的 layout_width 或 layout_height 是否设置了 wrap_content
    final boolean measureMatchParentChildren =
            MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
            MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();

    ···

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        ···
        if (measureMatchParentChildren) {
              if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);
                }
            }
        ···
    }

    ···

    count = mMatchParentChildren.size();
    if (count > 1) { //只有 measureMatchParentChildren 为 true 才会走进这里
        for (int i = 0; i < count; i++) {
            final View child = mMatchParentChildren.get(i);
            final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

            final int childWidthMeasureSpec;
            if (lp.width == LayoutParams.MATCH_PARENT) {
                final int width = Math.max(0, getMeasuredWidth()
                        - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                        - lp.leftMargin - lp.rightMargin);
                childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        width, MeasureSpec.EXACTLY);
            } else {
                childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                        getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                        lp.leftMargin + lp.rightMargin,
                        lp.width);
            }

            final int childHeightMeasureSpec;
            if (lp.height == LayoutParams.MATCH_PARENT) {
                final int height = Math.max(0, getMeasuredHeight()
                        - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                        - lp.topMargin - lp.bottomMargin);
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        height, MeasureSpec.EXACTLY);
            } else {
                childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                        getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                        lp.topMargin + lp.bottomMargin,
                        lp.height);
            }
           //重新 measure
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

## 5、ParentMeasureSpec  &  LayoutParams

对于 DecorView 来说，其 MeasureSpec 是通过测量屏幕宽高来生成的，这从 ViewRootImpl 的 `performTraversals()` 方法就可以体现出来

```java
private void performTraversals() {
    ···
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ···
}
```

**而对于 View 来说，其 MeasureSpec 是由其父容器 ViewGroup 的 MeasureSpec 和 View 自身的 LayoutParams 来共同决定的**。此处所说的 View 也包含 ViewGroup 类型，因为父容器 ViewGroup 在测量 childView 的时候并不关心下一级的具体类型，而只是负责下发测量要求并接收测量结果，下一级如果是 View 类型那么就只需要测量自身并返回结果，下一级如果是 ViewGroup 类型那么就重复以上步骤并返回结果，整个视图树的绘制流程就通过这种层层调用的方式来完成测量，和 View 的事件分发机制非常相似

以上结论也可以通过看 FrameLayout 的 `onMeasure` 方法来进行验证 

在该方法中，FrameLayout 通过 `measureChildWithMargins`方法来执行 childView 的 measure 流程，将 childView 的测量结果作为测量自身的依据之一，这里就用到了 FrameLayout 自身的 `widthMeasureSpec` 和 `heightMeasureSpec`

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();
    ···
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            //去测量 childView
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            maxWidth = Math.max(maxWidth,
                    child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                    child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);
                }
            }
        }
    }
    ···
}
```

`measureChildWithMargins`从名字上就可以看出在生成 childView 的 MeasureSpec 的时候会同时考虑 childView 是否设置了 margin，实际上还会用上 ViewGroup 的 padding 值。ViewGroup 必须先减去这两个属性值所占据的空间，剩余的空间才能用来容纳 childView。可以看到，**此时已经使用到 childView 的 LayoutParams 了**

```java
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, 
                                       int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);
    //去 measure View 
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

`getChildMeasureSpec` 方法是 **View 的 MeasureSpec 是由其父容器 ViewGroup 的 MeasureSpec 和 View 自身的 LayoutParams 来共同决定的** 这句话最直接的体现。spec 即 ViewGroup 的 MeasureSpec，childDimension 是 View 在布局文件中声明的尺寸值，padding 是 ViewGroup 加上 View 总的间距值，ViewGroup  和 View 本身共同决定了 View 的 MeasureSpec

例如，假设 ViewGroup 的 `layout_width` 是 `match_parent`，childView 的 `layout_width` 是 `wrap_content`，那么 childView 的宽度最多只能占满 ViewGroup 而不应该超出该范围。在这个设定下，ViewGroup 的`specMode`就是`EXACTLY`，`resultSize` 就等于`size`，`resultMode` 就是 `AT_MOST`，即 childView 最终的测量结果不得超出 size

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    //父容器的空间需要先减去 padding 后才能用来容纳 childView
    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    ···
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

# 二、Layout

layout 代表的是**确定位置**的过程，在这个过程中 View 需要计算得出自己在父容器中的显示位置

## 1、ViewRootImpl

View 的 layout 起始点也是从 ViewRootImpl 开始的，ViewRootImpl 的 `performLayout` 方法会调用 DecorView 的 `layout` 方法来启动 layout 流程，传入的后两个参数即屏幕的宽高大小

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth, int desiredWindowHeight) {
    mScrollMayChange = true;
    mInLayout = true;

    final View host = mView;
    if (host == null) {
        return;
    }
    if (DEBUG_ORIENTATION || DEBUG_LAYOUT) {
        Log.v(mTag, "Laying out " + host + " to (" +
                host.getMeasuredWidth() + ", " + host.getMeasuredHeight() + ")");
    }

    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
    try {
        //启动 layout 流程
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
       ···
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    mInLayout = false;
}
```

## 2、View

`layout` 是 View 类中的方法，传入的四个参数即我们熟知的 left、top、right、bottom，这四个值都是 View 相对父容器 ViewGroup 的坐标值。对于 DecorView 来说这四个值就分别是 0、0、screenWidth、screenHeight

```java
public void layout(int l, int t, int r, int b) {
    ···
    //重点
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        //重点
        onLayout(changed, l, t, r, b);
        ···
    }
    ···
}

protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```

`setFrame` 方法又会将 left、top、right、bottom 等四个值保存到 View 相应的几个全局变量上，至此 View 的 width 和 height 才真正确定下来，View 的 `getWidth()` 和 `getHeight()`方法都是依靠这四个值做减法运算得到的。此外，这里也会回调 `onSizeChanged` 方法，在自定义 View 时我们往往就通过该方法来得到 View 的准确宽高大小，并在这里接收宽高大小变化的通知

```java
protected boolean setFrame(int left, int top, int right, int bottom) {
    ···
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;
       ···

        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;

        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
        mPrivateFlags |= PFLAG_HAS_BOUNDS;

        if (sizeChanged) {
            sizeChange(newWidth, newHeight, oldWidth, oldHeight);
        }
        ···
    }
    return changed;
}

private void sizeChange(int newWidth, int newHeight, int oldWidth, int oldHeight) {
    onSizeChanged(newWidth, newHeight, oldWidth, oldHeight);
    ···
}

public final int getWidth() {
    return mRight - mLeft;
}

public final int getHeight() {
    return mBottom - mTop;
}
```

## 3、ViewGroup

`layout` 方法又会调用自身的 `onLayout` 方法。`onLayout` 方法在 View 类中是空实现，大部分情况下 View 都无需重写该方法。而 ViewGroup 又将其改为了抽象方法，即每个 ViewGroup 子类都需要通过实现该方法来管理自己的所有 childView 的摆放位置，FrameLayout 和 LinearLayout 等容器类就通过实现该方法来实现不同的布局效果

还是以 FrameLayout 为例子。FrameLayout 的布局特点就是会将所有的 childView 进行叠加覆盖显示，每个 childView 之间并不会形成相互约束，childView 主要是通过 `layout_gravity` 和 `layout_margin` 来声明自己在 FrameLayout 中的位置。FrameLayout 的 `padding` 也会占据一部分空间，从而影响 childView 的可用空间

FrameLayout 的 `layoutChildren` 方法就需要考虑以上因素，计算得出 childView 相对 FrameLayout 的 left、top、right、bottom 等值的大小，然后调用 childView 的 `layout` 方法，使得 childView 能够得到自己的真实宽高。如果 childView 也属于 ViewGroup 类型的话，就又会层层调用重复以上步骤完成整个视图树的 layout 操作

```java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false /* no force left gravity */);
}

void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
    final int count = getChildCount();

    final int parentLeft = getPaddingLeftWithForeground();
    final int parentRight = right - left - getPaddingRightWithForeground();

    final int parentTop = getPaddingTopWithForeground();
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            final int width = child.getMeasuredWidth();
            final int height = child.getMeasuredHeight();

            int childLeft;
            int childTop;

            int gravity = lp.gravity;
            if (gravity == -1) {
                gravity = DEFAULT_CHILD_GRAVITY;
            }

            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

            //考虑水平方向上的约束条件
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                    lp.leftMargin - lp.rightMargin;
                    break;
                case Gravity.RIGHT:
                    if (!forceLeftGravity) {
                        childLeft = parentRight - width - lp.rightMargin;
                        break;
                    }
                case Gravity.LEFT:
                default:
                    childLeft = parentLeft + lp.leftMargin;
            }

            //考虑竖直方向上的约束条件
            switch (verticalGravity) {
                case Gravity.TOP:
                    childTop = parentTop + lp.topMargin;
                    break;
                case Gravity.CENTER_VERTICAL:
                    childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                    lp.topMargin - lp.bottomMargin;
                    break;
                case Gravity.BOTTOM:
                    childTop = parentBottom - height - lp.bottomMargin;
                    break;
                default:
                    childTop = parentTop + lp.topMargin;
            }

            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}
```

# 三、Draw

draw 代表的是**绘制视图**的过程，在这个过程中 View 需要通过操作 Canvas 来实现自己 UI 效果

View 的 draw 起始点也是从 ViewRootImpl 开始的，ViewRootImpl 的 `performDraw` 方法会调用 `drawSoftware` 方法，再通过调用 DecorView 的 `draw` 方法来启动 draw 流程

```java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
    // Draw with software renderer.
    final Canvas canvas;   
    ···
    mView.draw(canvas);
    ···
}
```

View 的`draw`方法的重点看其调用的 `onDraw` 和 `dispatchDraw` 这两个方法即可，这两个方法在 View 类中都是空实现

- 对于自定义 View，我们需要重写`onDraw`方法来实现自己的特定 UI，无需关心`dispatchDraw`方法
- 对于 ViewGroup，除了需要绘制背景色前景色等外，无需绘制自身，所以 ViewGroup 无需重写`onDraw`方法。而 `dispatchDraw`方法就是为 ViewGroup 准备的，用于向所有 childView 下发 draw 请求

```java
public void draw(Canvas canvas) {
            ···
    // Step 3, draw the content
    onDraw(canvas);
    // Step 4, draw the children
    dispatchDraw(canvas);
    ···
    }
```

ViewGroup 的 `dispatchDraw` 方法会循环遍历所有 childView，使用同个 Canvas 对象来调用每个 childView 的 `draw`方法，层层调用完成整个视图树的绘制

```java
@Override
protected void dispatchDraw(Canvas canvas) {
    ···
    for (int i = 0; i < childrenCount; i++) {
        while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                //重点
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            transientIndex++;
            if (transientIndex >= transientCount) {
                transientIndex = -1;
            }
        }

        final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
        final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
            //重点
            more |= drawChild(canvas, child, drawingTime);
        }
    }
    ···
}

protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```

# 四、提问环节

## 1、ViewGroup 和 View 的绘制顺序

ViewGroup 和 View 在进行 measure、layout、draw 时是交叉在一起的，那么这两者具体的先后顺序是怎么样的呢？

先说结论：

- measure 阶段是先 View 后 ViewGroup
- layout 阶段是先 ViewGroup 后 View
- draw 阶段是先 ViewGroup 后 View

以 FrameLayout 为例，其 `onMeasure` 方法就需要先去完成所有 childView 的 measure 操作，得到 maxWidth 和 maxHeight 后才能确定自己的尺寸值

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();

    ···

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;

    //遍历 measure 所有 childView
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            //去完成 childView 的 measure 操作
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            //保存所有 childView 中的最大宽高值
            maxWidth = Math.max(maxWidth,
                    child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                    child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);
                }
            }
        }
    }

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));
    ···
}
```

在 layout 阶段，FrameLayout 的 `setFrame` 方法已经将外部传入的 left、top、right、bottom 等四个值保存起来了，至此 ViewGroup 自身的位置信息就已经确定下来了，之后才会调用 `layoutChildren` 方法去执行 childView 的 layout 操作

```java
public void layout(int l, int t, int r, int b) {
    ···
    //重点
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        //重点
        onLayout(changed, l, t, r, b);
        ···
    }
    ···
}

@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false /* no force left gravity */);
}
```

在 draw 阶段，FrameLayout 也是先执行自己的 `onDraw` 方法后，再去执行 `dispatchDraw` 方法，这也说明 ViewGroup 是先完成自身的绘制需求后才去绘制 childView，毕竟 ViewGroup 的视图显示层次要比 View 低

```java
public void draw(Canvas canvas) {
	···
    // Step 3, draw the content
    onDraw(canvas);
    // Step 4, draw the children
    dispatchDraw(canvas);
    ···
}
```

## 2、View 多个回调函数的先后顺序

View 开放给子类重写的回调方法有很多个，我们经常使用的到的有以下几个

```java
@Override
protected void onAttachedToWindow() {
    super.onAttachedToWindow();
}

@Override
protected void onDetachedFromWindow() {
    super.onDetachedFromWindow();
}

@Override
protected void onVisibilityChanged(@NonNull View changedView, int visibility) {
    super.onVisibilityChanged(changedView, visibility);
}

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
}

@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);
}

@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
}
```

从上文分析就可以得知，View 的整个绘制流程就是按照 `onMeasure`、`onLayout`、`onDraw` 的顺序来进行调用。那其它三个方法呢？

先了解下这三个方法的作用：

- onAttachedToWindow。当 View 附着到 Window 上时就会被回调，代表着 View 生命周期的开始，在 View 的整个生命周期中只会被调用一次，适合在该方法内做一些初始化操作
- onDetachedFromWindow。当 View 从 Window 上移除时就会被回调，代表着 View 生命周期的结束，在 View 的整个生命周期中只会被调用一次，适合在该方法内做一些资源回收操作
- onVisibilityChanged。当 View 的可见性发生变化时就会被回调，在 View 的整个生命周期中至少会被调用一次，可能会被调用多次。使用场景之一就是当 View 不可见的时候就在该方法内部暂停动画，避免资源浪费

这五个方法的调用先后顺序还需要从 ViewRootImpl 的 `performTraversals()` 方法开始看起，当判断到当前是首次绘制视图树时，就会调用 DecorView 的 `dispatchAttachedToWindow` 方法向自身及所有 childView 下发 AttachedToWindow 的事件通知，且该方法会在 `performMeasure` 之前调用，说明 AttachedToWindow 操作发生在 measure 操作之前

```java
private void performTraversals() {
    ···
    if (mFirst) {
        ···
        host.dispatchAttachedToWindow(mAttachInfo, 0);
    	···
    }
    ···
	childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
	childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
	performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ···
}
```

`dispatchAttachedToWindow` 方法是 View 类中的方法，参数 0 即 `View.VISIBLE`，即 DecorView 默认状态下就是可见的。该方法内部就会回调 `onAttachedToWindow` 和 `onVisibilityChanged` 这两个方法

```java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    ···
    onAttachedToWindow();
    ···
    onVisibilityChanged(this, visibility);
    ···
}
```

而 ViewGroup 重写了该方法，ViewGroup 会先调用自身再调用 childView 的 `dispatchAttachedToWindow`方法，这说明 ViewGroup 和内嵌的 View 之间具有明确的先后顺序。DecorView 就通过这种层层调用来执行内嵌 View 的 `dispatchAttachedToWindow`方法

```java
@Override
@UnsupportedAppUsage
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mGroupFlags |= FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;
    super.dispatchAttachedToWindow(info, visibility);
    mGroupFlags &= ~FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;

    final int count = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < count; i++) {
        final View child = children[i];
        child.dispatchAttachedToWindow(info,
                combineVisibility(visibility, child.getVisibility()));
    }
    final int transientCount = mTransientIndices == null ? 0 : mTransientIndices.size();
    for (int i = 0; i < transientCount; ++i) {
        View view = mTransientViews.get(i);
        view.dispatchAttachedToWindow(info,
                combineVisibility(visibility, view.getVisibility()));
    }
}
```

ViewRootImpl 的 `dispatchDetachedFromWindow()` 方法又负责调用 DecorView 的 `dispatchDetachedFromWindow` 方法

```java
void dispatchDetachedFromWindow() {
    ···
    if (mView != null && mView.mAttachInfo != null) {
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
        mView.dispatchDetachedFromWindow();
    }
    ···
}
```

View 收到该回调后就会再回调 `onDetachedFromWindow` 方法 

```java
@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
void dispatchDetachedFromWindow() {
    ···
    onDetachedFromWindow();
    ···
}
```

ViewGroup 则是会先调用 childView 再调用自身的 `dispatchDetachedFromWindow` 方法

```java
@Override
@UnsupportedAppUsage
void dispatchDetachedFromWindow() {
    ···
    final int count = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < count; i++) {
        children[i].dispatchDetachedFromWindow();
    }
    clearDisappearingChildren();
    final int transientCount = mTransientViews == null ? 0 : mTransientIndices.size();
    for (int i = 0; i < transientCount; ++i) {
        View view = mTransientViews.get(i);
        view.dispatchDetachedFromWindow();
    }
    super.dispatchDetachedFromWindow();
}
```

所以说，Activity 从开始展示到退出页面这个过程，View 的这五个方法的先后顺序是：

1. 父容器的 onAttachedToWindow 和 onVisibilityChanged 会先后调用，之后才会调用 childView 的这两个方法
2. childView 的 onDetachedFromWindow 会先被调用，所有 childView 都调用后才会调用父容器的该方法
3. View 的绘制流程就按照 onMeasure、onLayout、onLayout 的顺序进行，onAttachedToWindow 和 onVisibilityChanged 都会早于这三个方法

## 3、getWidth 和 getMeasuredWidth 的区别

`getMeasuredWidth()` 和 `getMeasuredHeight()`返回的是 View 在 measure 阶段的测量结果，用于在 `onMeasure` 方法后调用。`getWidth()` 和 `getHeight()`返回的是 View 的实际宽高值，用于在 `onLayout` 方法后调用。这两者可以说是 View 在不同阶段下的一个尺寸值，大多数情况下这两个值都是相等的，但 measureWidth 只是相当于一个预估值，View 的最终宽度并不一定遵循该值，View 的实际宽高需要在 layout 阶段才能最终确定下来

例如，我们完全可以通过重写 `layout` 方法来使得 View 的位置发生偏移，这就可以使得 View 的 width 和 measureWidth 两者不相等

```java
@Override
public void layout(int l, int t, int r, int b) {
    super.layout(l, t, r + 10, b + 10);
}
```

## 4、LayoutParams 在什么时候生成

我们知道，当我们在 Activity 中调用 `setContentView` 后，系统肯定是需要从布局文件中为每个 View 和 ViewGroup 构造生成对应的 LayoutParams，那这个过程具体是在什么时候呢？下面就来了解下

如果我们使用的 Activity 是继承于 AppCompatActivity，那么 `setContentView` 方法就会交由代理类 AppCompatDelegateImpl 来完成。contentParent 即 DecorView 中除了标题栏之外的内容区域，即最后我们的布局文件要挂载的地方，布局文件所代表的 View 其实就是通过 `ViewGroup.addView`的方式添加到 contentParent 中。重点看 LayoutInflater

```java
@Override
public void setContentView(int resId) {
    ensureSubDecor();
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    //重点
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
```

我们平时想要将布局文件映射为 View 对象时就需要用到 LayoutInflater，Activity 这里也是一样。LayoutInflater 的 `inflate` 方法从逻辑上看似乎是有两种解析布局文件的方式：预编译 + xml 解析，这里重点看 xml 解析的方式即可

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}

public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
              + Integer.toHexString(resource) + ")");
    }

    //似乎是和预编译有关？不了解，略过
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }

    //重点，通过解析 xml 文件 + 反射来实例化 View
    XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

以下的 `inflate` 方法的逻辑可以总结为：

1. 通过 `LayoutInflater.Factory` 或者反射来实例化布局文件中的根布局 View 所代表的对象，即 temp
2. 通过 root（即 contentParent ）的 `generateLayoutParams` 方法为 temp 生成 LayoutParams 对象，attrs 即我们在布局文件中为 temp 声明的各个标签属性，这样生成的 LayoutParams 才能对应上 temp 的宽高值
3. 进一步解析 temp 内嵌的所有 childView，实例化 childView 并赋予 LayoutParams，并一一将其通过 addView 添加到 temp 中
4. 对于 Activity 来说，root 肯定不为 null，attachToRoot 也肯定为 true，因此最终 `root.addView(temp, params)`方法就会将 temp 添加到 root 的视图树中，从而将我们声明的布局展示到 Activity 中

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    ···

    // Temp is the root view that was found in the xml
    //第一步
    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

    ViewGroup.LayoutParams params = null;

    if (root != null) {
        if (DEBUG) {
            System.out.println("Creating params from root: " +
                    root);
        }
        // Create layout params that match root, if supplied
        //第二步
        params = root.generateLayoutParams(attrs);
        if (!attachToRoot) {
            // Set the layout params for temp if we are not
            // attaching. (If we are, we use addView, below)
            temp.setLayoutParams(params);
        }
    }

    if (DEBUG) {
        System.out.println("-----> start inflating children");
    }

    // Inflate all children under temp against its context.
    //第三步
    rInflateChildren(parser, temp, attrs, true);

    if (DEBUG) {
        System.out.println("-----> done inflating children");
    }

    // We are supposed to attach all the views we found (int temp)
    // to root. Do that now.
    //第四步
    if (root != null && attachToRoot) {
        root.addView(temp, params);
    }

    ···
}
```

 temp 内嵌的所有 childView 既可能是 View 类型也可能是 ViewGroup 类型，而 childView 也可能包含多个 View 和 ViewGroup ，每个 childView 都需要通过 `addView` 的方式添加到其父容器 ViewGroup 中。因此`rInflateChildren`方法使用到了递归的思想来实现这个目的，它调用了 `rInflate` 方法，`rInflate` 方法又调用回`rInflateChildren` 方法

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}

void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;

    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();

        if (TAG_REQUEST_FOCUS.equals(name)) {
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
            throw new InflateException("<merge /> must be the root element");
        } else {
            //实例化 name 所代表的 View 对象
            final View view = createViewFromTag(parent, name, context, attrs);
            //通过 view 的父容器来获取 LayoutParams 对象
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            //通过递归调用来实例化 view 的所有 childView 并添加到 view 中
            rInflateChildren(parser, view, attrs, true);
            //将 view 添加到 viewGroup 中
            viewGroup.addView(view, params);
        }
    }

    if (pendingRequestFocus) {
        parent.restoreDefaultFocus();
    }

    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```

可以看出来，每个 View 和 ViewGroup 都需要通过其直接父容器 ViewGroup 的 `generateLayoutParams` 方法来生成 LayoutParams 对象。在默认情况下，ViewGroup 的 `generateLayoutParams` 方法返回的都是 LayoutParams 这个父类型，AttributeSet 中保存的即是我们在布局文件中为 childView 声明各个标签值

```java
public LayoutParams generateLayoutParams(AttributeSet attrs) {
    return new LayoutParams(getContext(), attrs);
}
```

对于 FrameLayout 来说，它返回的是一个自定义的 LayoutParams 对象，且继承的是 MarginLayoutParams，MarginLayoutParams 多提供了解析 `layout_margin`的能力，`FrameLayout.LayoutParams` 进一步提供了解析 `layout_gravity` 的能力

```java
@Override
public LayoutParams generateLayoutParams(AttributeSet attrs) {
    return new FrameLayout.LayoutParams(getContext(), attrs);
}

public static class LayoutParams extends MarginLayoutParams {

public static final int UNSPECIFIED_GRAVITY = -1;

public int gravity = UNSPECIFIED_GRAVITY;

public LayoutParams(@NonNull Context c, @Nullable AttributeSet attrs) {
    super(c, attrs);
    final TypedArray a = c.obtainStyledAttributes(attrs, R.styleable.FrameLayout_Layout);
    gravity = a.getInt(R.styleable.FrameLayout_Layout_layout_gravity, UNSPECIFIED_GRAVITY);
        a.recycle();
    }

   ···
}
```

所以说，每个 View 或者 ViewGroup 都需要通过其父容器来生成 LayoutParams，且每个 ViewGroup 子类返回的 LayoutParams 一般来说都需要继承于 MarginLayoutParams，这样才能具备解析 `layout_margin`的能力，且还需要再根据自身 ViewGroup 提供的标签属性来进一步扩展 MarginLayoutParams 的功能

## 5、LayoutInflater 生成的 View 是否有 LayoutParams

以下代码是我们惯用的一种将布局文件转换为 View 对象的方法，那么这种方法生成的 View 是否具有 LayoutParams 呢？

```java
View view = LayoutInflater.from(context).inflate(layoutId, root, attachToRoot)
```

是否具备主要要看 root 对象是否为 null：

- 如果 root 为 null，由于无法构造 LayoutParams，View 自然不包含 LayoutParams
- 如果 root 不为 null，那么就会通过 root 为 view 生成 LayoutParams。之后，如果 attachToRoot 为 false，那么就将 LayoutParams 直接赋予 view。如果 attachToRoot  为 true，那么就通过 `root.addView` 的方式将 view 添加到 root 中，root 内部一样会将 LayoutParams 赋予 view

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
            ···
             // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }

                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }

                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);

                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }

                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }
        ···
        return result;
    }
}
```

## 6、向 ViewGroup 添加的 View 是否有 LayoutParams

ViewGroup 包含以下三个没有传入 LayoutParams 的 `addView` 方法，可以看出来如果 child 内部不包含 LayoutParams 的话也会通过 `generateDefaultLayoutParams()` 方法来生成一个默认值，如果该方法返回了 null 的话将抛出异常，默认情况下 DefaultLayoutParams 的宽高值都是 `WRAP_CONTENT`

```java
public void addView(View child) {
    addView(child, -1);
}

public void addView(View child, int index) {
    if (child == null) {
        throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
    }
    LayoutParams params = child.getLayoutParams();
    if (params == null) {
        params = generateDefaultLayoutParams();
        if (params == null) {
            throw new IllegalArgumentException(
                    "generateDefaultLayoutParams() cannot return null");
        }
    }
    addView(child, index, params);
}

public void addView(View child, int width, int height) {
    final LayoutParams params = generateDefaultLayoutParams();
    params.width = width;
    params.height = height;
    addView(child, -1, params);
}

protected LayoutParams generateDefaultLayoutParams() {
    return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
}
```

## 7、requestLayout 方法调用后的流程

当我们动态改变了一个 View 的位置或者宽高大小的时候，就可以通过调用`requestLayout()`方法来重新触发 View 的绘制流程，使得我们的修改可以生效。该方法会触发其自身与父容器回调 `onMeasure` 和 `onLayout` 两个方法，但不会回调 `onDraw` 方法。这里来看下该方法的主干流程

我们知道，除了视图树的根布局 DecorView 外，每个 View 都是承载于其父容器 ViewGroup 中的，那么当 View 需要进行重新布局时，其父容器也肯定是需要进行重新布局的，所以 `requestLayout()`方法会先为本 View 设置 `PFLAG_FORCE_LAYOUT` 标记位，然后调用 `mParent` 的相同方法

```java
public void requestLayout() {
    ···
    //设置强制布局的标记位
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;
    if (mParent != null && !mParent.isLayoutRequested()) {
        //触发父容器进行重新布局
        mParent.requestLayout();
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}
```

通过层层向上传递，最终 DecorView 会收到 requestLayout 的请求，而 DecorView 的 `mParent`指向的是 ViewRootImpl，ViewRootImpl 最终又会调用 `scheduleTraversals()` 方法去重新触发视图树的绘制流程

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

因为本次发生变化的可能只是一小部分 View 而已，如果强制让整个视图树都进行刷新操作那肯定是比较浪费资源的，所以实际上系统有针对 `requestLayout()` 方法进行了一些优化措施，View 会通过 `PFLAG_FORCE_LAYOUT` 标记位来参与判断当前是否需要执行绘制流程

View 的 `measure` 方法调用 `onMeasure` 方法的前置条件之一就是当前有设置 `PFLAG_FORCE_LAYOUT` 标记位，如果 View 的视图没有发生变化又没有设置该标记位的话就不会参与此次绘制流程

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ···
    final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
    ···
    if (forceLayout || needsLayout) {
        ···
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        ···
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }
    ···
}
```

当调用了 `onMeasure` 方法后又会设置 `PFLAG_LAYOUT_REQUIRED` 标记位，该标记位又会参与判断是否需要调用 `onLayout` 方法

```java
public void layout(int l, int t, int r, int b) {
    ···
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);

        ···
    }
    ···
}
```

## 8、属性动画会触发绘制流程吗

会，但可能只会执行一部分流程

例如，如果我们动态改变的是 View 的 MinimumWidth 大小，`setMinimumWidth` 方法只会回调 `onMeasure` 和 `onLayout` 两个方法，但不会回调 `onDraw` 方法

```java
public void setMinimumWidth(int minWidth) {
    mMinWidth = minWidth;
    requestLayout();
}
```

如果我们动态改变的是 View 的 BackgroundColor，则会根据当前 View 的宽高大小或者位置是否发生了变化来决定是否调用 `requestLayout()` 方法，但最终一定会调用 `invalidate(true)` 方法来回调 `onDraw` 方法

```java
public void setBackgroundDrawable(Drawable background) {
    ···
    boolean requestLayout = false;
    ···
    if (requestLayout) {
        requestLayout();
    }
    mBackgroundSizeChanged = true;
    invalidate(true);
    invalidateOutline();
}
```

## 9、parent 是 wrap_content，child 是 match_parent，会怎么显示

假设 Activity 的布局如下所示，FrameLayout 的宽高都是 wrap_content，View 的宽高都是 match_parent。试验下就可以知道 View 视图将占满整个屏幕

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:background="#FF5722"
    tools:context=".MainActivity">

    <View
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#FFC107" />

</FrameLayout>
```

从 ViewGroup 的 `getChildMeasureSpec` 方法也可以得出该结果，该方法用于根据 ViewGroup 的 MeasureSpec 和 childView 的 LayoutParams 来一起生成 childView 的 MeasureSpec

FrameLayout 的 specMode 是 AT_MOST，能占据的最大空间 specSize 即整个屏幕大小。childDimension 等于 MATCH_PARENT，所以 childView 最终对应的 specSize 就是屏幕大小，specMode 就是 AT_MOST

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    ···

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            //直接使用 ViewGroup 能够占据的最大尺寸值
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    ···
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

而 View 对于 AT_MOST 就是默认使用其 specSize，从而使得 View 占据整个屏幕空间

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

# 五、自定义 View Demo

我在蛮久前曾写过几个自定义 View 来练手，这次就趁着写本篇文章的机会用 Kotlin 重写了一遍，需要的同学可以参照下

点击这里获取代码：[AndroidOpenSourceDemo](https://github.com/leavesczy/AndroidOpenSourceDemo)


![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b406f719153140a8aaff808730665813~tplv-k3u1fbpfcp-watermark.image?)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef4ec5a3a18f46f1997c7115c91c5967~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7c7d3cd985a4e4ebababc0b7e1c1a0c~tplv-k3u1fbpfcp-zoom-1.image)