View 的事件分发机制一直是 Android 开发中比较难啃的一块知识点，想要理顺 MotionEvent 在 ViewGroup 和 View 这两者之间流转的规则十分不容易，整个过程涉及分发、拦截、消费三个过程，每个过程根据返回值的不同在流程就会有很大差别，且 Activity 也会参与进这个过程，不参照源码进行分析的话就很难明白触摸事件的分发规则。在很久前我就想过要来动笔写这一块知识点，熬夜肝了一篇，希望对你有所帮助

# 一、坐标系

Android 中的坐标系可以分为两种：**屏幕坐标系** 和 **View 坐标系**

## 屏幕坐标系

屏幕坐标系以屏幕左上角作为坐标原点，水平向右方向为 X 轴正轴方向，竖直向下方向为 Y 轴正轴方向

## View 坐标系

View 坐标系以 View 所在的 ViewGroup 的左上角作为坐标原点，水平向右方向为 X 轴正轴方向，竖直向下方向为 Y 轴正轴方向。View 类包含了以下几个方法用于获取其相对父容器 ViewGroup 的距离：

- getLeft()。View 左侧到 ViewGroup 左侧之间的距离
- getTop()。View 顶部到 ViewGroup 顶部之间的距离
- getRight()。View 右侧到 ViewGroup 左侧之间的距离
- getBottom()。View 底部到 ViewGroup 顶部之间的距离

View 就依赖于这四个距离值来计算宽高大小

```java
public final int getWidth() {
    return mRight - mLeft;
}

public final int getHeight() {
    return mBottom - mTop;
}
```

# 二、MotionEvent

触摸事件最常见的有以下三种类型：

- ACTION_DOWN：用户手指的按下操作，是用户每次触摸屏幕时触发的第一个事件
- ACTION_MOVE：用户手指按压屏幕后，在松开手指之前如果滑动屏幕超出一定的阈值，则发生了 ACTION_MOVE 事件
- ACTION_UP：用户手指离开屏幕时触发的操作，是当次触摸操作的最后一个事件

一次完整的事件序列包含**用户从按下屏幕到离开屏幕之间触发的所有事件**，在这个序列当中，**ACTION_DOWN 和 ACTION_UP 是一定会有的，有且只有一个，ACTION_MOVE 则视情况而定，数量大于等于零**（这里不考虑多点触控的情况）

每个事件都会被包装为 MotionEvent 类

```kotlin
fun dispatchTouchEvent(event: MotionEvent) {
    when (event.action) {
        MotionEvent.ACTION_DOWN -> TODO()
        MotionEvent.ACTION_MOVE -> TODO()
        MotionEvent.ACTION_UP -> TODO()
    }
}
```

MotionEvent 包含了该次触摸事件发生的坐标点，分为两组不同的方法

```kotlin
//基于 View 左上角获取到的距离
motionEvent.getX();
motionEvent.getY();

//基于屏幕左上角获取到的距离
motionEvent.getRawX();
motionEvent.getRawY();
```

此外，系统内置了一个最小滑动距离值，只有先后两个坐标点之间的距离超出该值，才会认为属于滑动事件

```kotlin
ViewConfiguration.get(Context).getScaledTouchSlop() 
```

# 三、事件分发的三个阶段

在整个事件分发过程中，我们主要接触的是 ViewGroup 和 View 这两种视图类型。一次完整的事件分发过程会包括三个阶段，即事件的**发布、拦截和消费**，这三个过程分别对应声明在 View 和 ViewGroup 中的三个方法

## 发布

事件的发布对应着如下方法

```java
public boolean dispatchTouchEvent(MotionEvent ev)
```

Android 中的视图（View、ViewGroup、Activity 等）接收到的触摸事件都是通过这个方法来进行分发的，如果事件能够传递给当前视图，则此方法一定会被调用，即视图接收到的触摸事件都需要通过该方法来进行分发。该方法的返回值用于表明该视图或者内嵌视图是否消费了该事件。如果当前视图类型是 ViewGroup，该方法内部会调用 `onInterceptTouchEvent(MotionEvent)` 方法来判断是否拦截该事件

## 拦截

事件的拦截对应着如下方法

```java
public boolean onInterceptTouchEvent(MotionEvent ev)
```

ViewGroup 包含该方法，View 中不存在。该方法通过返回值来标明是否需要拦截对应的事件。返回 true 则表示拦截这个事件，不继续发布给子视图，并将事件交由自身的 `onTouchEvent(MotionEvent event)` 方法来进行处理；返回 false 则表示不拦截事件，继续传递给子视图。如果 ViewGroup 拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用

## 消费

事件的消费对应着如下方法

```java
public boolean onTouchEvent(MotionEvent event)
```

该方法返回 true 表示当前视图已经处理了对应的事件，事件将在这里完成消费，终止传递；返回 false 表示当前视图不处理这个事件，事件会被传递给其它视图

## 三者的联系

ViewGroup 完整包含以上三个过程，而 View 只包含**分发和消费**两个，既 View 类不包含 `onInterceptTouchEvent(MotionEvent)` 方法。三个方法之间的联系可以用如下伪代码来表示：

```kotlin
fun dispatchTouchEvent(event: MotionEvent): Boolean {
    var consume = false
    consume = if (onInterceptTouchEvent(event)) {
        onTouchEvent(event)
    } else {
        child.dispatchTouchEvent(event)
    }
    return consume
}
```

当触摸事件发生时，事件分发流程会按照如下执行：

- 根 ViewGroup 最先接收到 MotionEvent，其 dispatchTouchEvent 方法会被调用到，该方法内部会调用 onInterceptTouchEvent 方法来判断是否要拦截事件
- ViewGroup 的 onInterceptTouchEvent 方法如果返回 true，则表示当前 ViewGroup 要拦截事件，否则就会去调用 child（内嵌的 ViewGroup 或者是 View）重复分发过程
- View 和 ViewGroup 的 onTouchEvent 方法用来判断是否要消费该事件，如果返回了 true 则表示事件已被消费，终止传递

当然，View 的事件分发过程不是上述介绍的那么简单，实际上事件的流转过程很复杂，根据每个方法返回值的不同，事件序列的流转方向会有很大差异。直接看以下的例子才比较容易理解

# 四、举个例子

## 打印日志

这里分别继承于 RelativeLayout、LinearLayout 和 TextView，重写以上三个方法，打印各个方法的返回值，观察其调用时机

类似于这样：

```kotlin
class MyRelativeLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : RelativeLayout(context, attrs, defStyleAttr) {

    protected fun log(any: Any?) {
        Log.e("MyRelativeLayout", any?.toString() ?: "null")
    }

    override fun dispatchTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> log("dispatchTouchEvent ACTION_DOWN")
            MotionEvent.ACTION_MOVE -> log("dispatchTouchEvent ACTION_MOVE")
            MotionEvent.ACTION_UP -> log("dispatchTouchEvent ACTION_UP")
        }
        val flag = super.dispatchTouchEvent(event)
        log("dispatchTouchEvent return: $flag")
        return flag
    }

    override fun onInterceptTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> log("onInterceptTouchEvent ACTION_DOWN")
            MotionEvent.ACTION_MOVE -> log("onInterceptTouchEvent ACTION_MOVE")
            MotionEvent.ACTION_UP -> log("onInterceptTouchEvent ACTION_UP")
        }
        val flag = super.onInterceptTouchEvent(event)
        log("onInterceptTouchEvent return: $flag")
        return flag
    }

    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> log("onTouchEvent ACTION_DOWN")
            MotionEvent.ACTION_MOVE -> log("onTouchEvent ACTION_MOVE")
            MotionEvent.ACTION_UP -> log("onTouchEvent ACTION_UP")
        }
        val flag = super.onTouchEvent(event)
        log("onTouchEvent return: $flag")
        return flag
    }

}
```

布局的嵌套层次：

```xml
<github.leavesc.motion_event.MyRelativeLayout>

    <github.leavesc.motion_event.MyLinearLayout>

        <github.leavesc.motion_event.MyTextView/>

    </github.leavesc.motion_event.MyLinearLayout>

</github.leavesc.motion_event.MyRelativeLayout>
```

点击 TextView 内容区域，打印出来的日志信息：

```kotlin
MyRelativeLayout: dispatchTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent return: false

MyLinearLayout: dispatchTouchEvent ACTION_DOWN
MyLinearLayout: onInterceptTouchEvent ACTION_DOWN
MyLinearLayout: onInterceptTouchEvent return: false

MyTextView: dispatchTouchEvent ACTION_DOWN
MyTextView: onTouchEvent ACTION_DOWN
MyTextView: onTouchEvent return: false
MyTextView: dispatchTouchEvent return: false

MyLinearLayout: onTouchEvent ACTION_DOWN
MyLinearLayout: onTouchEvent return: false
MyLinearLayout: dispatchTouchEvent return: false

MyRelativeLayout: onTouchEvent ACTION_DOWN
MyRelativeLayout: onTouchEvent return: false
MyRelativeLayout: dispatchTouchEvent return: false
```

从以上日志信息可以总结出：

1. 当点击屏幕时，即使当前点击的区域处于 TextView 内，事件分发流程也是从 MyRelativeLayout 这个根 ViewGroup 开始的。系统会根据触摸点来判断手指是落在哪个 ViewGroup 内，然后通过遍历的方式来找到坐标点最终是落在哪个底层 View 内部，然后在 ViewGroup 和 View 之间来流转整个事件序列
2. 事件分发流程先从根 ViewGroup 从上往下（从外向内）向内嵌的底层 View 传递，即从 MyRelativeLayout 到 MyLinearLayout，再到 MyTextView，最终又反向传递从下往上（从内向外）进行传递。这里说的底层 View 既指 View 类的各种子类，也可以指不包含 View 的 ViewGroup，**本文特定指各种非 ViewGroup 类型的 View 子类**
3. 对于 ViewGroup 来说，其 dispatchTouchEvent 方法内部会先调用其 onInterceptTouchEvent 判断是否需要进行拦截，如果 onInterceptTouchEvent 方法返回了 false，则意味着其不打算拦截该事件，那么就会继续调用 child 的 dispatchTouchEvent 方法，继续重复以上步骤
4. 如果根 ViewGroup 内嵌的所有 ViewGroup 均不拦截触摸事件的话，那么事件通过循环传递就会分发给最底层的 View
5. 对于 View 来说，其不包含 onInterceptTouchEvent 方法，dispatchTouchEvent 方法会直接调用其 onTouchEvent 方法来决定是否消费该触摸事件。如果返回 false，则意味着其不打算消费该事件，返回 true 的话则意味着事件被其消费了，终止传递。此时触摸事件已经到了最底层，由于 TextView 默认就是不可点击的，在默认状态下不会消费任何触摸事件，由于找不到消费者，所以接着就会将事件依次返还给父容器
6. MyTextView 不打算消费该触摸事件后，MyLinearLayout 的 onTouchEvent 方法就会接着被调用，之后 MyLinearLayout 的 dispatchTouchEvent 才最终得到确定的返回值 false。这说明内部 View 的回调事件是由其父容器 ViewGroup 来负责调用的，通过递归调用的方式来完成整个事件的分发，从 MyRelativeLayout 的 dispatchTouchEvent 方法的返回值是最后才打印也可以看出来

## Activity 参与事件分发

前文有讲到，每次的触摸事件都是从 ACTION_DOWN 开始，以 ACTION_UP 作为结尾的，可是上面的日志信息却只看到了 ACTION_DOWN，ACTION_UP 去哪了呢？

其实，触摸事件的起始分发点应该从 Activity 看起才对，Activity 会早于 ViewGroup 收到触摸事件，重写 Activity 的 dispatchTouchEvent 和 onTouchEvent 两个方法，可以得到如下日志：

```kotlin
MotionMainActivity: dispatchTouchEvent ACTION_DOWN

MyRelativeLayout: dispatchTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent return: false

MyLinearLayout: dispatchTouchEvent ACTION_DOWN
MyLinearLayout: onInterceptTouchEvent ACTION_DOWN
MyLinearLayout: onInterceptTouchEvent return: false

MyTextView: dispatchTouchEvent ACTION_DOWN
MyTextView: onTouchEvent ACTION_DOWN
MyTextView: onTouchEvent return: false
MyTextView: dispatchTouchEvent return: false

MyLinearLayout: onTouchEvent ACTION_DOWN
MyLinearLayout: onTouchEvent return: false
MyLinearLayout: dispatchTouchEvent return: false

MyRelativeLayout: onTouchEvent ACTION_DOWN
MyRelativeLayout: onTouchEvent return: false
MyRelativeLayout: dispatchTouchEvent return: false

MotionMainActivity: onTouchEvent ACTION_DOWN
MotionMainActivity: onTouchEvent return: false
MotionMainActivity: dispatchTouchEvent return: false
MotionMainActivity: dispatchTouchEvent ACTION_UP
MotionMainActivity: onTouchEvent ACTION_UP
MotionMainActivity: onTouchEvent return: false
MotionMainActivity: dispatchTouchEvent return: false
```

从以上日志信息可以总结出：

1. Activity 会早于各个 ViewGroup 和 View 接收到触摸事件，ViewGroup 和 View 没有消费掉的 ACTION_DOWN 事件最终还是会交由 Activity 来消化掉
2. 由于 ViewGroup 和 View 均没有消费掉 ACTION_DOWN 事件，所以后续的 ACTION_UP 事件不会再继续向它们下发，而是会直接调用 Activity 的 onTouchEvent 方法，由 Activity 来消化掉

## ViewGroup 拦截事件

如果 ViewGroup 自身拦截且消费了 ACTION_DOWN 事件，即 onInterceptTouchEvent 和 onTouchEvent 两个方法均返回了 true，那么本次事件序列的后续事件就都会交由其进行处理（如果能接收得到的话），不会再调用其 onInterceptTouchEvent 方法来判断是否进行拦截，dispatchTouchEvent 方法会直接调用 onTouchEvent 方法

而如果 ViewGroup 拦截了 ACTION_DOWN 事件，但是 onTouchEvent 方法中又没有消费掉该事件的话，那么本次事件序列的后续事件都不会再被其接收到，而是直接交由父视图进行处理。View 对 ACTION_DOWN 事件的处理逻辑也是如此

如果所有的 ViewGroup 和 View 都没有消耗 ACTION_DOWN 事件的话，则后续事件（ACTION_MOVE 和 ACTION_UP 等）都会直接交由 Activity 进行处理， ViewGroup 和 View 没有机会再接触到后续事件

可以改下处于中间层次的 MyLinearLayout 来验证下，onInterceptTouchEvent 方法接收到 ACTION_DOWN 事件时直接返回 true

```kotlin
class MyLinearLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : LinearLayout(context, attrs, defStyleAttr) {

    override fun onInterceptTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                log("onInterceptTouchEvent ACTION_DOWN")
                return true
            }
            MotionEvent.ACTION_MOVE -> log("onInterceptTouchEvent ACTION_MOVE")
            MotionEvent.ACTION_UP -> log("onInterceptTouchEvent ACTION_UP")
        }
        val flag = super.onInterceptTouchEvent(event)
        log("onInterceptTouchEvent return: $flag")
        return flag
    }

}
```

此时 MyLinearLayout 拦截了 ACTION_DOWN 事件，所以 MyTextView 不会接收到该事件。但由于 MyLinearLayout 并没有消费掉该事件，所以 ACTION_DOWN 事件还是会传回给父容器 MyRelativeLayout，而 MyRelativeLayout 默认也是不会消费该事件，所以后续的 ACTION_UP 也只会交由 Activity 进行处理

```kotlin
MotionMainActivity: dispatchTouchEvent ACTION_DOWN

MyRelativeLayout: dispatchTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent return: false

MyLinearLayout: dispatchTouchEvent ACTION_DOWN
MyLinearLayout: onInterceptTouchEvent ACTION_DOWN
MyLinearLayout: onTouchEvent ACTION_DOWN
MyLinearLayout: onTouchEvent return: false
MyLinearLayout: dispatchTouchEvent return: false

MyRelativeLayout: onTouchEvent ACTION_DOWN
MyRelativeLayout: onTouchEvent return: false
MyRelativeLayout: dispatchTouchEvent return: false

MotionMainActivity: onTouchEvent ACTION_DOWN
MotionMainActivity: onTouchEvent return: false
MotionMainActivity: dispatchTouchEvent return: false
MotionMainActivity: dispatchTouchEvent ACTION_UP
MotionMainActivity: onTouchEvent ACTION_UP
MotionMainActivity: onTouchEvent return: false
MotionMainActivity: dispatchTouchEvent return: false
```

而如果 MyLinearLayout 既拦截也消费了 ACTION_DOWN 事件，那么还是可以接收到后续事件

```kotlin
class MyLinearLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : LinearLayout(context, attrs, defStyleAttr) {

    override fun onInterceptTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                log("onInterceptTouchEvent ACTION_DOWN")
                return true
            }
            MotionEvent.ACTION_MOVE -> log("onInterceptTouchEvent ACTION_MOVE")
            MotionEvent.ACTION_UP -> log("onInterceptTouchEvent ACTION_UP")
        }
        val flag = super.onInterceptTouchEvent(event)
        log("onInterceptTouchEvent return: $flag")
        return flag
    }

    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                log("onTouchEvent ACTION_DOWN")
                return true
            }
            MotionEvent.ACTION_MOVE -> log("onTouchEvent ACTION_MOVE")
            MotionEvent.ACTION_UP -> log("onTouchEvent ACTION_UP")
        }
        val flag = super.onTouchEvent(event)
        log("onTouchEvent return: $flag")
        return flag
    }

}
```

从日志可以看到 MyLinearLayout 接收到了后续的 ACTION_MOVE 和 ACTION_UP 事件，且此时并没有再次调用 onInterceptTouchEvent 方法，而是直接调用了 onTouchEvent 方法

```kotlin
MyRelativeLayout: dispatchTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent return: false

MyLinearLayout: dispatchTouchEvent ACTION_DOWN
MyLinearLayout: onInterceptTouchEvent ACTION_DOWN
MyLinearLayout: onTouchEvent ACTION_DOWN
MyLinearLayout: dispatchTouchEvent return: true

MyRelativeLayout: dispatchTouchEvent return: true
MyRelativeLayout: dispatchTouchEvent ACTION_MOVE
MyRelativeLayout: onInterceptTouchEvent ACTION_MOVE
MyRelativeLayout: onInterceptTouchEvent return: false

MyLinearLayout: dispatchTouchEvent ACTION_MOVE
MyLinearLayout: onTouchEvent ACTION_MOVE
MyLinearLayout: onTouchEvent return: false
MyLinearLayout: dispatchTouchEvent return: false

MyRelativeLayout: dispatchTouchEvent return: false
MyRelativeLayout: dispatchTouchEvent ACTION_UP
MyRelativeLayout: onInterceptTouchEvent ACTION_UP
MyRelativeLayout: onInterceptTouchEvent return: false

MyLinearLayout: dispatchTouchEvent ACTION_UP
MyLinearLayout: onTouchEvent ACTION_UP
MyLinearLayout: onTouchEvent return: false
MyLinearLayout: dispatchTouchEvent return: false

MyRelativeLayout: dispatchTouchEvent return: false
```

**此外，有一个需要注意的点是，即使每个 ACTION_MOVE 事件 MyLinearLayout 均没有消费掉，MyLinearLayout 一样可以完整接收到整个事件序列的消息，且此时父容器的 onTouchEvent 方法也不会被回调。因为在正常情况下，一个事件序列只应该由单独一个 View 或者 ViewGroup 进行处理，既然 MyLinearLayout 已经消费了 ACTION_DOWN 事件，那么后续的事件应该也都交由其进行处理**

## View 消费事件

View 没有拦截事件这个过程，但如果有消费掉 ACTION_DOWN 事件的话，后续事件就都可以接收到

```kotlin
class MyTextView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : AppCompatTextView(context, attrs, defStyleAttr) {

    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                log("onTouchEvent ACTION_DOWN")
                return true
            }
            MotionEvent.ACTION_MOVE -> log("onTouchEvent ACTION_MOVE")
            MotionEvent.ACTION_UP -> log("onTouchEvent ACTION_UP")
        }
        val flag = super.onTouchEvent(event)
        log("onTouchEvent return: $flag")
        return flag
    }

}
```

可以看到，MyTextView 接收到了后续的 ACTION_MOVE 和 ACTION_UP 事件

```kotlin
MyRelativeLayout: dispatchTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent ACTION_DOWN
MyRelativeLayout: onInterceptTouchEvent return: false

MyLinearLayout: dispatchTouchEvent ACTION_DOWN
MyLinearLayout: onInterceptTouchEvent ACTION_DOWN
MyLinearLayout: onInterceptTouchEvent return: false

MyTextView: dispatchTouchEvent ACTION_DOWN
MyTextView: onTouchEvent ACTION_DOWN
MyTextView: dispatchTouchEvent return: true

MyLinearLayout: dispatchTouchEvent return: true

MyRelativeLayout: dispatchTouchEvent return: true
MyRelativeLayout: dispatchTouchEvent ACTION_MOVE
MyRelativeLayout: onInterceptTouchEvent ACTION_MOVE
MyRelativeLayout: onInterceptTouchEvent return: false

MyLinearLayout: dispatchTouchEvent ACTION_MOVE
MyLinearLayout: onInterceptTouchEvent ACTION_MOVE
MyLinearLayout: onInterceptTouchEvent return: false

MyTextView: dispatchTouchEvent ACTION_MOVE
MyTextView: onTouchEvent ACTION_MOVE
MyTextView: onTouchEvent return: false
MyTextView: dispatchTouchEvent return: false

MyLinearLayout: dispatchTouchEvent return: false

MyRelativeLayout: dispatchTouchEvent return: false
MyRelativeLayout: dispatchTouchEvent ACTION_UP
MyRelativeLayout: onInterceptTouchEvent ACTION_UP
MyRelativeLayout: onInterceptTouchEvent return: false

MyLinearLayout: dispatchTouchEvent ACTION_UP
MyLinearLayout: onInterceptTouchEvent ACTION_UP
MyLinearLayout: onInterceptTouchEvent return: false

MyTextView: dispatchTouchEvent ACTION_UP
MyTextView: onTouchEvent ACTION_UP
MyTextView: onTouchEvent return: false
MyTextView: dispatchTouchEvent return: false

MyLinearLayout: dispatchTouchEvent return: false

MyRelativeLayout: dispatchTouchEvent return: false
```

和上面一个例子一样。即使每个 ACTION_MOVE 事件 MyTextView 均没有消费掉，MyTextView 一样可以完整接收到整个事件序列的消息，且此时父容器的 onTouchEvent 方法也不会被回调，整个事件序列都只交由 MyTextView 来处理了

我们也可以通过修改代码来使得上层 ViewGroup 主动拦截后续事件，但这也会导致一些问题，因为如果 MyTextView 没有接收到 ACTION_UP 事件的话，会导致其 OnClickListener 无法被回调

**总的来说，View 是否能接收到整个事件序列的消息主要就取决于其是否消费了 ACTION_DOWN 事件，ACTION_DOWN 事件是整个事件序列的起始点，View 必须消耗了起始事件才有机会完整处理整个事件序列**

# 五、总结

1. Activity 会早于各个 ViewGroup 和 View 接收到触摸事件，Activity 可以通过主动拦截掉各个事件的下发使得 ViewGroup 和 View 接收不到任何事件。而如果 ViewGroup 和 View 接收到了 ACTION_DOWN 事件但没有消费掉，那么事件最终还是会交由 Activity 来消费
2. 当触摸事件被触发时，系统会根据触摸点的坐标系找到根 ViewGroup，然后向底层 View 下发事件，即事件分发流程先是从根 ViewGroup 从上往下（从外向内）向内嵌的底层 View 传递的，如果在这个过程中事件没有被消费的话，最终又会反向传递从下往上（从内向外）进行传递
3. ViewGroup 在接收到 ACTION_DOWN 事件时，其 dispatchTouchEvent 方法内部会先调用 onInterceptTouchEvent 判断是否要进行拦截，如果 onInterceptTouchEvent 方法返回了 false，则意味着其不打算拦截该事件，那么就会继续调用 child 的 dispatchTouchEvent 方法，继续重复以上步骤。如果拦截了，那么就会调用 onTouchEvent 进行消费
4. 如果 ViewGroup 自身拦截且消费了 ACTION_DOWN 事件，那么本次事件序列的后续事件就会都交由其进行处理（如果能接收得到的话），不会再调用其 onInterceptTouchEvent 方法来判断是否进行拦截，也不会再次遍历 child，dispatchTouchEvent 方法会直接调用 onTouchEvent 方法。这是为了尽量避免无效操作，提高系统的绘制效率
5. 如果根 ViewGroup 和内嵌的所有 ViewGroup 均没有拦截 ACTION_DOWN 事件的话，那么事件通过循环传递就会分发给最底层的 View。对于 View 来说，其不包含 onInterceptTouchEvent 方法，dispatchTouchEvent 方法会调用其 onTouchEvent 方法来决定是否消费该事件。如果返回 false，则意味着其不打算消费该事件，事件将依次调用父容器的 onTouchEvent 方法；返回 true 的话则意味着事件被其消费了，事件终止传递
6. 而不管 ViewGroup 有没有拦截 ACTION_DOWN 事件，只要其本身和所有 child 均没有消费掉 ACTION_DOWN 事件，即 dispatchTouchEvent 方法返回了 false，那么此 ViewGroup 就不会再接收到后续事件，后续事件会被 Activity 直接消化掉
7. 而不管是 ViewGroup 还是 View，只要其消费了 ACTION_DOWN 事件，即使 onTouchEvent 方法在处理每个后续事件时均返回了 false，都还是可以完整接收到整个事件序列的消息。后续事件会根据在在处理 ACTION_DOWN 事件保留的引用链，从上往下依次下发
8. View 是否能接收到整个事件序列的消息主要就取决于其是否消费了 ACTION_DOWN 事件，ACTION_DOWN 事件是整个事件序列的起始点，View 必须消耗了起始事件才有机会完整处理整个事件序列
9. 处于上游的 ViewGroup 不关心到底是下游的哪个 ViewGroup 或者 View 消费了触摸事件，只要下游的 dispatchTouchEvent 方法返回了 true，上游就会继续向下游下发后续事件
10. ViewGroup 和 View 对于每次事件序列的消费过程是独立的，即上一次事件序列的消费结果不影响新一次的事件序列

# 六、View

View 是 Android 整个体系最基础的基类之一，这里来对 View 的事件分发源码做下分析，以此来验证上边我给出的结论，基于 SDK 30 版本进行分析

## dispatchTouchEvent

View 的 dispatchTouchEvent 方法逻辑上还比较简单，可以总结为：

1. 对应第一步。如果 View 是 ENABLED 状态，既处于可用状态，且当前是通过鼠标设备输出的 ScrollBarDragging 事件并被处理了，那么就说明当前 View 消耗了本次触摸事件
2. 对应第二步。如果 View 是 ENABLED 状态，且此时外部设置的 OnTouchListener 返回了 true，那么就说明当前 View 交由外部消耗了本次触摸事件
3. 对应第三步。如果以上几步均不成立，那么就会再调用 onTouchEvent 方法，如果该方法返回了 true，那么也说明当前 View 消耗了本次触摸事件
4. 所以说，外部设置的 OnTouchListener 的优先级会高于自身的 onTouchEvent 方法，OnTouchListener 可以通过返回 true 使得 onTouchEvent 方法不被调用

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    ···
    // 用于表示当前 View 是否消费了该事件
    boolean result = false;
    final int actionMasked = event.getActionMasked();
    ···
    if (onFilterTouchEventForSecurity(event)) {
        // 第一步
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        // 第二步
        ListenerInfo li = mListenerInfo;
        if (li != null
                && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        // 第三步
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    ···
    return result;
}
```

## onTouchEvent

onTouchEvent 方法就比较复杂了，我们只看其主干思路即可，可以总结为：

1. 对应第一步。如果当前 View 处于禁用状态 DISABLED 的话，当前 View 是否会消耗触摸事件都由 clickable 来决定，即 CLICKABLE、LONG_CLICKABLE、CONTEXT_CLICKABLE 这三个条件至少有一个满足的话，那么就返回 true。这三个条件分别对应着：可点击、可长按点击、可上下文点击
2. 对应第二步。如果 TouchDelegate 存在且消耗了触摸事件，那么就返回 true
3. 对应第三步。如果当前 View 处于 clickable 状态或者 `(viewFlags & TOOLTIP) == TOOLTIP` 成立的话，那么也会消耗当前事件。TOOLTIP 可以通过添加 `android:tooltipText="tips"` 来开启，开启后长按 TextView 会显示一个悬浮窗形式的提示文本
4. 对应第四步。在接收到 ACTION_UP 事件的时候，判断是否回调外部设置的 OnClickListener。因此如果外部设置的 OnTouchListener 返回了 true，那么 OnClickListener 就根本没有机会被调用，且如果上层视图消耗了 ACTION_UP 事件或者是当前 View 处于禁用状态 DISABLED 的话，OnClickListener 也不会被调用

```java
public boolean onTouchEvent(MotionEvent event) {
    ···
    final boolean clickable =
            ((viewFlags & CLICKABLE) == CLICKABLE
                            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        ···
        // 第一步
        return clickable;
    }
    // 第二步
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
    // 第三步
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        ···
        if (!focusTaken) {
            if (mPerformClick == null) {
                mPerformClick = new PerformClick();
            }
            if (!post(mPerformClick)) {
                // 第四步
                performClickInternal();
            }
        }
        ···
        return true;
    }
    return false;
}
```

所以说，dispatchTouchEvent 内部的确是会调用 onTouchEvent 方法，且如果 View 处于可点击状态的话，那么就会消耗该触摸事件，且 OnClickListener 是在 onTouchEvent 方法中被调用的

举个例子。TextView 默认是不可点击状态，而 Button 是直接继承于 TextView 的，因此 Button 默认状态也是不可点击且不会消耗任何触摸事件的，而 Button 之所以在我们日常使用过程中会消耗掉触摸事件，是因为往往我们都会为其设置 OnClickListener，此时就会将 Button 的 Clickable 置为 true

```java
public void setOnClickListener(@Nullable OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}
```

# 七、ViewGroup

ViewGroup 直接继承于 View，其逻辑是在 View 的基础上来做扩展的，这里就直接看 ViewGroup 类是如何来实现上述介绍的三个方法的

## dispatchTouchEvent

ViewGroup 的 dispatchTouchEvent 方法相对 View 就要复杂很多了，因为 View 在整个视图体系中处于最基础的底层，只需要管理好自己就可以，而 ViewGroup 还需要管理其内嵌的布局，可能会包含多个子 ViewGroup 和子 View

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    ···
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        //第一步
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        //第二步
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action);
            } else {
                intercepted = false;
            }
        } else {
            intercepted = true;
        }

        ···

        if (!canceled && !intercepted) {
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;

            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                //第三步
                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    ···
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        ···
                        }
                    }
                    ···
                }
               ···
            }
        }

        if (mFirstTouchTarget == null) {
            //第四步
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            //第五步
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }
        ···
    }

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```

该方法的主要流程可以总结为：

1. 对应第一步。如果当前接收到的是 ACTION_DOWN 事件，说明是一次新的事件序列，则清除掉 mFirstTouchTarget 的引用。在每次事件序列中，如果 child 消费了 ACTION_DOWN 事件，那么 ViewGroup 就会通过 mFirstTouchTarget 来指向 child，后续事件就可以通过该引用来直接传递而不需再次进行遍历
2. 对应第二步。此步骤用来判断是否要拦截事件。如果 if 条件成立，说明当前处理的是**新的一次事件序列**或者是 **ACTION_DOWN 之后的事件且之前的 ACTION_DOWN 已经被 child 消费了**，那么就通过调用 onInterceptTouchEvent 方法来决定是否拦截。而如果 child 主动通过调用 `mParent.requestDisallowInterceptTouchEvent` 请求当前 ViewGroup 不进行拦截的话（既 disallowIntercept 为 true），那么就直接将 intercepted 置为 false，不进行拦截。这就说明了，除非 child 主动要求 ViewGroup 不拦截，否则属于 child 的事件序列父布局都还是有机会进行拦截的
3. 在第二步中，需要注意 ACTION_DOWN 事件不受 FLAG_DISALLOW_INTERCEPT 这个标记的控制，即 child 无法主动阻止 ViewGroup 不拦截 ACTION_DOWN 事件，ViewGroup 的 onInterceptTouchEvent 方法依然会被调用
4. 在第二步中，假设 ViewGroup 在接收到 ACTION_DOWN 的时候进行了拦截，那么 mFirstTouchTarget 就不会被赋值，这也导致了在接收后续事件时 if 语句不成立，这样在整个事件序列中 onInterceptTouchEvent 方法只会执行一次，这也是上文给出的总结内容之一
5. 对应第三步。ACTION_DOWN 事件会走到这里，由于当前 intercepted 为 false，即不拦截事件，因此此时就会去遍历 children，判断触摸点坐标系是落在哪个 child 内，找得到的话就用 mFirstTouchTarget 指向该 child
6. 对应第四步。此时 mFirstTouchTarget 为 null，说明 ViewGroup 没有找到下一个可以接收事件的 child，也许是没有 child，也许是 child 均不处理该事件，也可能是 ViewGroup 自己拦截了该事件，那么就将当前 ViewGroup 当做一个普通的 View 子类，通过调用 dispatchTransformedTouchEvent 方法来执行父类 View 的 dispatchTouchEvent 方法，按照原始的 View 分发逻辑进行执行。因此 ViewGroup 在主动拦截事件后就会去调用 onTouchEvent 方法
7. 在第四步中，如果此 ViewGroup 最终消费了该事件，那么在接收到后续事件的时候，此时 mFirstTouchTarget 没有指向 child，还是为 null，所以就会直接走第二步的 else 语句，从而不去遍历 children。这就意味着后续事件既不会回调 onInterceptTouchEvent 方法，也不会去遍历 child，这也是上文给出的总结内容之一
8. 对应第五步。此时 mFirstTouchTarget 不为 null，那么就会去调用 child 的 dispatchTouchEvent 方法，重复以上步骤，从而得知 child 对该事件的处理结果 handled
9. 所以说，ViewGroup 通过这种递归调用，最终就会为上层视图 Activity 返回最终的事件处理结果

## onInterceptTouchEvent

onInterceptTouchEvent 方法只在特定几种情况下才会返回 true，成立条件似乎是当存在外置鼠标设备的时候才有可能成立，读者只需要记住该方法默认返回 false 即可，既默认不进行拦截

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
            && ev.getAction() == MotionEvent.ACTION_DOWN
            && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
            && isOnScrollbarThumb(ev.getX(), ev.getY())) {
        return true;
    }
    return false;
}
```

## onTouchEvent

ViewGroup 没有重写其父类 View 的 onTouchEvent 方法，所以此方法和 View 类的逻辑保持一致

# 八、Activity、PhoneWindow、DecorView

上文在很多地方都讲到了 Activity 会参与 View 的事件分发机制，而实际上除了 Activity 外，这个过程还包含 PhoneWindow 和 DecorView，只是这两个都被包含在 Activity 内部，日常开发中一般都不会接触到这一块。这里就再来介绍下这三者的作用

- 每个 Activity 都对应一个 PhoneWindow，即每个 Activity 实例均包含了一个 PhoneWindow 实例
- 每个 PhoneWindow 都对应一个 DecorView，DecorView 依靠 PhoneWindow 作为构造参数之一来实例化
- DecorView 是 FrameLayout 的子类，是 Activity 视图树的根视图，我们平时调用 setContentView 所添加的 View 就对应 DecorView 的 ContentParent 区域
- 在这三者中 DecorView 会最先接收到触摸事件，DecorView 作为视图树的根视图，就负责向其内部 View 下发触摸事件

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d474d52a06d4cce9116958136dc01f6~tplv-k3u1fbpfcp-zoom-1.image)

DecorView 的 dispatchTouchEvent 方法会拿到 PhoneWindow 内含的 Window.Callback 对象向其转发事件，而这里的 Window.Callback 实际上就对应的是 Activity，Activity 实现了 Window.Callback 接口

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {

    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev)
                : super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        ···
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        return onInterceptTouchEvent(event);
    }
}
```

Activity 的 dispatchTouchEvent 和 onTouchEvent 两个方法逻辑都比较简单，在接收到事件的时候都会判断 PhoneWindow 是否要消费该事件，要的话则直接交由其处理，否则默认不消费任何事件。此外 Activity 也提供了一个空实现的 onUserInteraction 方法，向子类提供了 ACTION_DOWN 事件的触发通知

```java
public class Activity {

    private Window mWindow;

    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }

    public void onUserInteraction() {}

    public boolean onTouchEvent(MotionEvent event) {
        if (mWindow.shouldCloseOnTouch(this, event)) {
            finish();
            return true;
        }

        return false;
    }
}
```

PhoneWindow 是 Window 这个抽象类的唯一实现类，PhoneWindow 又将对应的事件交给了 DecorView

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {

    private DecorView mDecor;

    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
}
```

再回过头看 DecorView。DecorView 的 superDispatchTouchEvent 方法直接就调用了父类的 dispatchTouchEvent 方法

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {

    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev)
                : super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        ···
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        return onInterceptTouchEvent(event);
    }
}
```

这三者之间的联系又是怎样的呢？这样兜兜转转一圈，其实就是 DecorView 先将事件传给了 Activity，Activity 又传给了 PhoneWindow，PhoneWindow 又将事件传给了 DecorView，DecorView 最后又按照 ViewGroup 默认的方式进行事件分发，看起来就是在绕圈，这样设计的意义是什么呢？

其实，DecorView 作为触摸事件的第一个接收者，是触摸事件从系统下发到 Activity 之间的一个沟通桥梁，而开发者可以直接接触并继承的是 Activity。DecorView 需要先将事件转发给最外层的 Activity，使得开发者可以通过重写 dispatchTouchEvent 和 onTouchEvent 方法以达到对当前屏幕触摸事件进行拦截的目的。DecorView 作为 View 树的根节点，从 PhoneWindow 接收到事件后，又负责将将事件事件分发给子 View，从而将整个事件链给串联了起来

因此，这三者之间的事件流转机制，可以说是为了给开发者一个可以进行全局事件拦截的机会

# 九、滑动冲突

如果父容器和子 View 都可以响应滑动事件的话，那么就有可能发生滑动冲突的情况。解决滑动冲突的方法大致上可以分为两种：**外部拦截法** 和 **内部拦截法**

## 外部拦截法

父容器根据实际情况在 onInterceptTouchEvent 方法中对触摸事件进行选择性拦截，如果判断到当前滑动事件自己需要，那么就拦截事件并消费，否则就交由子 View 进行处理。该方式有几个注意点：

- ACTION_DOWN 事件父容器不能进行拦截，否则根据 View 的事件分发机制，后续的 ACTION_MOVE 与 ACTION_UP 事件都将默认交由父容器进行处理
- 根据实际的业务需求，父容器判断是否需要处理 ACTION_MOVE 事件，如果需要处理则进行拦截消费，否则交由子 View 去处理
- 原则上 ACTION_UP 事件父容器不应该进行拦截，否则子 View 的 onClick 事件将无法被触发

伪代码：

```kotlin
override fun onInterceptTouchEvent(event: MotionEvent): Boolean {
    var intercepted = false
    when (event.action) {
        MotionEvent.ACTION_DOWN -> {
            intercepted = false
        }
        MotionEvent.ACTION_MOVE -> {
            intercepted = if (满足拦截要求) {
                true
            } else {
                false
            }
        }
        MotionEvent.ACTION_UP -> {
            intercepted = false
        }
    }
    return intercepted
}
```

## 内部拦截法

内部拦截法则是要求父容器不拦截任何事件，所有事件都传递给子 View，子 View 根据实际情况判断是自己来消费还是传回给父容器进行处理。该方式有几个注意点：

- 父容器不能拦截 ACTION_DOWN 事件，否则后续的触摸事件子 View 都无法接收到
- 滑动事件的舍取逻辑放在子 View 的 `dispatchTouchEvent` 方法中，如果父容器需要处理事件则调用 `parent.requestDisallowInterceptTouchEvent(false)` 方法让父容器去拦截事件

伪代码：

子 View 修改其 dispatchTouchEvent 方法，根据实际需求来控制是否允许父容器拦截事件

```kotlin
override fun dispatchTouchEvent(event: MotionEvent): Boolean {
    when (event.action) {
        MotionEvent.ACTION_DOWN -> {
            //让父容器不拦截 ACTION_DOWN 的后续事件
            parent.requestDisallowInterceptTouchEvent(true)
        }
        MotionEvent.ACTION_MOVE -> {
            if (父容器需要此事件) {
                //让父容器拦截后续事件
                parent.requestDisallowInterceptTouchEvent(false)
            }
        }
        MotionEvent.ACTION_UP -> {
        }
    }
    return super.dispatchTouchEvent(event)
}
```

由于 ViewGroup 的 dispatchTouchEvent 方法会预先判断子 View 是否有要求其不拦截事件，如果没有的话才会调用自身的 onInterceptTouchEvent 方法，所以除了 ACTION_DOWN 外，如果子 View 不拦截的话那么 ViewGroup 都进行拦截

```kotlin
override fun onInterceptTouchEvent(event: MotionEvent): Boolean {
    return event.action != MotionEvent.ACTION_DOWN
}
```

# 十、解决滑动冲突

我经常会在网上看到一些开发者在问怎么解决 ScrollView 嵌套 ScrollView 后内部 ScrollView 无法滑动的问题，有这问题就是因为发生了滑动冲突，根本原因就是因为用户的滑动操作都被外部 ScrollView 拦截并消费了，导致内部 ScrollView 一直无法响应滑动事件。这里就以 ScrollView 嵌套 ScrollView 的情况作为例子，来看看怎么解决它们之间的滑动冲突问题

页面布局如下所示，内部的 ScrollView 是无法单独滑动的，只能随着外部 ScrollView 一起上下滑动

```xml
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

        <ScrollView
            android:layout_width="match_parent"
            android:layout_height="200dp">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical">

                <include layout="@layout/item_content_b" />

                <include layout="@layout/item_content_b" />

                <include layout="@layout/item_content_b" />

                <include layout="@layout/item_content_b" />

                <include layout="@layout/item_content_b" />

                <include layout="@layout/item_content_b" />

            </LinearLayout>

        </ScrollView>

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

        <include layout="@layout/item_content_a" />

    </LinearLayout>

</ScrollView>
```

这里选择使用内部拦截法来解决问题。首先需要让外部 ScrollView 拦截 ACTION_DOWN 之外的任何事件

```kotlin
class ExternalScrollView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : ScrollView(context, attrs, defStyleAttr) {

    override fun onInterceptTouchEvent(motionEvent: MotionEvent): Boolean {
        val intercepted: Boolean
        when (motionEvent.action) {
            MotionEvent.ACTION_DOWN -> {
                intercepted = false
                super.onInterceptTouchEvent(motionEvent)
            }
            else -> {
                intercepted = true
            }
        }
        return intercepted
    }

}
```

内部 ScrollView 判断自身是否还处于可滑动状态，如果滑动到了最顶部还想再往下滑动，或者是滑动到了最底部还想再往上滑动，那么就将事件都交由外部 ScrollView 处理，其它情况都直接拦截并消费掉事件，这样内部 ScrollView 就可以实现内部滑动了

```kotlin
class InsideScrollView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : ScrollView(context, attrs, defStyleAttr) {

    private var lastX = 0f

    private var lastY = 0f

    override fun dispatchTouchEvent(motionEvent: MotionEvent): Boolean {
        val x = motionEvent.x
        val y = motionEvent.y
        when (motionEvent.action) {
            MotionEvent.ACTION_DOWN -> {
                parent.requestDisallowInterceptTouchEvent(true)
            }
            MotionEvent.ACTION_MOVE -> {
                val deltaX = x - lastX
                val deltaY = y - lastY
                if (abs(deltaX) < abs(deltaY)) { //上下滑动的操作
                    if (deltaY > 0) { //向下滑动
                        if (scrollY == 0) { //滑动到顶部了
                            parent.requestDisallowInterceptTouchEvent(false)
                        }
                    } else { //向上滑动
                        if (height + scrollY >= computeVerticalScrollRange()) { //滑动到底部了
                            parent.requestDisallowInterceptTouchEvent(false)
                        }
                    }
                }
            }
            MotionEvent.ACTION_UP -> {
            }
        }
        lastX = x
        lastY = y
        return super.dispatchTouchEvent(motionEvent)
    }

}
```

# 十一、提问环节

## 事件为什么是由外向内？

在上面提供的例子里，当点击 MyTextView 区域时，最外层的 MyRelativeLayout 还是会最先接收到触摸事件。那么，为什么 Android 系统要将事件分发机制设计成由外向内的形式呢？能不能是由内向外的形式？或者是直接只交于点击区域所在的 View 进行处理呢？

**将触摸事件只交于点击区域所在的 View 进行处理是肯定不行的**。想像个场景，一个 ViewPager 包含了多个 Fragment，每个 Fragment 均包含一个 RecyclerView，如果将触摸事件只交于 RecyclerView 处理的话，那么 RecyclerView 可以正常响应上下滑动的事件，但是 ViewPager 就无法左右滑动了，因为左右滑动的事件都被 RecyclerView 给消费掉了，即使该事件对于 RecyclerView 本身来说是不需要的。所以事件分发机制必须要包含**一个在父容器和子内容区域之间流转触摸事件的流程**，各个 View 根据各自所需来进行选择性消费

**那能不能是由内向外的形式呢？也不合适。**一个 ViewGroup 可能包含一个到多个 View，ViewGroup 需要通过判断触摸点的坐标系位于哪个 View 区域内来确定触摸事件的下一个接收者。而我们知道，触摸事件按照从外向内的传递顺序是： DecorView -> Activity -> PhoneWindow -> DecorView -> ContentView，由于触摸事件的早期接收者已经是处于外层的 DecorView 了，所以按照从外向内进行传递会更加合适（这也只是我自己的个人见解，有误的话欢迎指出）

## mFirstTouchTarget 怎么设计的？

由于触摸事件的发生频率是很高的，且布局的嵌套层次也可能很深，如果每次在下发事件时都进行全量遍历的话不利于提升绘制效率。为了提高事件的下发效率并减少对象的重复创建，ViewGroup 中声明了一个 TouchTarget 类型的全局变量，即 mFirstTouchTarget

mFirstTouchTarget 中的 child 变量指向消费了触摸事件的下游 View，每个层级的 ViewGroup 都通过 mFirstTouchTarget 来指向下游，这样当后续事件到来时，就不必通过 DFS 算法再次进行遍历了，通过 mFirstTouchTarget 将后续事件层层往下传递给最终的消费者

此外，TouchTarget 中的静态成员变量 sRecycleBin 就用于提供对象复用功能，以链表的形式最多缓存 MAX_RECYCLED 个对象，调用 `obtain` 方法的时候就会以切换 next 引用的形式来获取一个独立的 TouchTarget 对象

```java
private static final class TouchTarget {
    private static final int MAX_RECYCLED = 32;
    private static final Object sRecycleLock = new Object[0];
    private static TouchTarget sRecycleBin;
    private static int sRecycledCount;

    public static final int ALL_POINTER_IDS = -1; // all ones

    // The touched child view.
    @UnsupportedAppUsage public View child;

    // The combined bit mask of pointer ids for all pointers captured by the target.
    public int pointerIdBits;

    // The next target in the target list.
    public TouchTarget next;

    @UnsupportedAppUsage
    private TouchTarget() {}

    public static TouchTarget obtain(@NonNull View child, int pointerIdBits) {
        if (child == null) {
            throw new IllegalArgumentException("child must be non-null");
        }
        final TouchTarget target;
        synchronized (sRecycleLock) {
            if (sRecycleBin == null) {
                target = new TouchTarget();
            } else {
                target = sRecycleBin;
                sRecycleBin = target.next;
                sRecycledCount--;
                target.next = null;
            }
        }
        target.child = child;
        target.pointerIdBits = pointerIdBits;
        return target;
    }

    public void recycle() {
        if (child == null) {
            throw new IllegalStateException("already recycled once");
        }
        synchronized (sRecycleLock) {
            if (sRecycledCount < MAX_RECYCLED) {
                next = sRecycleBin;
                sRecycleBin = this;
                sRecycledCount += 1;
            } else {
                next = null;
            }
            child = null;
        }
    }
}
```

## 讲下 ACTION_CANCEL 事件？

按照正常情况来说，每个事件序列应该是都只交由一个 View 或者 ViewGroup 进行消费的，可是还存在一种特殊情况，即 View 消费了 ACTION_DOWN 事件，而后续的 ACTION_MOVE 和 ACTION_UP 事件被其上层容器 ViewGroup 拦截了，导致 View 接收不到后续事件。这会导致一些异常问题， 例如，Button 在接收到 ACTION_DOWN 事件后 UI 后呈现按压状态，如果接收不到 ACTION_UP 这个结束事件的话可能就无法恢复 UI 状态了。为了解决这个问题，Android 系统就通过 ACTION_CANCEL 事件来作为事件序列的另外一种结束消息

当存在上诉情况时，ViewGroup 就会通过 `dispatchTransformedTouchEvent` 方法构造一个 ACTION_CANCEL 事件并将之下发给 View，从而使得 View 即使没有接受到 ACTION_UP 事件也可以知道本次事件序列已经结束了

```java
private boolean dispatchTransformedTouchEvent(
        MotionEvent event, boolean cancel, View child, int desiredPointerIdBits) {
    final boolean handled;

    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    ···
}
```

同时，ViewGroup 也会将 View 从 mFirstTouchTarget 中移除，这样后续事件也就不会再尝试向 View 下发了

## onUserInteraction 方法的作用？

前文有讲到，Activity 提供了一个空实现的 `onUserInteraction` 方法，向子类提供了 ACTION_DOWN 事件的触发通知，那么该方法能够用来做什么呢？

`onUserInteraction` 方法在 Activity 接收到 ACTION_DOWN 事件的时候才会被调用，这可以用于某些需要知道 Activity 是否处于长期“闲置”状态的需求。例如，如果我们需要在 Activity 没有被操作一段时间后自动隐藏标题栏的话，就可以用该方法来设置一个定时任务控制标题栏的隐藏状态

# 十二、Demo 下载

上述的所有示例代码我都放到了 Github 上，按需自取：[AndroidOpenSourceDemo](https://github.com/leavesczy/AndroidOpenSourceDemo)
