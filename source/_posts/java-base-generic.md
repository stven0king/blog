---
title: JAVA回忆录之泛型篇
date: 2017-8-18 18:23:00
tags: [泛型]
categories: Java 
description: "泛型是JDK1.5版本中加入的，在没有泛型之前，从集合中读取到的每一个对象都必须进行转化。如果有人不小心插入了类型错误的对象，在运行时的转化处理就会出错。有了泛型之后，可以告诉变一起每个集合中接受那些对象类型。编译器自动地为你的插入进行转化，并在编译时告知是否插入了类型错误的对象。。。"
---
<center><img src="http://upload-images.jianshu.io/upload_images/1319879-2a8ba439a173436d.JPG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" width="600px" />

泛型是什么
===
泛型是JDK1.5版本中加入的，在没有泛型之前，从集合中读取到的每一个对象都必须进行转化。如果有人不小心插入了类型错误的对象，在运行时的转化处理就会出错。有了泛型之后，可以告诉变一起每个集合中接受那些对象类型。编译器自动地为你的插入进行转化，并在编译时告知是否插入了类型错误的对象。

泛型最精准的定义：参数化类型。具体点说就是处理的数据类型不是固定的，而是可以作为参数传入。定义泛型类、泛型接口、泛型方法，这样，同一套代码，可以用于多种数据类型。

>- K ——键，比如映射的键。
>- V ——值，比如 List 和 Set 的内容，或者 Map中的值。
>- E ——异常类。
>- T ——泛型。

数组是协变的（协变：其实只是表示如果Stub为Super的子类型，那么类型Stub[]就是Super[]的子类型），泛型不是协变的。因此数组和泛型不能好好地混合使用。


泛型类、接口和泛型方法
===
泛型类、接口
---
```java
public interface Iterable<T> {
    Iterator<T> iterator();
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }
    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}
```
一般而言，声明泛型接口的方式与声明泛型类相同。

泛型方法
---
```java
public class Collections {
    /***其他代码省略***/
    public static <T> void sort(List<T> list, Comparator<? super T> c) {
        list.sort(c);
    }
    /***其他代码省略***/
}
```

泛型擦除
===
```java
public class Main {
    public static void main(String[] args) {
        Generic<Integer> generic = new Generic<>();
        generic.set(new Integer(1));
        System.out.println(generic.get().intValue());
    }
    public static class Generic<T extends Number> {
        private T value;
        public T get() {
            return value;
        }
        public void set(T value) {
            this.value = value;
        }
    }
}
```
这是一个泛型使用的列子，我们反编译查看它的class文件：
```vim
Compiled from "Main.java"
public class com.loadclass.generic.Main$Generic<T extends java.lang.Number> {
  public com.loadclass.generic.Main$Generic();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public T get();
    Code:
       0: aload_0
       1: getfield      #2                  // Field value:Ljava/lang/Number;
       4: areturn

  public void set(T);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #2                  // Field value:Ljava/lang/Number;
       5: return
}

public class com.loadclass.generic.Main {
  public com.loadclass.generic.Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class com/loadclass/generic/Main$Generic
       3: dup
       4: invokespecial #3                  // Method com/loadclass/generic/Main$Generic."<init>":()V
       7: astore_1
       8: aload_1
       9: new           #4                  // class java/lang/Integer
      12: dup
      13: iconst_1
      14: invokespecial #5                  // Method java/lang/Integer."<init>":(I)V
      17: invokevirtual #6                  // Method com/loadclass/generic/Main$Generic.set:(Ljava/lang/Number;)V
      20: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
      23: aload_1
      24: invokevirtual #8                  // Method com/loadclass/generic/Main$Generic.get:()Ljava/lang/Number;
      27: checkcast     #4                  // class java/lang/Integer
      30: invokevirtual #9                  // Method java/lang/Integer.intValue:()I
      33: invokevirtual #10                 // Method java/io/PrintStream.println:(I)V
      36: return
}

```
从class文件中我们可以看出来，其实泛型在编译后都进行了擦除，类型T都转化为了它的超类Number，然后在需要使用的时候进行checkcast。当然在泛型没有做任何显示的时候比如Generic<T>，这样在编译生成的class文件中T都是转化为Object类型来处理的。

有界泛型类型<T extends superclass>
===
泛型参数类型可以使用任意参数类型替换。对于大多数情况这很好，但是限制能够传递给类型参数的类型是有时有用的。例如，假设希望创建一个泛型类，类中返回数据中数据平均值的方法（类型数字包括：整数、单精度和双精度）。
```java
public class Stats<T extends Number> {
    T[] nums;
    public Stats(T[] nums) {
        this.nums = nums;
    }
    double average() {
        double sum = 0.0;
        for (int i = 0; i < nums.length; i++)
            sum += nums[i].doubleValue();
        return sum / nums.length;
    }
}
```
向上边代码我们可以使用Number对T类型做限制，这样所有T类型对象都可以调用doubleValue()方法，因为该方法是由Number声明的。

泛型通配符参数
===
先看一段代码：
```java
class Stats<T extends Number> {
    T[] nums;
    public Stats(T[] nums) {
        this.nums = nums;
    }
    double average() {
        double sum = 0.0;
        for (int i = 0; i < nums.length; i++)
            sum += nums[i].doubleValue();
        return sum / nums.length;
    }
     boolean sameAge(Stats<T> ob) {
        if (average() == ob.average()) {
            return true;
        }
        return false;
    }
}
```
这样的实现导致只有sameAge方法的参数类型和地啊用对象的类型相同时才能工作。

为了创建smaeAvg方法，必须使用Java泛型的另一个特性：通配符参数。通配符参数是由“？”指定的，表示未知类型。
```java
class Stats<T extends Number> {
    T[] nums;
    public Stats(T[] nums) {
        this.nums = nums;
    }
    double average() {
        double sum = 0.0;
        for (int i = 0; i < nums.length; i++)
            sum += nums[i].doubleValue();
        return sum / nums.length;
    }
     boolean sameAge(Stats<？> ob) {
        if (average() == ob.average()) {
            return true;
        }
        return false;
    }
}
```
此时，Stats<?>和所有的Stats对象匹配，允许任意两个Stats对象比较它们的平均值。

在讲述有界通配符之前我们先看一段代码：
```java
Apple[] apples = new Apple[1];
Fruit[] fruits = apples;
fruits[0] = new Strawberry();
```
因为数组可以协变的，所以Apple的数据applse可以赋值给子类数组fruits，但是在给数组中的某个Fruit对象赋值的时候假如不是Apple或者其子类类型，那么代码可以编译，但在允许时会抛出java.lang.ArrayStoreException的异常。

有界通配符（上界）
---
向上造型一个泛型对象的引用<? extends superclass>

我们可以使用通配符把相关的代码转换程泛型：因为Apple是Fruit的一个子类，我们使用? extends 通配符，这样就能将一个List<Apple>对象的定义赋到一个List<? extends Fruit>的声明上：
```java
List<Apple> apples = new ArrayList<Apple>();
List<? extends Fruit> fruits = apples;
```
这次，代码就编译不过去了！Java编译器会阻止你往一个Fruit list里加入strawberry。在编译时我们就能检测到错误，在运行时就不需要进行检查来确保往列表里加入不兼容的类型了。即使你往list里加入Fruit对象也不行：
```java
fruits.add(new Fruit());
```
事实上你不能够往一个使用了? extends的数据结构里写入任何的值。

原因非常的简单，你可以这样想：这个? extends T 通配符告诉编译器我们在处理一个类型T的子类型，但我们不知道这个子类型究竟是什么。因为没法确定，为了保证类型安全，我们就不允许往里面加入任何这种类型的数据。另一方面，因为我们知道，不论它是什么类型，它总是类型T的子类型，当我们在读取数据时，能确保得到的数据是一个T类型的实例：
```java
Fruit get = fruits.get(0);
```

有界通配符（下界）
---
向下造型一个泛型对象的引用<? super superclass>

使用<? super superclass>通配符一般是什么情况？让我们先看看这个：
```java
List<Fruit> fruits = new ArrayList<Fruit>();
List<? super Apple> = fruits;
```
我们看到fruits指向的是一个装有Apple的某种超类(supertype)的List。同样的，我们不知道究竟是什么超类，但我们知道Apple和任何Apple的子类都跟它的类型兼容。既然这个未知的类型即是Apple，也是GreenApple的超类，我们就可以写入：
```java
fruits.add(new Apple());
fruits.add(new GreenApple());
```
如果我们想往里面加入Apple的超类，编译器就会警告你：
```java
fruits.add(new Fruit());
fruits.add(new Object());
```
因为我们不知道它是怎样的超类，所有这样的实例就不允许加入。

从这种形式的类型里获取数据又是怎么样的呢？结果表明，你只能取出Object实例：因为我们不知道超类究竟是什么，编译器唯一能保证的只是它是个Object，因为Object是任何Java类型的超类。

存取原则和PECS法则
===
请记住PECS法则：生产者(Producer)使用extends，消费者(Consumer)使用super。
- 生产者使用extends

> 如果你需要一个列表提供T类型的元素（即你想从列表中读取T类型的元素），你需要把这个列表声明成<? extends T>，比如List<? extends Integer>，因此你不能往改列表中添加任何元素。

- 消费者使用super

> 如果需要一个列表使用T类型的元素（即你想把T类型的元素加入到列表中），你需要把这个列表声明成<？ super T>，比如List<? super Integer>，因此你不能保证从中读取到的元素的类型。

- 即是生产者，也是消费者

> 如果一个列表既要生成，又要消费，你不能使用泛型通配符声明列表，比如List<Integer>。

泛型类的层次问题
===
泛型类可以是类层次的一部分，就像非泛型类那样，因此，泛型类可以作为超类或子类。泛型和非泛型层次之间的关键区别是：在泛型层次中，类层次中的所有子类都必须向上传递超类所需要的所有类型参数。这与必须沿着类层次向上构造函数的参数类似。

使用泛型超类
---
```java
class Gen<T> {
    private T obj;
    Gen(T object) {
        this.obj = object;
    }
    public T get() {
        return obj;
    }
}
```
Generic继承Gen:
```java
public class Generic<T> extends Gen<T> {
    Generic(T object) {
        super(object);
    }
}
```
继承泛型类的子类也可以拥有自己的泛型：
```java
public class Generic<V,T> extends Gen<T> {
    private V value;
    Generic(T object, V value) {
        super(object);
        this.value = value;
    }
    public V getValue() {
        return value;
    }
}
```
使用泛型子类
---
```java
class NoGen{
    int num;
    NoGen(int num) {
        this.num = num;
    }
    int getNum(){
        return num;
    }
}
```
子类继承NoGen并实现泛型：
```java
class Gen<T> extends NoGen {
    private T obj;
    Gen(T object, int value) {
        super(value);
        this.obj = object;
    }
    public T get() {
        return obj;
    }
}
```
强制转化
---
只有当两个泛型类型实例的类型相互兼容并且他们的类型参数也是相同时，才能将其中的一个实例转化为另一个实例：
```java
List<Integer> listInteger = new ArrayList<Integer>();//legal
List<Integer> listLong = new ArrayList<Long>();//illegal
```
重写泛型类的方法
---
可以像重写其他任何方法那样重写泛型类的方法：
```java
class Gen<T>{
    T obj;
    Gen(T object) {
        this.obj = object;
    }
    public T get() {
        return obj;
    }
}
public class Generic<T> extends Gen<T> {
    Generic(T object) {
        super(object);
    }
    public T get() {
        obj = null;
        return obj;
    }
}
```
泛型类型推断
---
```java
List<Integer> list = new ArrayList<Integer>();//<1.7
List<Integer> integerList = new ArrayList<>();//1.7
```
在JDK1.6的时候我们声明泛型和new一个泛型实例时必须制定相同的类型。而从
JDK1.7开始new的泛型实例不用制定类型，编译期会默认与声明的对象用于相同的泛型类型。

擦除
---
前文中讲过泛型的擦除，为什么这里还需要再讲述呢？这里讲述在继承泛型时的擦除，仔细阅读会有不一样的发现哦~！
```java
class Gen<T>{
    T obj;
    Gen(T object) {
        this.obj = object;
    }
    public T get() {
        return obj;
    }
}
public class Generic extends Gen<String> {
    Generic(String object) {
        super(object);
    }
    public String get() {
        return obj;
    }
}
```
我们分析编译后生成的Generic的class文件：
```vim
Compiled from "Generic.java"
public class com.genericity.Generic extends com.genericity.Gen<java.lang.String> {
  com.genericity.Generic(java.lang.String);
    Code:
       0: aload_0
       1: aload_1
       2: invokespecial #1                  // Method com/genericity/Gen."<init>":(Ljava/lang/Object;)V
       5: return

  public java.lang.String get();
    Code:
       0: aload_0
       1: getfield      #2                  // Field obj:Ljava/lang/Object;
       4: checkcast     #3                  // class java/lang/String
       7: areturn

  public java.lang.Object get();
    Code:
       0: aload_0
       1: invokevirtual #4                  // Method get:()Ljava/lang/String;
       4: areturn
}
```
我们可以看出Generic中有两个get方法，这是编译期偶尔需要为类添加桥接方法。
> 桥接方法
> 子类中重写方法的类型擦除不能产生于超类中方法相同的擦除。对于这种情况，会生成使用超类类型擦除的方法，并且这个方法调用具有由子类指定的类型擦除的方法。当然桥接方法只会在字节码级别发生。

模糊性错误
---
泛型的引入，增加了引起一种新类型错误——模糊性错误的可能，必须注意防范。当擦除导致两个看起来不同的泛型声明，在擦除后变成相同的类型而导致冲突时，就会发生模糊性错误。
```java
class Gen<T>{
    private T obj;
    public void set(T object) {
        this.obj = object;
    }
    public T get() {
        return obj;
    }
}
public class Generic<V, T> extends Gen<T> {
    V value;
    public void set(V value) {//illegal
        this.value = value;
    }
}
```
这样进行重载防范的时候，如果进行泛型擦除那么两个方法一模一样。像这样的情况使用两个独立的方法名会更好一些，而不是试图重载set方法。

泛型在使用过程中应该注意的问题
===
不能用基本类型实例化类型参数
---
```java
Map<int, int> pair = new HashMap<int, int>();
```
这样的语句是非法的。

不能抛出也不能捕获泛型类实例
---
泛型类扩展Throwable即为不合法，因此无法抛出或捕获泛型类实例。但在异常声明中使用类型参数是合法的：
```java
public static <T extends Throwable> void doWork(T t) throws T {
  try {
    ...
  } catch (Throwable realCause) {
    t.initCause(realCause);
    throw t;
  }
}
```

数据不能结合泛型使用
---
在Java中数据是协变的，Object[]数组可以是任何数组的父类（因为任何一个数组都可以向上转型为它在定义时指定元素类型的父类的数组）。考虑以下代码：
```java
String[] strs = new String[10];
Object[] objs = strs;
objs[0] = new Long(1);
```
在上述代码中，我们将数组元素赋值为满足父类（Object）类型，但不同于原始类型Long的对象，在编译时能够通过，而在运行时会抛出ArrayStoreException异常。

不能实例化类型变量
---
不能以诸如“new T(...)", "new T[...]", "T.class"的形式使用类型变量。Java禁止我们这样做的原因很简单，编译期不知道创建那种类型的对象。T只是一个占位符。

对静态成员的一些限制
---
注意，这里我们强调了泛型类。因为普通类中可以定义静态泛型方法，如上面我们提到的ArrayAlg类中的getMiddle方法。关于为什么有这样的规定，请考虑下面的代码：
```java
public class People<T> { 
  public static T name; 
  public static T getName() { 
    ... 
  }
}
```
我们知道，在同一时刻，内存中可能存在不只一个People<T>类实例。假设现在内存中存在着一个People<String>对象和People<Integer>对象，而类的静态变量与静态方法是所有类实例共享的。那么问题来了，name究竟是String类型还是Integer类型呢？基于这个原因，Java中不允许在泛型类的静态上下文中使用类型变量。

泛型在我们编码的过程中，特别是写一些框架或者通用组件时是非常有帮助的。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)