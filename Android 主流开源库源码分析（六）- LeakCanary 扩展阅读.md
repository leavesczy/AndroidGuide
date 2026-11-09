> 对于 Android Developer 来说，很多开源库都是属于**开发必备**的知识点，从使用方式到实现原理再到源码解析，这些都需要我们有一定程度的了解和运用能力。所以我打算来写一系列关于开源库**源码解析**和**实战演练**的文章，初定的目标是 **EventBus、ARouter、LeakCanary、Retrofit、Glide、OkHttp、Coil** 等七个知名开源库，希望对你有所帮助

上篇文章对 LeakCanary 进行了一次比较全面的源码解析，按流程来说本篇文章应该是属于实战篇的，可是由于某些原因就不打算写实战内容了（其实就是自己有点菜，写不太出来），就还是来写一篇关于内存泄露相关的扩展阅读吧

Java 的一个很显著的优点就在于内存自动回收机制，Java 通过垃圾收集器(Garbage Collection，GC)来自动管理内存的回收过程，而无需开发者来主动释放内存。这种自动化行为有效地节省了开发人员的开发成本，但也让一些开发者误以为 Java 就不存在**内存泄漏**的问题了，或者是误认为内存泄露应该是 GC 或者 JVM 层面来关心和解决的问题。这种想法是不正确的，因为内存泄露大多时候是由于程序本身存在缺陷而导致的，GC 和 JVM 并无法精准理解程序的实现初衷，所以还是需要由开发人员来主动解决问题

# 一、内存泄露和内存溢出

**内存泄露(Memory Leak)** 和 **内存溢出(Out Of Memory)** 两个概念经常会一起被提及，两者有相互关联的地方，但实质上还是有着很大的区别：

- 内存泄露。内存泄漏属于代码错误，这种错误会导致应用程序长久保留那些不再被需要的对象的引用，从而导致分配给该对象的内存无法被回收，造成程序的可用内存逐步降低，严重时甚至会导致 OOM。例如，当 Activity 的 onDestroy() 方法被调用后，正常来说 Activity 本身以及它涉及到的 View、Bitmap 等对象都应该被回收。但如果有一个后台线程持续持有对这个 Activity 的引用的话，那么 Activity 占据的内存就无法被回收，严重时将导致 OOM，最终 Crash
- 内存溢出。指一个应用在申请内存时，系统没有足够的内存空间可以供其使用

两者都会导致应用运行出现问题、性能下降或崩溃。不同点主要在于：

- 内存泄露是导致内存溢出的原因之一，内存泄露严重时将导致内存溢出
- 内存泄露是由于代码缺陷引起的，可以通过完善代码来避免；内存溢出可以通过调整配置来减少发生频率，但无法彻底避免

对于一个存在内存泄露的程序来说，即使每次仅会泄露少量内存，程序的可用内存也是会逐步降低，在长期运行过后，程序也是隐藏着崩溃的危险

# 二、内存管理

为了判断程序是否存在内存泄露的情况，我们首先必须先了解 Java 是如何管理内存的，Java 的内存管理就是对象的分配和释放过程

在 Java 中，我们都是通过关键字 new 来申请内存空间并创建对象的（基本类型除外），所有的对象都在堆 (Heap)中分配空间。总的来说，Java 的内存区域可以分为三类：

1. 静态存储区：在程序整个运行期间都存在，编译时就分配好空间，主要用于存放静态数据和常量
2. 栈区：当一个方法被执行时会在栈区内存中创建方法体内部的局部变量，方法结束后自动释放内存
3. 堆区：通常存放 new 出来的对象

对象的释放则由 GC 来完成。GC 负责监控每一个对象的运行状态，包括对象的申请、引用、被引用、赋值等行为。当某个对象被 GC 判断为不再被引用了时，GC 就会回收并释放该对象对应的内存空间

一个对象的引用方式可以分为四种：

1. 强引用(StrongReference)：JVM 宁可抛出 OOM 也不会让 GC 回收具有强引用的对象
2. 软引用(SoftReference)：如果一个对象只具有软引用，那么在内存空间不足时就会回收该对象
3. 弱引用(WeakReference)：如果一个对象只具有弱引用，那么在 GC 时不管当前内存空间是否足够，都会回收该对象
4. 虚引用(PhantomReference)：任何时候都可以被 GC 回收，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否存在该对象的虚引用，来了解这个对象是否将要被回收

而一个对象不再被引用的标记就是其不再被强引用，JVM 会通过**引用计数法**或者是**可达性分析**等方法来判断一个对象是否还被强引用着

在 Java 中，内存泄露的就意味着发生了这么一种情况：一个对象是可达的，存在其它对象强引用着该对象，但该对象是无用的，程序以后不会再使用这些对象。满足这种情况的对象就意味着该对象已经泄露，该对象不会被 GC 所回收（因为该对象可达，还未达到 GC 的标准），然而却一直持续占用着内存。例如，由于非静态内部类会持有对外部类的隐式引用，所以当非静态内部类在被回收之前，外部类也无法被回收

# 三、常见的内存泄露

以下列举九种常见的内存泄露场景及相应的解决方案，内容来自于国外的一篇文章：[9 ways to avoid memory leaks in Android](https://android.jlelse.eu/9-ways-to-avoid-memory-leaks-in-android-b6d81648e35e)

## 1、Broadcast Receivers

如果在 Activity 中注册了 BroadcastReceiver 而忘记了 **unregister** 的话，BroadcastReceiver 就将一直持有对 Activity 的引用，即使 Activity 已经执行了 `onDestroy`

```java
public class BroadcastReceiverLeakActivity extends AppCompatActivity {

    private BroadcastReceiver broadcastReceiver;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);
    }

    private void registerBroadCastReceiver() {
        broadcastReceiver =
                new BroadcastReceiver() {
                    @Override
                    public void onReceive(Context context, Intent intent) {
                        // your receiver code goes here!
                    }
                };
        registerReceiver(broadcastReceiver, new IntentFilter("SmsMessage.intent.MAIN"));
    }

    @Override
    protected void onStart() {
        super.onStart();
        registerBroadCastReceiver();
    }

    @Override
    protected void onStop() {
        super.onStop();

        /*
         * Uncomment this line in order to avoid memory leak.
         * You need to unregister the broadcast receiver since the broadcast receiver keeps a reference of the activity.
         * Now when its time for your Activity to die, the Android framework will call onDestroy() on it
         * but the garbage collector will not be able to remove the instance from memory because the broadcastReceiver
         * is still holding a strong reference to it.
         * */

        if (broadcastReceiver != null) {
            unregisterReceiver(broadcastReceiver);
        }
    }
}
```

开发者必须谨记在 `Activity.onStop()` 的时候调用 `unregisterReceiver`。但需要注意的是，如果 BroadcastReceiver 是在 `onCreate()` 中进行注册的，那么当应用进入后台并再次切换回来时，BroadcastReceiver 将不会被再次注册。所以，最好在 Activity 的 `onStart()` 或者 `onResume()` 方法中进行注册，然后在 `onStop()` 时进行注销

## 2、Static Activity or View Reference

看下面的示例代码，将 TextView 声明为了静态变量（无论出于什么原因）。不管是直接还是间接通过静态变量引用了 Activity 或者 View，在 Activity 被销毁后都无法对其进行垃圾回收

```java
public class StaticReferenceLeakActivity extends AppCompatActivity {

    /*
     * This is a bad idea!
     */
    private static TextView textView;
    private static Activity activity;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        textView = findViewById(R.id.activity_text);
        textView.setText("Bad Idea!");

        activity = this;
    }
}
```

永远不要通过静态变量来引用 Activity、View 和 Context

## 3、Singleton Class Reference

看下面的例子，定义了一个 Singleton 类，该类需要传递 Context 以便从本地存储中获取一些文件

```java
public class SingletonLeakExampleActivity extends AppCompatActivity {

    private SingletonSampleClass singletonSampleClass;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        /*
         * Option 1: Do not pass activity context to the Singleton class. Instead pass application Context
         */
        singletonSampleClass = SingletonSampleClass.getInstance(this);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

        /*
         * Option 2: Unregister the singleton class here i.e. if you pass activity context to the Singleton class,
         * then ensure that when the activity is destroyed, the context in the singleton class is set to null.
         */
        singletonSampleClass.onDestroy();
    }
}
```

```java
public class SingletonSampleClass {
  
    private Context context;
    private static SingletonSampleClass instance;
  
    private SingletonSampleClass(Context context) {
        this.context = context;
    }

    public synchronized static SingletonSampleClass getInstance(Context context) {
        if (instance == null) instance = new SingletonSampleClass(context);
        return instance;
    }
  
    public void onDestroy() {
       if(context != null) {
          context = null; 
       }
    }
}
```

此时如果没有主动将 SingletonSampleClass 包含的 context 置空的话，就将导致内存泄露。那如何解决这个问题？

- 可以传递 ApplicationContext，而不是将 ActivityContext 传递给 singleton 类
- 如果真的必须使用 ActivityContext，那么当 Activity 被销毁的时候，需要确保传递将 singleton 类的 Context 设置为 null

## 4、Inner Class Reference

看下面的例子，定义了一个 LeakyClass 类，你需要传递 Activity 才能重定向到新的 Activity

```java
public class InnerClassReferenceLeakActivity extends AppCompatActivity {

    /*
     * Mistake Number 1:
     * Never create a static variable of an inner class
     * Fix I:
     * private LeakyClass leakyClass;
     */
    private static LeakyClass leakyClass;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        new LeakyClass(this).redirectToSecondScreen();

        /*
         * Inner class is defined here
         * */
        leakyClass = new LeakyClass(this);
        leakyClass.redirectToSecondScreen();
    }

    /*
     * Mistake Number 2:
     * 1. Never create a inner variable of an inner class
     * 2. Never pass an instance of the activity to the inner class
     */
    private class LeakyClass {

        private Activity activity;

        public LeakyClass(Activity activity) {
            this.activity = activity;
        }

        public void redirectToSecondScreen() {
            this.activity.startActivity(new Intent(activity, SecondActivity.class));
        }
    }
}
```

如何解决这个问题？

- 就如之前所述，不要创建内部类的静态变量
- LeakyClass 设置为静态类，静态内部类不会持有对其外部类的隐式引用
- 对任何 View/Activity 都使用 weakReference。如果只有弱引用指向某个对象，那么垃圾回收器就可以回收该对象

```java
public class InnerClassReferenceLeakActivity extends AppCompatActivity {

    /*
     * Mistake Number 1:
     * Never create a static variable of an inner class
     * Fix I:
     */
    private LeakyClass leakyClass;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        new LeakyClass(this).redirectToSecondScreen();

        /*
         * Inner class is defined here
         * */
        leakyClass = new LeakyClass(this);
        leakyClass.redirectToSecondScreen();
    }

    /*
     * How to fix the above class:
     * Fix memory leaks:
     * Option 1: The class should be set to static
     * Explanation: Instances of anonymous classes do not hold an implicit reference to their outer class
     * when they are "static".
     *
     * Option 2: Use a weakReference of the textview or any view/activity for that matter
     * Explanation: Weak References: Garbage collector can collect an object if only weak references
     * are pointing towards it.
     * */
    private static class LeakyClass {

        private final WeakReference<Activity> messageViewReference;

        public LeakyClass(Activity activity) {
            this.activity = new WeakReference<>(activity);
        }

        public void redirectToSecondScreen() {
            Activity activity = messageViewReference.get();
            if (activity != null) {
                activity.startActivity(new Intent(activity, SecondActivity.class));
            }
        }
    }
}
```

## 5、Anonymous Class Reference

匿名内存类带来的内存泄漏问题和上一节内容相同，解决办法如下所示：

```java
public class AnonymousClassReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        textView = findViewById(R.id.activity_text);
        textView.setText(getString(R.string.text_inner_class_1));
        findViewById(R.id.activity_dialog_btn).setVisibility(View.INVISIBLE);

        /*
         * Runnable class is defined here
         * */
        new Thread(new LeakyRunnable(textView)).start();
    }

    private static class LeakyRunnable implements Runnable {

        private final WeakReference<TextView> messageViewReference;

        private LeakyRunnable(TextView textView) {
            this.messageViewReference = new WeakReference<>(textView);
        }

        @Override
        public void run() {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            TextView textView = messageViewReference.get();
            if (textView != null) {
                textView.setText("Runnable class has completed its work");
            }
        }
    }
}
```

## 6、AsyncTask Reference

看下面的示例，通过 AsyncTask 来获取一个字符串值，该值用于在 `onPostExecute()` 方法中更新 textView

```java
public class AsyncTaskReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;
    private BackgroundTask backgroundTask;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        /*
         * Executing AsyncTask here!
         * */
        backgroundTask = new BackgroundTask(textView);
        backgroundTask.execute();
    }

    /*
     * Couple of things we should NEVER do here:
     * Mistake number 1. NEVER reference a class inside the activity. If we definitely need to, we should set the class as static as static inner classes don’t hold
     *    any implicit reference to its parent activity class
     * Mistake number 2. We should always cancel the asyncTask when activity is destroyed. This is because the asyncTask will still be executing even if the activity
     *    is destroyed.
     * Mistake number 3. Never use a direct reference of a view from acitivty inside an asynctask.
     * */
    private class BackgroundTask extends AsyncTask<Void, Void, String> {
        @Override
        protected String doInBackground(Void... voids) {

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "The task is completed!";
        }

        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
            textView.setText(s);
        }
    }
}
```

如何解决这个问题？

- 当 Activity 被销毁时，我们应该取消异步任务，这是因为即使 Activity 已经走向 Destoryed，未结束的 AsyncTask 仍将继续执行
- 永远不要在 Activity 中引用内部类。如果确实需要，我们应该将其设置为静态类，因为静态内部类不会包含对其外部类的任何隐式引用
- 通过 weakReference 来引用 textview

```java
public class AsyncTaskReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;
    private BackgroundTask backgroundTask;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        /*
         * Executing AsyncTask here!
         * */
        backgroundTask = new BackgroundTask(textView);
        backgroundTask.execute();
    }

    /*
     * Fix number 1
     * */
    private static class BackgroundTask extends AsyncTask<Void, Void, String> {

        private final WeakReference<TextView> messageViewReference;

        private BackgroundTask(TextView textView) {
            this.messageViewReference = new WeakReference<>(textView);
        }

        @Override
        protected String doInBackground(Void... voids) {

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "The task is completed!";
        }

        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
            /*
             * Fix number 3
             * */
            TextView textView = messageViewReference.get();
            if (textView != null) {
                textView.setText(s);
            }
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

        /*
         * Fix number 2
         * */
        if (backgroundTask != null) {
            backgroundTask.cancel(true);
        }
    }
}
```

## 7、Handler Reference

看下面的例子，通过 Handler 在五秒后更新 UI

```java
public class HandlersReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;

    /*
     * Mistake Number 1
     * */
     private Handler leakyHandler = new Handler();

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        /*
         * Mistake Number 2
         * */
        leakyHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                textView.setText(getString(R.string.text_handler_1));
            }
        }, 5000);
    }
```

如何解决这个问题？

- 永远不要在 Activity 中引用内部类。如果确实需要，我们应该将其设置为静态类。这是因为当 Handler 在主线程上实例化后，它将与 Looper 的 MessageQueue 相关联，发送到 MessageQueue 的 Message 将持有对 Handler 的引用，以便当 Looper 最终处理消息时，framework 可以调用 Handler#handleMessage(message) 方法
- 通过 weakReference 来引用 Activity

```java
public class HandlersReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;

    /*
     * Fix number I
     * */
    private final LeakyHandler leakyHandler = new LeakyHandler(this);

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        leakyHandler.postDelayed(leakyRunnable, 5000);
    }

    /*
     * Fix number II - define as static
     * */
    private static class LeakyHandler extends Handler {
      
    /*
     * Fix number III - Use WeakReferences
     * */      
        private WeakReference<HandlersReferenceLeakActivity> weakReference;
        public LeakyHandler(HandlersReferenceLeakActivity activity) {
            weakReference = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            HandlersReferenceLeakActivity activity = weakReference.get();
            if (activity != null) {
                activity.textView.setText(activity.getString(R.string.text_handler_2));
            }
        }
    }

    private static final Runnable leakyRunnable = new Runnable() {
        @Override
        public void run() { /* ... */ }
    }
```

## 8、Threads Reference

Thread 和 TimerTask 也可能会导致内存泄露问题

```java
public class ThreadReferenceLeakActivity extends AppCompatActivity {

    /*  
     * Mistake Number 1: Do not use static variables
     * */    
    private static LeakyThread thread;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        createThread();
        redirectToNewScreen();
    }

    private void createThread() {
        thread = new LeakyThread();
        thread.start();
    }

    private void redirectToNewScreen() {
        startActivity(new Intent(this, SecondActivity.class));
    }

    /*
     * Mistake Number 2: Non-static anonymous classes hold an 
     * implicit reference to their enclosing class.
     * */
    private class LeakyThread extends Thread {
        @Override
        public void run() {
            while (true) {
            }
        }
    }
```

如何解决这个问题？

- 非静态匿名类会包含对其外部类的隐式引用，将 LeakyThread 改为静态内部类
- 在 Activity 的 onDestroy() 方法中停止线程，以避免线程泄漏

```java
public class ThreadReferenceLeakActivity extends AppCompatActivity {

    /*
     * FIX I: make variable non static
     * */
    private LeakyThread leakyThread = new LeakyThread();

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        createThread();
        redirectToNewScreen();
    }

    private void createThread() {
        leakyThread.start();
    }

    private void redirectToNewScreen() {
        startActivity(new Intent(this, SecondActivity.class));
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // FIX II: kill the thread
        leakyThread.interrupt();
    }

    /*
     * Fix III: Make thread static
     * */
    private static class LeakyThread extends Thread {
        @Override
        public void run() {
            while (!isInterrupted()) {}
        }
    }
}
```

## 9、TimerTask Reference

对于 TimerTask 也可以遵循相同的原则，修复内存泄漏的示例如下所示：

```java
public class TimerTaskReferenceLeakActivity extends Activity {

    private CountDownTimer countDownTimer;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        startTimer();
    }

    /*
     * Mistake 1: Cancel Timer is never called
     * even though activity might be completed
     * */
    public void cancelTimer() {
        if (countDownTimer != null) countDownTimer.cancel();
    }

    private void startTimer() {
        countDownTimer =
                new CountDownTimer(1000, 1000) {
                    @Override
                    public void onTick(long millisUntilFinished) {
                        final int secondsRemaining = (int) (millisUntilFinished / 1000);
                        // update UI
                    }

                    @Override
                    public void onFinish() {
                        // handle onFinish
                    }
                };
        countDownTimer.start();
    }
}
```

如何解决这个问题？

- 在 Activity 的 onDestroy() 方法中停止计时器，以避免内存泄漏

```java
public class TimerTaskReferenceLeakActivity extends Activity {

    private CountDownTimer countDownTimer;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        startTimer();
    }

    public void cancelTimer() {
        if (countDownTimer != null) countDownTimer.cancel();
    }

    private void startTimer() {
        countDownTimer =
                new CountDownTimer(1000, 1000) {
                    @Override
                    public void onTick(long millisUntilFinished) {
                        final int secondsRemaining = (int) (millisUntilFinished / 1000);
                        // update UI
                    }

                    @Override
                    public void onFinish() {
                        // handle onFinish
                    }
                };
        countDownTimer.start();
    }

    /*
     * Fix 1: Cancel Timer when
     * activity might be completed
     * */
    @Override
    protected void onDestroy() {
        super.onDestroy();
        cancelTimer();
    }
}
```

## 10、总结

最后再来简单总结一下：

1. 尽可能使用 ApplicationContext 而不是 ActivityContext。如果真的必须使用 ActivityContext，那么当 Activity 被销毁时，请确保将传递的 Context 置为 null
2. 不要通过静态变量来引用 View 和 Activity
3. 不要在 Activity 中引用内部类，如果确实需要，那么应该将它声明为静态的，不管它是 Thread、Handler、Timer 还是 AsyncTask
4. 务必记住注销 Activity 中的 BroadcastReceiver 和 Timer，在 onDestroy() 方法中取消任何异步任务和线程
5. 通过 weakReference 来持有对 Activity 和 View 的引用