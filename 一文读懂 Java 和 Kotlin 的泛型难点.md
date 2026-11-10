Java 和 Kotlin 的泛型算作是一块挺大的知识难点了，涉及到很多很难理解的概念：**泛型型参、泛型实参、类型参数、不变、型变、协变、逆变、内联**等等。本篇文章就将 Java 和 Kotlin 结合着一起讲，按照我的个人理解来阐述泛型的各个知识难点，希望对你有所帮助

# 一、泛型类型

泛型允许你定义带**类型形参**的数据类型，当这种类型的实例被创建出来后，**类型形参**便被替换为称为**类型实参**的具体类型。例如，对于 `List<T>`，List 称为**基础类型**，T 便是**类型型参**，T 可以是任意类型，当没有指定 T 的具体类型时，我们只能知道 `List<T>` 是一个集合列表，但不知道承载的具体数据类型。而对于 `List<String>`，当中的 String 便是**类型实参**，我们可以明白地知道该列表承载的都是字符串，在这里 String 就相当于一个参数传递给了 List，在这语义下 String 也称为**类型参数**

此外，在 Kotlin 中我们可以实现**实化类型参数**，在运行时的**内联函数**中拿到作为**类型实参**的具体类型，即可以实现 `T::class.java`，但在 Java 中却无法实现，因为**内联函数**是 Kotlin 中的概念，Java 中并不存在

# 二、为什么需要泛型

泛型是在 Java 5 版本开始引入的，先通过几个小例子来明白泛型的重要性

以下代码可以成功编译，但是在运行时却抛出了 ClassCastException。了解 ArrayList 源码的同学就知道其内部是用一个 `Object[]` 数组来存储数据的，这使得 ArrayList 能够存储任何类型的对象，所以在没有泛型的年代开发者一不小心就有可能向 ArrayList 存入了非期望值，编译期完全正常，等到在运行时就会抛出类型转换异常了

```java
public class GenericTest {

    public static void main(String[] args) {
        List stringList = new ArrayList();
        addData(stringList);
        String str = (String) stringList.get(0);
    }

    public static void addData(List dataList) {
        dataList.add(1);
    }
}
```

```java
Exception in thread "main" java.lang.ClassCastException: 
java.lang.Integer cannot be cast to java.lang.String
```

而有了泛型后，我们就可以写出更加健壮安全的代码，以下错误就完全可以在编译阶段被发现，且取值的时候也不需要进行类型强转

```java
public static void main(String[] args) {
    List<String> stringList = new ArrayList();
    addData(stringList); // 报错
    String str = stringList.get(0);
}

public static void addData(List<Integer> dataList) {
    dataList.add(1);
}
```

此外，利用泛型我们可以写出更加具备通用性的代码。例如，假设我们需要从一个 List 中筛选出大于 0 的全部数字，那我们自然不想为 Integer、Float、Double 等多种类型各写一个筛选方法，此时就可以利用泛型来抽象筛选逻辑

```java
public static void main(String[] args) {
    List<Integer> integerList = new ArrayList<>();
    integerList.add(-1);
    integerList.add(1);
    integerList.add(2);
    List<Integer> result1 = filter(integerList);

    List<Float> floatList = new ArrayList<>();
    floatList.add(-1f);
    floatList.add(1f);
    floatList.add(2f);
    List<Float> result2 = filter(floatList);
}

public static <T extends Number> List<T> filter(List<T> data) {
    List<T> filterList = new ArrayList<>();
    for (T datum : data) {
        if (datum.doubleValue() > 0) {
            filterList.add(datum);
        }
    }
    return filterList;
}
```

总的来说，泛型有以下几点优势：

- 类型检查，在编译阶段就能发现错误
- 更加语义化，看到 `List<String>` 我们就知道存储的数据类型是 String
- 自动类型转换，在取值时无需进行手动类型转换
- 能够将逻辑抽象出来，使得代码更加具有通用性

# 三、类型擦除

泛型是在 Java 5 版本开始引入的，所以在 Java 4 中 ArrayList 还不属于泛型类，其内部通过 **Object 向上转型**和**外部强制类型转换**来实现数据存储和逻辑复用，此时开发者的项目中已经充斥了大量以下类型的代码：

```java
List stringList = new ArrayList();
stringList.add("业志陈");
stringList.add("https://juejin.cn/user/923245496518439");
String str = (String) stringList.get(0);
```

而在推出泛型的同时，Java 官方也必须保证二进制的向后兼容性，用 Java 4 编译出的 Class 文件也必须能够在 Java 5 上正常运行，即 Java 5 必须保证以下两种类型的代码能够在 Java 5 上共存且正常运行

```java
List stringList = new ArrayList();
List<String> stringList = new ArrayList();
```

为了实现这一目的，Java 就通过**类型擦除**这种比较别扭的方式来实现泛型。编译器在编译时会擦除类型实参，在运行时不存在任何类型相关的信息，泛型对于 JVM 来说是透明的，有泛型和没有泛型的代码通过编译器编译后所生成的二进制代码是完全相同的

例如，分别声明两个泛型类和非泛型类，拿到其 class 文件

```java
public class GenericTest {

    public static class NodeA {

        private Object obj;

        public NodeA(Object obj) {
            this.obj = obj;
        }
    }

    public static class NodeB<T> {

        private T obj;

        public NodeB(T obj) {
            this.obj = obj;
        }
    }

    public static void main(String[] args) {
        NodeA nodeA = new NodeA("业志陈");
        NodeB<String> nodeB = new NodeB<>("业志陈");
        System.out.println(nodeB.obj);
    }
}
```

可以看到 NodeA 和 NodeB 两个对象对应的字节码其实是完全一样的，最终都是使用 Object 来承载数据，就好像传递给 NodeB 的类型参数 String 不见了一样，这便是类型擦除

```java
public class generic.GenericTest {
  public generic.GenericTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class generic/GenericTest$NodeA
       3: dup
       4: ldc           #3                  // String 业志陈
       6: invokespecial #4                  // Method generic/GenericTest$NodeA."<init>":(Ljava/lang/Object;)V
       9: astore_1
      10: new           #5                  // class generic/GenericTest$NodeB
      13: dup
      14: ldc           #3                  // String 业志陈
      16: invokespecial #6                  // Method generic/GenericTest$NodeB."<init>":(Ljava/lang/Object;)V
      19: astore_2
      20: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
      23: aload_2
      24: invokestatic  #8                  // Method generic/GenericTest$NodeB.access$000:(Lgeneric/GenericTest$NodeB;)Ljava/lang/Object;
      27: checkcast     #9                  // class java/lang/String
      30: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      33: return
}
```

而如果让 NodeA 直接使用 String 类型，并且为泛型类 NodeB 设定上界约束 String，两者的字节码也会完全一样

```java
public class GenericTest {

    public static class NodeA {

        private String obj;

        public NodeA(String obj) {
            this.obj = obj;
        }
    }

    public static class NodeB<T extends String> {

        private T obj;

        public NodeB(T obj) {
            this.obj = obj;
        }
    }

    public static void main(String[] args) {
        NodeA nodeA = new NodeA("业志陈");
        NodeB<String> nodeB = new NodeB<>("业志陈");
        System.out.println(nodeB.obj);
    }
}
```

可以看到 NodeA 和 NodeB 的字节码是完全相同的

```java
public class generic.GenericTest {
  public generic.GenericTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class generic/GenericTest$NodeA
       3: dup
       4: ldc           #3                  // String 业志陈
       6: invokespecial #4                  // Method generic/GenericTest$NodeA."<init>":(Ljava/lang/String;)V
       9: astore_1
      10: new           #5                  // class generic/GenericTest$NodeB
      13: dup
      14: ldc           #3                  // String 业志陈
      16: invokespecial #6                  // Method generic/GenericTest$NodeB."<init>":(Ljava/lang/String;)V
      19: astore_2
      20: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
      23: aload_2
      24: invokestatic  #8                  // Method generic/GenericTest$NodeB.access$000:(Lgeneric/GenericTest$NodeB;)Ljava/lang/String;
      27: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      30: return
}
```

所以说，当泛型类型被擦除后有两种转换方式

- 如果泛型没有设置上界约束，那么将泛型转化成 Object 类型
- 如果泛型设置了上界约束，那么将泛型转化成该上界约束

该结论也可以通过反射泛型类的 Class 对象来验证

```java
public class GenericTest {

    public static class NodeA<T> {

        private T obj;

        public NodeA(T obj) {
            this.obj = obj;
        }
    }

    public static class NodeB<T extends String> {

        private T obj;

        public NodeB(T obj) {
            this.obj = obj;
        }
    }

    public static void main(String[] args) {
        NodeA<String> nodeA = new NodeA<>("业志陈");
        getField(nodeA.getClass());
        NodeB<String> nodeB = new NodeB<>("https://juejin.cn/user/923245496518439");
        getField(nodeB.getClass());
    }

    private static void getField(Class clazz) {
        for (Field field : clazz.getDeclaredFields()) {
            System.out.println("fieldName: " + field.getName());
            System.out.println("fieldTypeName: " + field.getType().getName());
        }
    }
}
```

NodeA 对应的是 Object，NodeB 对应的是 String

```java
fieldName: obj
fieldTypeName: java.lang.Object
fieldName: obj
fieldTypeName: java.lang.String
```

那既然在运行时不存在任何类型相关的信息，泛型又为什么能够实现**类型检查**和**类型自动转换**等功能呢？

其实，类型检查是编译器在**编译前**帮我们完成的，编译器知道我们声明的具体的类型实参，所以类型擦除并不影响类型检查功能。而类型自动转换其实是通过内部强制类型转换来实现的，上面给出的字节码中也可以看到有一条类型强转 checkcast 的语句

```java
27: checkcast     #9                  // class java/lang/String
```

例如，ArrayList 内部虽然用于存储数据的是 Object 数组，但 get 方法内部会自动完成类型强转

```java
transient Object[] elementData;

public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}

@SuppressWarnings("unchecked")
E elementData(int index) {
    // 强制类型转换
    return (E) elementData[index];
}
```

所以 Java 的泛型可以看做是一种特殊的语法糖，因此也被人称为**伪泛型**

# 四、类型擦除的后遗症

Java 泛型对于类型的约束只在编译期存在，运行时仍然会按照 Java 5 之前的机制来运行，泛型的具体类型在运行时已经被删除了，所以 JVM 是识别不到我们在代码中指定的具体的泛型类型的

例如，虽然 `List<String>` 只能用于添加字符串，但我们只能**泛化地**识别到它属于 `List<?>` 类型，而无法具体判断出该 List 内部包含的具体类型

```java
List<String> stringList = new ArrayList<>();
//正常
if (stringList instanceof ArrayList<?>) {

}
//报错
if (stringList instanceof ArrayList<String>) {

}
```

我们只能对具体的对象实例进行类型校验，但无法判断出泛型形参的具体类型

```java
public <T> void filter(T data) {
    // 正常
    if (data instanceof String) {}

    // 报错
    if (T instanceof String) {}

    // 报错
    Class<T> tClass = T::getClass;
}
```

此外，类型擦除也会导致 Java 中出现多态问题。例如，以下两个方法的方法签名并不完全相同，但由于类型擦除的原因，入参参数的数据类型都会被看成 `List<Object>`，从而导致两者无法共存在同一个区域内

```java
public void filter(List<String> stringList) {}

public void filter(List<Integer> stringList) {}
```

# 五、Kotlin 泛型

Kotlin 泛型在大体上和 Java 一致，毕竟两者需要保证兼容性

```kotlin
class Plate<T>(val t: T) {

    fun cut() {
        println(t.toString())
    }

}

class Apple

class Banana

fun main() {
    val plateApple = Plate<Apple>(Apple())
    //泛型类型自动推导
    val plateBanana = Plate(Banana())
    plateApple.cut()
    plateBanana.cut()
}
```

Kotlin 也支持在扩展函数中使用泛型

```kotlin
fun <T> List<T>.find(t: T): T? {
    val index = indexOf(t)
    return if (index > -1) get(index) else null
}
```

需要注意的是，为了实现向后兼容，目前高版本 Java 依然允许实例化没有具体类型参数的泛型类，这可以说是一个对新版本 JDK 危险但对旧版本友好的兼容措施。但 Kotlin 要求在使用泛型时需要**显式声明泛型类型**或者是**编译器能够类型推导出具体类型**，任何不具备具体泛型类型的泛型类都无法被实例化。因为 Kotlin 一开始就是基于 Java 6 版本的，一开始就存在了泛型，自然就不存在需要兼容老代码的问题，因此以下例子和 Java 会有不同的表现

```kotlin
val arrayList1 = ArrayList() //错误，编译器报错

val arrayList2 = arrayListOf<Int>() //正常

val arrayList3 = arrayListOf(1, 2, 3) //正常
```

还有一个比较容易让人误解的点。我们经常会使用 `as` 和 `as?` 来进行类型转换，但如果转换对象是泛型类型的话，那就会由于类型擦除而出现误判。如果转换对象有正确的**基础类型**，那么转换就会成功，而不管类型实参是否相符。因为在运行时转换发生的时候类型实参是未知的，此时编译器只会发出 “unchecked cast” 警告，代码还是可以正常编译的

例如，在以下例子中代码的运行结果还符合我们的预知。第一个转换操作由于类型相符，所以打印出了相加值。第二个转换操作由于基础类型是 Set 而非 List，所以抛出了 IllegalAccessException

```kotlin
fun main() {
    printSum(listOf(1, 2, 3)) //6
    printSum(setOf(1, 2, 3)) //IllegalAccessException
}

fun printSum(c: Collection<*>) {
    val intList = c as? List<Int> ?: throw IllegalAccessException("List is expected")
    println(intList.sum())
}
```

而在以下例子中抛出的却是 ClassCastException，这是因为在运行时不会判断且无法判断出类型实参到底是否是 Int，而只会判断基础类型 List 是否相符，所以 `as?` 操作会成功，等到要执行相加操作时才会发现拿到的是 String 而非 Number

```kotlin
printSum(listOf("1", "2", "3"))

Exception in thread "main" java.lang.ClassCastException: 
java.lang.String cannot be cast to java.lang.Number
```

# 六、上界约束

泛型本身已经带有类型约束的作用，我们也可以进一步细化其支持的具体类型

例如，假设存在一个盘子 Plate，我们要求该 Plate 只能用于装水果 Fruit，那么就可以对其泛型声明做进一步约束，Java 中使用 extend 关键字来声明约束规则，而 Kotlin 使用的是 **:** 。这样 Plate 就只能用于 Fruit 和其子类，而无法用于 Noodles 等不相关的类型，这种类型约束就被称为**上界约束**

```kotlin
open class Fruit

class Apple : Fruit()

class Noodles

class Plate<T : Fruit>(val t: T)

fun main() {
    val applePlate = Plate(Apple()) //正常
    val noodlesPlate = Plate(Noodles()) //报错
}
```

如果上界约束拥有多层类型元素，Java 是使用 & 符号进行链式声明，Kotlin 则是用 where 关键字来依次进行声明

```kotlin
interface Soft

class Plate<T>(val t: T) where T : Fruit, T : Soft

open class Fruit

class Apple : Fruit()

class Banana : Fruit(), Soft

fun main() {
    val applePlate = Plate(Apple()) //报错
    val bananaPlate = Plate(Banana()) //正常
}
```

此外，没有指定上界约束的类型形参会默认使用 Any? 作为上界，即我们可以使用 String 或 String? 作为具体的类型实参。如果想确保最终的类型实参一定是非空类型，那么就需要主动声明上界约束为 Any

# 七、类型通配符 & 星号投影

假设现在有个需求，需要我们提供一个方法用于遍历所有类型的 List 集合并打印元素

第一种做法就是直接将方法参数类型声明为 List，不包含任何泛型类型声明。这种做法可行，但编译器会警告无法确定 `list` 元素的具体类型，所以这不是最优解法

```java
public static void printList1(List list) {
    for (Object o : list) {
        System.out.println(o);
    }
}
```

可能会想到的第二种做法是：将泛型类型直接声明为 Object，希望让其适用于任何类型的 List。这种做法完全不可行，因为即使 `String` 是 `Object` 的子类，但 `List<String>` 和 `List<Object>` 并不具备从属关系，这导致 `printList2` 方法实际上只能用于 `List<Object>` 这一种具体类型

```java
public static void printList2(List<Object> list) {
    for (Object o : list) {
        System.out.println(o);
    }
}
```

最优解法就是要用到 Java 的类型通配符 **?** 了，`printList3` 方法完全可行且编译器也不会警告报错

```java
public static void printList3(List<?> list) {
    for (Object o : list) {
        System.out.println(o);
    }
}
```

？ 表示我们并不关心具体的泛型类型，而只是想配合其它类型进行一些条件限制。例如，`printList3` 方法希望传入的是一个 List，但不限制泛型的具体类型，此时 `List<?>` 就达到了这一层限制条件

类型通配符也存在着一些限制。因为 `printList3` 方法并不包含具体的泛型类型，所以我们从中取出的值只能是 Object 类型，且无法向其插入值，这都是为了避免发生 ClassCastException

Java 的**类型通配符**对应 Kotlin 中的概念就是**星号投影 * **，Java 存在的限制在 Kotlin 中一样有

```kotlin
fun printList(list: List<*>) {
    for (any in list) {
        println(any)
    }
}
```

此外，星号投影只能出现在**类型形参**的位置，不能作为**类型实参**

```kotlin
val list: MutableList<*> = ArrayList<Number>() //正常

val list2: MutableList<*> = ArrayList<*>() //报错
```

# 八、协变 & 不变

看以下例子。Apple 和 Banana 都是 Fruit 的子类，可以发现 Apple[] 类型的对象是可以赋值给 Fruit[] 的，且 Fruit[] 可以容纳 Apple 对象和 Banana 对象，这种设计就被称为**协变**，即如果 A 是 B 的子类，那么 A[] 就是 B[] 的子类型。相对的，Object[] 就是所有数组对象的父类型

```java
static class Fruit {}

static class Apple extends Fruit {}

static class Banana extends Fruit {}

public static void main(String[] args) {
    Fruit[] fruitArray = new Apple[10];
    // 正常
    fruitArray[0] = new Apple();
    // 编译时正常，运行时抛出 ArrayStoreException
    fruitArray[1] = new Banana();
}
```

而 Java 中的泛型是**不变**的，这意味着 String 虽然是 Object 的子类，但 `List<String>` 并不是 `List<Object>` 的子类型，两者并不具备继承关系

```java
List<String> stringList = new ArrayList<>();
List<Object> objectList = stringList; // 报错
```

那为什么 Java 中的泛型是**不变**的呢？

这可以通过看一个例子来解释。假设 Java 中的泛型是**协变**的，那么以下代码就可以成功通过编译阶段的检查，在运行时就不可避免地将抛出 ClassCastException，而引入泛型的初衷就是为了实现类型安全，支持协变的话那泛型也就没有比数组安全多少了，因此就将泛型被设计为**不变**的

```java
List<String> strList = new ArrayList<>();
List<Object> objs = strList; //假设可以运行，实际上编译器会报错
objs.add(1);
String str = strList.get(0); //将抛出 ClassCastException，无法将整数转换为字符串
```

再来想个问题，既然**协变**本身并不安全，那么数组为何又要被设计为协变呢？

Arrays 类包含一个 `equals` 方法用于比较两个数组对象是否相等。如果数组是协变的，那么就需要为每一种数组对象都定义一个 `equals` 方法，包括开发者自定义的数据类型。想要避免这种情况，就需要让 Object[] 可以接收任意数组类型，即**让 Object[] 成为所有数组对象的父类型**，这就使得数组必须支持协变，这样多态才能生效

```java
public class Arrays {

    public static boolean equals(Object[] a, Object[] a2) {
        if (a == a2) return true;
        if (a == null || a2 == null) return false;

        int length = a.length;
        if (a2.length != length) return false;

        for (int i = 0; i < length; i++) {
            Object o1 = a[i];
            Object o2 = a2[i];
            if (!(o1 == null ? o2 == null : o1.equals(o2))) return false;
        }

        return true;
    }
}
```

需要注意的是，Kotlin 中的数组和 Java 中的数组并不一样，Kotlin 数组并不支持协变，Kotlin 数组类似于集合框架，具有对应的实现类 Array，Array 属于泛型类，支持了泛型因此也不再协变

```kotlin
val stringArray = arrayOfNulls<String>(3)
val anyArray: Array<Any?> = stringArray //报错
```

> Java 的泛型也并非完全**不变**的，只是实现**协变**需要满足一些条件，甚至也可以实现**逆变**，下面就来介绍下泛型如何实现**协变**和**逆变**

# 九、泛型协变

假设我们定义了一个 `copyAll` 希望用于 List 数据迁移。那以下操作在我们看来就是完全安全的，因为 Integer 是 Number 的子类，按道理来说是能够将 Integer 保存为 Number 的，但由于泛型不变性，`List<Integer>` 并不是 `List<Number>` 的子类型，所以实际上该操作将报错

```java
public static void main(String[] args) {
    List<Number> numberList = new ArrayList<>();

    List<Integer> integerList = new ArrayList<>();
    integerList.add(1);
    integerList.add(2);
    integerList.add(3);

    copyAll(numberList, integerList); // 报错
}

private static <T> void copyAll(List<T> to, List<T> from) {
    to.addAll(from);
}
```

思考下该操作为什么会报错？

编译器的作用之一就是**进行安全检查并阻止可能发生不安全行为的操作**，`copyAll` 方法会报错，那么肯定就是编译器觉得该方法有可能会触发不安全的操作。开发者的本意是希望将 Integer 类型的数据转移到 NumberList 中，只有这种操作且这种操作在我们看来肯定是安全的，但是编译器不知道开发者最终所要做的具体操作啊

假设 `copyAll` 方法可以正常调用，那么 `copyAll` 方法自然只会把 `from` 当做 `List<Number>` 来看待。因为 Integer 是 Number 的子类，从 `integerList` 获取到的数据对于 `numberList` 来说自然是安全的。而如果我们在 `copyAll` 方法中偷偷向 `integerList` 传入了一个 Number 类型的值的话，那么自然就将抛出异常，因为 from 实际上是 `List<Integer>` 类型

为了阻止这种不安全的行为，编译器选择通过直接报错来进行提示。为了解决报错，我们就需要向编译器做出安全保证：**从 from 取出来的值只会当做 Number 类型，且不会向 from 传入任何值**

为了达成以上保证，需要修改下 `copyAll` 方法

```java
private static <T> void copyAll(List<T> to, List<? extends T> from) {
    to.addAll(from);
}
```

`? extends T` 表示 `from` 接受 T 或者 T 的子类型，而不单单是 T 自身，这意味着我们可以安全地从 `from` 中取值并声明为 T 类型，但由于我们并不知道 T 代表的具体类型，写入操作并不安全，因此编译器会阻止我们向 `from` 执行传值操作。有了该限制后，从 `integerList` 中取出来的值只能是当做 Number 类型，且避免了向 `integerList` 插入非法值的可能，此时 `List<Integer>` 就相当于 `List<? extends Number>` 的子类型了，从而使得 `copyAll` 方法可以正常使用

简而言之，带 **extends** 限定了上界的通配符类型使得**泛型参数类型是协变的**，即如果 A 是 B 的子类，那么 `Generic<A>` 就是 `Generic<? extends B>` 的子类型

# 十、泛型逆变

**协变**所能做到的是：如果 A 是 B 的子类，那么 `Generic<A>` 就是 `Generic<? extends B>` 的子类型。**逆变**相反，其代表的是：如果 A 是 B 的子类，那么 `Generic<B>` 就是 `Generic<? super A>` 的子类型

协变还比较好理解，毕竟其继承关系是相同的，但逆变就比较反直觉了，整个继承关系都倒过来了

逆变的作用可以通过相同的例子来理解，`copyAll` 方法如下修改也可以正常使用，此时就是向编译器做出了另一种安全保证：**向 numberList 传递的值只会是 Integer 类型，且从 numberList 取出的值也只会当做 Object 类型**

```java
private static <T> void copyAll(List<? super T> to, List<T> from) {
    to.addAll(from);
}
```

`? super T` 表示 `to` 接收 T 或者 T 的父类型，而不单单是 T 自身，这意味着我们可以安全地向 `to` 传类型为 T 的值，但由于我们并不知道 T 代表的具体类型，所以从 `to` 取出来的值只能是 Object 类型。有了该限制后，`integerList` 只能向 `numberList` 传递类型为 Integer 的值，且避免了从 `numberList` 中获取到非法类型值的可能，此时 `List<Number>` 就相当于 `List<? super Integer>` 的子类型了，从而使得 `copyAll` 方法可以正常使用

简而言之，带 **super** 限定了下界的通配符类型使得**泛型参数类型是逆变的**，即如果 A 是 B 的子类，那么 `Generic<B>` 就是 `Generic<? super A>` 的子类型

# 十一、out & in

Java 中关于泛型的困境在 Kotlin 中一样存在，out 和 in 都是 Kotlin 的关键字，其作用都是为了来应对泛型问题。`in` 和 `out` 是一个对立面，同时它们又与泛型**不变**相对立，统称为**型变**

- out 本身带有**出去**的意思，本身带有倾向于**取值操作**的意思，用于**泛型协变**
- in 本身带有**进来**的意思，本身带有倾向于**传值操作**的意思，用于**泛型逆变**

再来看下相同例子，该例子在 Java 中存在的问题在 Kotlin 中一样有

```kotlin
fun main() {
    val numberList = mutableListOf<Number>()

    val intList = mutableListOf(1, 2, 3, 4)

    copyAll(numberList, intList) //报错

    numberList.forEach {
        println(it)
    }
}

fun <T> copyAll(to: MutableList<T>, from: MutableList<T>) {
    to.addAll(from)
}
```

报错原因和 Java 完全一样，因为此时编译器无法判断出我们到底是否会做出不安全的操作，所以我们依然要来向编译器做出安全保证

此时就需要在 Kotlin 中来实现**泛型协变**和**泛型逆变**了，以下两种方式都可以实现：

```kotlin
fun <T> copyAll(to: MutableList<T>, from: MutableList<out T>) {
    to.addAll(from)
}

fun <T> copyAll(to: MutableList<in T>, from: MutableList<T>) {
    to.addAll(from)
}
```

`out` 关键字就相当于 Java 中的 `<?  extends T>`，其作用就是限制了 `from` 不能用于接收值而只能向其取值，这样就避免了从 `to` 取出值然后向 `from` 赋值这种不安全的行为了，即实现了泛型协变

`in` 关键字就相当于 Java 中的 `<? super T>`，其作用就是限制了 `to` 只能用于接收值而不能向其取值，这样就避免了从 `to` 取出值然后向 `from` 赋值这种不安全的行为了，即实现了泛型逆变

> 从这也可以联想到，`MutableList<*>` 就相当于 `MutableList<out Any?>` 了，两者都带有相同的限制条件：不允许写值操作，允许读值操作，且读取出来的值只能当做 `Any?` 进行处理

# 十二、支持协变的 List

在上述例子中，想要实现协变还有另外一种方式，那就是使用 List

将 from 的类型声明从 `MutableList<T>` 修改为 `List<T>` 后，可以发现 `copyAll` 方法也可以正常调用了

```kotlin
fun <T> copyAll(to: MutableList<T>, from: List<T>) {
    to.addAll(from)
}
```

对 Kotlin 有一定了解的同学应该知道，Kotlin 中的集合框架分为两种大类：**可读可写**和**只能读不能写**

以 Java 中的 ArrayList 为例，Kotlin 将之分为了 MutableList 和 List 两种类型的接口。而 List 接口中的泛型已经使用 out 关键字进行修饰了，且不包含任何**传入值并保存**的方法，即 List 接口**只支持读值而不支持写值**，其本身就已经满足了协变所需要的条件，因此 `copyAll` 方法可以正常使用

```kotlin
public interface List<out E> : Collection<E> {
    override val size: Int
    override fun isEmpty(): Boolean
    override fun contains(element: @UnsafeVariance E): Boolean
    override fun iterator(): Iterator<E>
    override fun containsAll(elements: Collection<@UnsafeVariance E>): Boolean
    public operator fun get(index: Int): E
    public fun indexOf(element: @UnsafeVariance E): Int
    public fun lastIndexOf(element: @UnsafeVariance E): Int
    public fun listIterator(): ListIterator<E>
    public fun listIterator(index: Int): ListIterator<E>
    public fun subList(fromIndex: Int, toIndex: Int): List<E>
}
```

> 虽然 List 接口中有几个方法也接收了 E 类型的入参参数，但该方法本身不会进行写值操作，所以实际上可以正常使用，Kotlin 也使用 `@UnsafeVariance` 抑制了编译器警告

# 十三、reified & inline

上文讲了，由于类型擦除，Java 和 Kotlin 的泛型类型实参都会在编译阶段被擦除，在 Kotlin 中存在一个额外手段可以来解决这个问题，即**内联函数**

用关键字 inline 标记的函数就称为内联函数，再用 reified 关键字修饰内联函数中的泛型形参，编译器在进行编译的时候便会将内联函数的字节码插入到每一个调用的地方，当中就包括泛型的类型实参。而内联函数的类型形参能够被实化，就意味着我们可以在运行时引用实际的类型实参了

例如，我们可以写出以下这样的一个内联函数，用于判断一个对象是否是指定类型

```kotlin
fun main() {
    println(1.isInstanceOf<String>())
    println("string".isInstanceOf<Int>())
}

inline fun <reified T> Any.isInstanceOf(): Boolean {
    return this is T
}
```

将以上的 Kotlin 代码反编译为 Java 代码，可以看出来 `main()` 方法最终是没有调用 `isInstanceOf` 方法的，具体的判断逻辑都被插入到了 `main()` 方法内部，最终是执行了 `instanceof` 操作，且指定了具体的泛型类型参数 String 和 Integer

```java
public final class GenericTest6Kt {
    public static final void main() {
        Object $this$isInstanceOf$iv = 1;
        int $i$f$isInstanceOf = false;
        boolean var2 = $this$isInstanceOf$iv instanceof String;
        $i$f$isInstanceOf = false;
        System.out.println(var2);
        Object $this$isInstanceOf$iv = "string";
        $i$f$isInstanceOf = false;
        var2 = $this$isInstanceOf$iv instanceof Integer;
        $i$f$isInstanceOf = false;
        System.out.println(var2);
    }

    // $FF: synthetic method
    public static void main(String[] var0) {
        main();
    }

    // $FF: synthetic method
    public static final boolean isInstanceOf(Object $this$isInstanceOf) {
        int $i$f$isInstanceOf = 0;
        Intrinsics.checkNotNullParameter($this$isInstanceOf, "$this$isInstanceOf");
        Intrinsics.reifiedOperationMarker(3, "T");
        return $this$isInstanceOf instanceof Object;
    }
}
```

inline 和 reified 比较有用的一个场景是用在 Gson 反序列的时候。由于泛型运行时**类型擦除**的问题，目前用 Gson 反序列化泛型类时步骤是比较繁琐的，利用 inline 和 reified 我们就可以简化很多操作

```kotlin
val gson = Gson()

inline fun <reified T> toBean(json: String): T {
    return gson.fromJson(json, T::class.java)
}

data class BlogBean(val name: String, val url: String)

fun main() {
    val json = """{"name":"业志陈","url":"https://juejin.cn/user/923245496518439"}"""
    val listJson = """[{"name":"业志陈","url":"https://juejin.cn/user/923245496518439"},{"name":"业志陈","url":"https://juejin.cn/user/923245496518439"}]"""

    val blogBean = toBean<BlogBean>(json)
    val blogMap = toBean<Map<String, String>>(json)
    val blogBeanList = toBean<List<BlogBean>>(listJson)
    
    //BlogBean(name=业志陈, url=https://juejin.cn/user/923245496518439)
    println(blogBean)
    //{name=业志陈, url=https://juejin.cn/user/923245496518439}
    println(blogMap)
    //[{name=业志陈, url=https://juejin.cn/user/923245496518439}, {name=业志陈, url=https://juejin.cn/user/923245496518439}]
    println(blogBeanList)
}
```

我也利用 Kotlin 的这个强大特性写了一个用于简化 Java / Kotlin 平台的序列化和反序列化操作的库：[JsonHolder](https://github.com/leavesczy/JsonHolder)

# 十四、总结

最后来做个简单的总结

| | 协变 | 逆变 | 不变 |
| ------ | -------------------------------------------------- | ------------------------------------------------------------ | --------------------------- |
| Kotlin | `<out T>`，只能作为消费者，只能读取不能添加 | `<in T>`，只能作为生产者，只能添加，读取出的值只能当做 Any 类型 | `<T>`，既可以添加也可以读取 |
| Java | `<?  extends T>`，只能作为消费者，只能读取不能添加 | `<? super T>`，只能作为生产者，只能添加，读取出的值只能当做 Object 类型 | `<T>`，既可以添加也可以读取 |