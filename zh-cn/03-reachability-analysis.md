# 可达性分析算法

![访问统计](https://visitor-badge.glitch.me/badge?page_id=senlypan.jvm.03-reachability-analysis&left_color=blue&right_color=red)

> 作者: 潘深练
>
> 更新: 2022-03-08

## 什么是可达性分析算法

在主流的商用程序语言如`Java`、`C#`等的主流实现中，都是通过可达性分析(`Reachability Analysis`)来判断对象是否存活的。此算法的基本思路就是通过一系列的“`GC Roots`”的对象作为起始点，从起始点开始向下搜索到对象的路径。搜索所经过的路径称为引用链(`Reference Chain`)，当一个对象到任何`GC Roots`都没有引用链时，则表明对象“`不可达`”，即该对象是`不可用`的。

![03-reachability-analysis-001](../_media/image/03-reachability-analysis/03-reachability-analysis-001.png)

**在Java语言中，可作为GC Roots的对象包括下面几种：**

- 栈帧中的局部变量表中的`reference`引用所引用的对象
- 方法区中`static`静态引用的对象
- 方法区中`final`常量引用的对象
- 本地方法栈中`JNI`(`Native`方法)引用的对象
- `Java`虚拟机内部的引用， 如基本数据类型对应的`Class`对象， 一些常驻的异常对象（比如 `NullPointExcepiton`、`OutOfMemoryError`） 等， 还有系统类加载器。
- 所有被同步锁（`synchronized`关键字） 持有的对象。
- 反映`Java`虚拟机内部情况的`JMXBean`、 `JVMTI`中注册的回调、 本地代码缓存等。

![03-reachability-analysis-002](../_media/image/03-reachability-analysis/03-reachability-analysis-002.png)

## JVM判断对象是否存活

**`finalize()`方法最终判定对象是否存活**

即使在可达性分析算法中判定为`不可达`的对象， 也不是“`非死不可`”的， 这时候它们暂时还处于“`缓刑`”阶段，要真正宣告一个对象死亡， 至少要经历两次标记过程：

### 第一次标记

如果对象在进行可达性分析后发现没有与`GC Roots`相连接的引用链， 那它将会被第一次标记， 随后进行一次筛选， 筛选的条件是此对象是否有必要执行`finalize()`方法。

### 没有必要

假如对象没有覆盖`finalize()`方法， 或者`finalize()`方法已经被虚拟机调用过， 那么虚拟机将这两种情况都视为“`没有必要执行`”。

### 有必要

如果这个对象被判定为确有必要执行`finalize()`方法， 那么该对象将会被放置在一个名为`F-Queue`的 队列之中，并在稍后由一条由虚拟机自动建立的、低调度优先级的Finalizer线程去执行它们的`finalize()` 方法。 `finalize()`方法是对象逃脱死亡命运的最后一次机会，稍后收集器将对`F-Queue`中的对象进行第二次小规模的标记，如果对 象要在`finalize()`中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己（`this`关键字）赋值给某个类变量或者对象的成员变量， 那在第二次标记时它将被移出“即将回收”的集 合； 如果对象这时候还没有逃脱，那基本上它就真的要被回收了。

![03-reachability-analysis-003](../_media/image/03-reachability-analysis/03-reachability-analysis-003.png)

一次对象自我拯救的演示

```java
/**
* 此代码演示了两点：
* 1.对象可以在被GC时自我拯救。
* 2.这种自救的机会只有一次， 因为一个对象的finalize()方法最多只会被系统自动调用一次
*/
public class FinalizeEscapeGC {
  public static FinalizeEscapeGC SAVE_HOOK = null;
  public void isAlive() {
    System.out.println("yes, i am still alive :)");
 }
  @Override
  protected void finalize() throws Throwable {
    super.finalize();
    System.out.println("finalize method executed!");
    FinalizeEscapeGC.SAVE_HOOK = this;
 }
  public  static void main(String[] args) throws Throwable {
    SAVE_HOOK = new FinalizeEscapeGC();
    //对象第一次成功拯救自己
    SAVE_HOOK = null;
    System.gc();
    // 因为Finalizer方法优先级很低， 暂停0.5秒， 以等待它
    Thread.sleep(500);
    if (SAVE_HOOK != null) {
     SAVE_HOOK.isAlive();
   } else {
      System.out.println("no, i am dead :(");
   }
    //下面这段代码与上面的完全相同，但是这次自救却失败了
    SAVE_HOOK = null;
    System.gc();
    // 因为Finalizer方法优先级很低， 暂停0.5秒， 以等待它
    Thread.sleep(500);
    if (SAVE_HOOK != null) {
      SAVE_HOOK.isAlive();
   } else {
      System.out.println("no, i am dead :(");
   }
 }
}
```

> 注意

`Finalizer`线程去执行它们的`finalize()` 方法, 这里所说的“执行”是指虚拟机会触发这个方法开始运行， 但并不承诺一定会等待它运行结束。 这样做的原因是， 如果某个对象的`finalize()`方法执行缓慢， 或者更极端地发生了死循环， 将很可能导致`F-Queue`队列中的其他对象永久处于等待， 甚至导致整个内存回收子系统的崩溃。

## 再谈引用

在JDK1.2以前，Java中引用的定义很传统: 如果引用类型的数据中存储的数值代表的是另一块内存的起始地址，就称这块内存代表着一个引用。这种定义有些狭隘，一个对象在这种定义下只有被引用或者没有被引用两种状态。 我们希望能描述这一类对象: 当内存空间还足够时，则能保存在内存中；如果内存空间在进行垃圾回收后还是非常紧张，则可以抛弃这些对象。很多系统中的缓存对象都符合这样的场景。 在JDK1.2之后，Java对引用的概念做了扩充，将引用分为 `强引用(Strong Reference)` 、 `软引用(Soft Reference)` 、` 弱引用(Weak Reference)` 和 `虚引用(Phantom Reference)` 四种，这四种引用的强度依次递减。

> 强引用（StrongReference）

**new 出的对象之类的引用，只要强引用还在，永远不会回收。**

```java
Object strongReference = new Object();
```

强引用是使用最普遍的引用。如果一个对象具有强引用，那垃圾回收器绝不会回收它。当内存空间不足，Java虚拟机宁愿抛出`OutOfMemoryError`错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足的问题。 ps：强引用其实也就是我们平时`A a = new A()`这个意思。

> 软引用（SoftReference）

**引用但非必须的对象，内存溢出异常之前，回收。**

软引用可以和一个引用队列(ReferenceQueue)联合使用。如果软引用所引用对象被垃圾回收，JAVA虚拟机就会把这个软引用加入到与之关联的引用队列中。

```java

ReferenceQueue<String> referenceQueue = new ReferenceQueue<>();
String str = new String("abc");
SoftReference<String> softReference = new SoftReference<>(str, referenceQueue);

str = null;
// Notify GC
System.gc();

System.out.println(softReference.get()); // abc

Reference<? extends String> reference = referenceQueue.poll();
System.out.println(reference); //null

```

如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。 软引用可以和一个引用队列（`ReferenceQueue`）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。

> 弱引用（WeakReference）

**非必须的对象，对象能生存到下一次垃圾收集发生之前。**

`WeakReference`对象的生命周期基本由垃圾回收器决定，一旦垃圾回收线程发现了弱引用对象，在下一次GC过程中就会对其进行回收。

```java
String str = new String("abc");
WeakReference<String> weakReference = new WeakReference<>(str);
str = null;
```

用来描述那些非必须对象， 但是它的强度比软引用更弱一些， 被弱引用关联的对象只能生存到下一次垃圾收集发生为止。 当垃圾回收器开始工作， 无论当前内存是否足够， 都会回收掉只 被弱引用关联的对象。 在JDK 1.2版之后提供了`WeakReference`类来实现弱引用。 弱引用可以和一个引用队列（`ReferenceQueue`）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。

**弱引用与软引用的区别在于:**

1. 更短暂的生命周期;
2. 一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。

> 虚引用（PhantomReference）

**对生存时间无影响，在垃圾回收时得到通知。**

“`虚引用`”顾名思义，它是最弱的一种引用关系。如果一个对象仅持有虚引用，在任何时候都可能被垃圾回收器回收。虚引用主要用来跟踪对象被垃圾回收器回收的活动。

**虚引用主要用来跟踪对象被垃圾回收器回收的活动。 虚引用与软引用和弱引用的一个区别在于：**

1. 虚引用必须和引用队列 （`ReferenceQueue`）联合使用。
2. 当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。

```java
    String str = new String("abc");
    ReferenceQueue queue = new ReferenceQueue();
    // 创建虚引用，要求必须与一个引用队列关联
    PhantomReference pr = new PhantomReference(str, queue);
```

程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要进行垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

> Java中4种引用的级别和强度由高到低依次为：强引用 -> 软引用 -> 弱引用 -> 虚引用

当垃圾回收器回收时，某些对象会被回收，某些不会被回收。垃圾回收器会从根对象`Object`来标记存活的对象，然后将某些不可达的对象和一些引用的对象进行回收。

（本篇完）

?> ❤️ 您也可以参与梳理，快来提交 [issue](https://github.com/senlypan/jvm-docs/issues) 或投稿参与吧~