---
layout: post
title: "Android官方编程建议"
date: 2013-08-27 21:28
comments: true
categories: Android

---
（原文<http://developer.android.com/training/articles/perf-tips.html>）


这篇文档主要介绍一些微优化，将这些优化技巧组合起来将使你的应用性能整体得到提升，当然这不会造成非常戏剧化的性能提升。本文中所提到的技巧可以作为小技巧帮助你养成良好和高效的编码习惯。


编写高效的代码有两条基本原则：

* 不要做你不需要做的工作
* 如果可以避免，就不要申请内存

当我们进行微优化的时候面对的最棘手的问题是你的应用需要运行在各种不同类型的硬件上，不同版本的VM运行在不同的处理器上的时候会有不同的运行速度，因此不能武断的说“设备X比设备Y运行速度快F”。模拟器只能告诉你非常有限的性能信息，同时支持JIT的设备上最适合运行的代码和不支持JIT的设备所适合运行的代码是不一样的。

为了保证你的应用能在很多设备上表现良好，你必须保证你的代码在每个层面上都是最优的.

###避免生成不需要的对象
生成对象永远不是免费的。当你在你的应用中申请了太多的对象的时候，你就会强制系统执行定期的垃圾收集，这会对用户体验造成不好的影响（卡顿）。Android 2.3开始提出的并发垃圾收集器可以帮忙减轻这样的问题，但是这些不需要的工作还是应该要避免。下面是一些例子：

* 如果你有一个方法返回一个String，然后你知道这个String会被连接到其他StringBuffer上，那就直接将这个方法连接过去，而不是创建一个临时的变量用来转接。
* 当你需要从一串输入值中返回一个字符串的时候，直接返回原始数据的subString而不是建立一个新的对象。这样做虽然会创建一个新的String对象，但是它会共享原有数据的char[]数据。（结果就是如果你只使用原始输入数据的一小部分内容，这部分内容会始终留在内存中）
* int数组比Integer数组要好很多，两个并行的int数组比(int,int)对象组成的数组好，其他的原始类型组合也有相同的特性。
* 如果你需要实现一个存放(Foo,Bar)对象的容器，记住Foo[]和Bar[]比自定义一个(Foo,Bar)对象性能好。

总体来说，尽可能避免创建临时对象。创建的对象越少，垃圾回收的频率会越低，这会直接影响用户体验。

###尽可能的使用静态方法
如果你不需要访问一个对象的变量，那就把对象里面的方法设计为静态的，调用静态方法会提升15%-20%的速度。这同样是一个很好的实践，因为把方法设计成静态就等于明确告诉调用者对方法的调用不会改变对象的状态。

###使用Static Final来定义常量
简单分析下下面的两个定义

```
static int intVal = 42;
static String strVal = "hello,world";
```

编译器会生成一个类的初始化方法，名为<clinit>，当这个类第一次被使用的时候会执行这个方法。这个方法会把42存到intVal里面，然后会从类文件的String常量表里面为strVal提取出一个引用。当之后这些值被引用的时候，他们通过值查找来找到。

我们可以使用final关键字来优化这一过程

```
static final int intVal = 42;
static final String strVal = "hello,world";
```

这样做之后，类文件将不再需要<clinit>方法，这些常量将由DEX文件中的静态变量初始化负责。代码调用intVal的时候会直接使用42，调用strVal的时候会使用一个消耗比较小的“String常量”指令替代非final时使用的值查找。

###避免使用内部的Getters/Setters

在一般的面向对象语言中，Getter和Setter是很好的编程习惯，它可以帮助你做一些访问控制或者在访问时做一些其他的操作。

但是在Android中，这并不是一个很好的习惯，调用虚拟方法的成本永远比直接访问变量来的高。当你在编写公用的接口的时候，按照规范编写getter和setter是合理的，但是在你的类内部，你应该每次都直接访问变量。

在没有JIT的时候，直接访问变量比访问getter快三倍，当有JIT的时候，直接访问变量的速度会快七倍。

###使用强化的for循环语句
```
static class Foo {
    int mSplat;
}

Foo[] mArray = ...

public void zero() {
    int sum = 0;
    for (int i = 0; i < mArray.length; ++i) {
        sum += mArray[i].mSplat;
    }
}

public void one() {
    int sum = 0;
    Foo[] localArray = mArray;
    int len = localArray.length;

    for (int i = 0; i < len; ++i) {
        sum += localArray[i].mSplat;
    }
}

public void two() {
    int sum = 0;
    for (Foo a : mArray) {
        sum += a.mSplat;
    }
}
```

上面三个方法中，方法zero()是最慢的，因为每次循环都回去计算一次length，这一步无法被JIT或者其他编译器优化。

方法one()稍微快一些，将length存放在本地变量中，不需要每次都进行运算。

方法two()在没有使用JIT的设备上速度最快，但是在使用JIT的设备上和one()差不多，它使用了JAVA 1.5版本开始有的强化的for循环。

因此总体来说你应该使用强化的for循环。但是当你需要遍历一个ArrayList的时候，你应该选择用普通的for循环，这会为你带来三倍的速度提升。（原文没说原因，接下来研究一下）

###使用包替代对私有类的私有访问

```
public class Foo {
    private class Inner {
        void stuff() {
            Foo.this.doStuff(Foo.this.mValue);
        }
    }

    private int mValue;

    public void run() {
        Inner in = new Inner();
        mValue = 27;
        in.stuff();
    }

    private void doStuff(int value) {
        System.out.println("Value is " + value);
    }
}
```

在上面的例子里，我们定义了一个私有的内部类(Foo$Inner)，这个类直接访问外部类的私有方法和私有变量。这样做是合法的，代码能正常打印出"Value is 27"。

这里存在的的问题是：尽管JAVA语言允许内部类访问外部类的私有方法，但是虚拟机认为从Foo$Inner里面直接访问Foo的私有方法是非法的，因为Foo和Foo$Inner是不同的类。为了实现这样的功能，编译器会生成一组合成方法：

```
/*package*/ static int Foo.access$100(Foo foo) {
    return foo.mValue;
}
/*package*/ static void Foo.access$200(Foo foo, int value) {
    foo.doStuff(value);
}
```

每当你需要调用私有类中的方法的时候，系统会自动调用生成的getter方法来实现调用。前面已经提过，通过setter和getter方法来访问变量是非常低效的，因此这样做在无形中增加了系统的消耗，降低了性能。

如果你必须在对性能要求很高的点使用类似的代码，建议将需要访问的变量定义为包级别的访问权限，这样同时会造成同一个包里面的代码都可以访问这些变量，因此不应该在公开的api中使用这样的访问权限。


###避免使用浮点数

这里有一条经验法则：在Android设备上，浮点数比整数的运算效率低两倍。

在速度层面上，float和double效果差不多，double会比float大一倍。

就算是使用整数，乘法比除法和余除效率高很多，除法和余除是通过软件来实现而不是硬件加速实现的。

###了解和使用系统库

在Android中，很多系统库都做了专门的调优，因此使用系统方法基本上都会比自己实现方法来的高效。很多JAVA的方法都已经由Dalvik做过专门的优化了，比如String.indexOf()和类似的API，类似的System.arraycopy()方法比手写的循环快9倍。

###谨慎使用本地方法

通过Android NDK编写本地代码开发应用并不会比使用JAVA编程来的高效。一个主要的原因是JAVA做本地代码转换的时候需要消耗一部分性能，JIT无法优化这一部分功能。当你使用本地代码来实现某些功能的时候，你可能需要编译很多个不同的拷贝来适配不同的设备。

本地代码最主要的用途是将你已有的代码编译到Android中来节省开发时间，本地代码并不能加速你应用中那些本来用JAVA实现的代码。

###性能的秘密

当设备上不支持JIT的时候，直接通过变量调用方法比通过接口调用方法要更高效。（打个比方调用HashMap map比调用Map map更高效，尽管两种情况下map都是一个HashMap）速度的差别在6%左右，如果设备上有JIT，这个差别将忽略不计。

在没有JIT的设备上，缓存一个变量用来访问比每次都直接访问一个变量要快20%。

###总是进行测量

在你开始优化之前，你需要确信你知道你需要解决的问题是什么。确保你能准确的测量你现有的性能，否则你将无法知道你的努力给你带来了多大的成效。

本文中所有的申明都有相关的标准，这些标准可以在<https://code.google.com/p/dalvik/source/browse/#svn/trunk/benchmarks>找到。

这些数据是通过[Caliper](https://code.google.com/p/caliper/)框架来实现的。强烈推荐使用Caliper来实现你自己的检测数据。
