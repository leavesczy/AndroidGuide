在蛮久前有同事问过我关于一个 Gson 和 Kotlin dataClass 的问题，当时答不上来也没去细究，但一直都放在心底，今天就认真探究下原因，也输出总结了一下，希望能帮助你避开这个坑

来看个小例子，猜猜其运行结果会是怎样的

```kotlin
data class UserBean(val userName: String, val userAge: Int)

fun main() {
    val json = """{"userName":null,"userAge":26}"""  
    val userBean = Gson().fromJson(json, UserBean::class.java) //第一步
    println(userBean) //第二步
    printMsg(userBean.userName) //第三步
}

fun printMsg(msg: String) {

}
```
UserBean 是一个 dataClass，其 userName 字段被声明为非 null 类型，而 json 字符串中 userName 对应的值明确就是 null，那用 Gson 到底能不能反序列化成功呢？程序能不能成功运行完以上三个步骤？

实际上程序能够正常运行到第二步，但在执行第三步的时候反而直接报 NPE 异常了

```kotlin
UserBean(userName=null, userAge=26)
Exception in thread "main" java.lang.NullPointerException: Parameter specified as non-null is null: method temp.TestKt.printMsg, parameter msg
    at temp.TestKt.printMsg(Test.kt)
    at temp.TestKt.main(Test.kt:16)
    at temp.TestKt.main(Test.kt)
```
# 一、为啥会抛出 NEP？

`printMsg` 方法接收了参数后实际上什么也没做，为啥会抛出 NPE？

通过 IDEA 将 `printMsg` 反编译为 Java 方法，可以发现方法内部会对入参进行空校验，当发现为 null 时就会直接抛出 NullPointerException

```java
public static final void printMsg(@NotNull String msg) {
    Intrinsics.checkNotNullParameter(msg, "msg");
}
```
这个比较好理解，毕竟 Kotlin 的类型系统会严格区分**可 null** 和**不可为 null** 两种类型，其区分手段之一就是会自动在我们的代码里插入一些类型校验逻辑，即自动加上了非空断言，当发现不可为 null 的参数传入了 null 的话就会马上就抛出 NPE，即使我们并没有用到该参数

当然，这个自动插入的校验逻辑只会在 Kotlin 代码中生成，如果我们是将 `userBean.userName` 传给 Java 方法的话，就不会有这个效果，而是会等到我们使用到了该参数时才发生 NPE

# 二、Kotlin 的 nullSafe 失效了吗？

既然 UserBean 中的 userName 字段已经被声明为非 null 类型了，那么为什么还可以反序列化成功呢？按照我自己的第一直觉，应该在进行反序列的时候就直接抛出异常才对，Gson 是怎么绕过 Kotlin 的 null 检查的呢？

这个需要来看下 Gson 是如何实现反序列的

通过断点，可以发现 UserBean 是在 ReflectiveTypeAdapterFactory 里完成构建的，这里的主要步骤就分为两步：

- 通过 `constructor.construct()` 得到一个 UserBean 对象，此时该对象内部的属性值都为默认值
- 遍历 JsonReader，根据 Json 内部的 key 值和 UserBean 包含的字段进行对应，对应得上的话就进行赋值

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8b74fe9e1124f7ca21525923749d3b7~tplv-k3u1fbpfcp-zoom-1.image)

第二步很好理解，那第一步又是具体怎么实现的？再断点看下 `constructor.construct()` 是如何实现的

constructor 的取值途径可以在 ConstructorConstructor 这个类中看到

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2205a73a877b46b9a293623f4c4ab71a~tplv-k3u1fbpfcp-zoom-1.image)

分为三种可能：

- newDefaultConstructor。通过反射无参构造函数来生成对象
- newDefaultImplementationConstructor。通过反射为 Collection 和 Map 等集合框架类型来生成对象
- newUnsafeAllocator。通过 Unsafe 包来生成对象，是最后兜底的方案

首先，第二个肯定不符合条件，看第一个和第三个就行

作为一个 dataClass，UserBean 是否有无参构造函数呢？反编译后可以看到是没有的，只有一个包含两个参数的构造函数，所以第一步也肯定会反射失败

```kotlin
public final class UserBean {
   @NotNull
   private final String userName;
   private final int userAge;

   @NotNull
   public final String getUserName() {
      return this.userName;
   }

   public final int getUserAge() {
      return this.userAge;
   }

   public UserBean(@NotNull String userName, int userAge) {
      Intrinsics.checkNotNullParameter(userName, "userName");
      super();
      this.userName = userName;
      this.userAge = userAge;
   }

   ···
    
}
```
此外，还有一种方法可以验证出来 UserBean 没有被调用到构造函数。我们知道，子类在通过构造函数来进行初始化的时候，肯定是需要先连锁调用父类的构造函数，那么就可以通过为 UserBean 声明一个父类，然后通过判断父类的 init 方法块是否有打印日志就可以知道 UserBean 是否有被调用到构造函数了

```kotlin
open class Person() {

    init {
        println("Person")
    }

}

data class UserBean(val userName: String, val userAge: Int) : Person()
```
前两种都无法满足，再来看 `newUnsafeAllocator` 是如何进行兜底的

Unsafe 是位于 `sun.misc` 包下的一个类，主要提供一些用于执行低级别、不安全操作的方法，如直接访问系统内存资源、自主管理内存资源等，这些方法在提升 Java 运行效率、增强 Java 语言底层资源操作能力方面起到了很大的作用。但由于 Unsafe 类使 Java 语言拥有了类似 C 语言指针一样操作内存空间的能力，这无疑也增加了程序发生相关指针问题的风险。在程序中过度、不正确使用 Unsafe 类会使得程序出错的概率变大，使得 Java 这种安全的语言变得不再安全，因此对 Unsafe 的使用一定要慎重

Unsafe 提供了一个非常规实例化对象的方法：`allocateInstance`，该方法提供了通过 Class 对象就可以创建出相应实例的功能，而且不需要调用其构造函数、初始化代码、JVM 安全检查等，即使构造函数是 private 的也能通过此方法进行实例化

Gson 的 UnsafeAllocator 类中就通过 `allocateInstance` 方法来完成了 UserBean 的初始化，因此也不会调用到其构造函数

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dfe8cc0d04ac4e3d8da970a7420867a1~tplv-k3u1fbpfcp-zoom-1.image)

做下总结：

- UserBean 的构造函数只有一个，其包含两个构造参数，在构造函数内部也对 userName 这个字段进行了 null 检查，当发现为 null 时会直接抛出 NPE
- Gson 是通过 Unsafe 包来实例化 UserBean 对象的，并不会调用到其构造函数，相当于绕过了 Kotlin 的 null 检查，所以即使 userName 值为 null 最终也能够反序列化成功

# 三、构造参数默认值失效？

再看个例子

如果我们为 UserBean 的 userName 字段设置了一个默认值，且 json 中不包含该 key，那么会发现默认值并不会生效，还是为 null

```kotlin
data class UserBean(val userName: String = "leavesC", val userAge: Int)

fun main() {
    val json = """{"userAge":26}"""
    val userBean = Gson().fromJson(json, UserBean::class.java)
    println(userBean)
}
```
```kotlin
UserBean(userName=null, userAge=26)
```
为构造参数设置默认值是一个很常见的需求，能降低使用者初始化对象的成本，而且如果将 UserBean 作为网络请求接口的承载体的话，接口可能不会返回该字段，此时也希望该字段能够有个默认值

通过上一节内容的分析，我们知道 Unsafe 包是不会调用 UserBean 的任何构造函数的，所以默认值也一定不会生效，那就只能找其它解决方案了，有以下几种方案可以解决：

## 1、无参构造函数

UserBean 提供一个**无参构造函数**，让 Gson 通过反射该函数来实例化 UserBean，从而同时进行默认值赋值

```kotlin
data class UserBean(val userName: String, val userAge: Int) {

    constructor() : this("leavesC", 0)

}
```
## 2、添加注解

可以通过向构造函数添加一个 `@JvmOverloads` 注解来解决，这种方式实际上也是通过提供一个无参构造函数来解决问题的。所以缺点就是需要每个构造参数都提供默认值，所以才能生成无参构造函数

```kotlin
data class UserBean @JvmOverloads constructor(
    val userName: String = "leavesC",
    val userAge: Int = 0
)
```
## 3、声明为字段

这种方式和前两种类似，也是通过间接提供一个无参构造函数来实现的。将所有字段都声明在类内部而非构造参数，此时声明的字段也一样具有默认值

```kotlin
class UserBean {

    var userName = "leavesC"

    var userAge = 0

    override fun toString(): String {
        return "UserBean(userName=$userName, userAge=$userAge)"
    }

}
```
## 4、改用 moshi

Gson 由于本身定位就是用于 Java 语言的，所以目前对于 Kotlin 的友好程度不高，导致默认值无法直接生效。我们可以改用另外一个 Json 序列化库：[moshi](https://github.com/square/moshi)

moshi 是 square 提供的一个开源库，对 Kotlin 的支持程度会比 Gson 高很多

导入依赖：

```groovy
dependencies {
    implementation 'com.squareup.moshi:moshi-kotlin:1.11.0'
}
```
此时不需要做特殊操作，在反序列化的时候默认值就可以直接生效

```kotlin
data class UserBean(val userName: String = "leavesC", val userAge: Int)

fun main() {
    val json = """{"userAge":26}"""
    val moshi = Moshi.Builder()
        .addLast(KotlinJsonAdapterFactory())
        .build()
    val jsonAdapter: JsonAdapter<UserBean> = moshi.adapter(UserBean::class.java)
    val userBean = jsonAdapter.fromJson(json)
    println(userBean)
}
```
```kotlin
UserBean(userName=leavesC, userAge=26)
```
但如果 json 字符串中 userName 字段明确返回了 null 的话，此时也会由于类型校验不通过导致直接抛出异常，而这严格来说也更加符合 Kotlin 风格

```kotlin
fun main() {
    val json = """{"userName":null,"userAge":26}"""
    val moshi = Moshi.Builder()
        .addLast(KotlinJsonAdapterFactory())
        .build()
    val jsonAdapter: JsonAdapter<UserBean> = moshi.adapter(UserBean::class.java)
    val userBean = jsonAdapter.fromJson(json)
    println(userBean)
}
```
```kotlin
Exception in thread "main" com.squareup.moshi.JsonDataException: Non-null value 'userName' was null at $.userName
    at com.squareup.moshi.internal.Util.unexpectedNull(Util.java:663)
    at com.squareup.moshi.kotlin.reflect.KotlinJsonAdapter.fromJson(KotlinJsonAdapter.kt:87)
    at com.squareup.moshi.internal.NullSafeJsonAdapter.fromJson(NullSafeJsonAdapter.java:41)
    at com.squareup.moshi.JsonAdapter.fromJson(JsonAdapter.java:51)
    at temp.TestKt.main(Test.kt:21)
    at temp.TestKt.main(Test.kt)
```
# 四、扩展知识

**再来看个扩展知识，和 Gson 无直接关联，但是在开发中也是蛮重要的一个知识点**

json 为空字符串，此时 Gson 可以成功反序列化，且得到的 userBean 为 null

```kotlin
fun main() {
    val json = ""
    val userBean = Gson().fromJson(json, UserBean::class.java)
}
```
如果加上类型声明：UserBean?，那也可以成功反序列化

```kotlin
fun main() {
    val json = ""
    val userBean: UserBean? = Gson().fromJson(json, UserBean::class.java)
}
```
如果加上的类型声明是 UserBean 的话，那就比较好玩了，会直接抛出 NullPointerException

```kotlin
fun main() {
    val json = ""
    val userBean: UserBean = Gson().fromJson(json, UserBean::class.java)
}
```
```kotlin
Exception in thread "main" java.lang.NullPointerException: Gson().fromJson(json, UserBean::class.java) must not be null
    at temp.TestKt.main(Test.kt:22)
    at temp.TestKt.main(Test.kt)
```
以上三个例子会有不同区别的原因是什么呢？

这就要牵扯到 Kotlin 的平台类型了。Kotlin 的一大特色就可以和 Java 实现百分百互通，平台类型是 Kotlin 对 Java 所作的一种平衡性设计。Kotlin 将对象的类型分为了可空类型和不可空类型两种，但 Java 平台的一切对象类型均为可空的，当在 Kotlin 中引用 Java 变量时，如果将所有变量均归为可空类型，最终将多出许多 null 检查；如果均看成不可空类型，那么就很容易就写出忽略了 NPE 风险的代码。为了平衡两者，Kotlin 引入了平台类型，即当在 Kotlin 中引用 Java 变量值时，既可以将之看成可空类型，也可以将之看成不可空类型，由开发者自己来决定是否进行 null 检查

因此，当我们从 Kotlin 承接 Gson 这个 Java 类返回的变量时，既可以将其当做 UserBean 类型，也可以当做 UserBean? 类型。而如果我们直接显式声明为 UserBean 类型，就说明我们确信返回的是非空类型，当返回的是 null 时就会触发 Kotlin 的 null 检查，导致直接抛出 NullPointerException

关于平台类型的知识点摘抄自我的另一篇 Kotlin 教程文章：[两万六千字带你 Kotlin 入门](https://juejin.cn/post/6880602489297895438#heading-36)