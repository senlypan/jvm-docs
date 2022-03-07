# 常见内存溢出异常

> 作者: 潘深练
>
> 更新: 2022-03-07

## Java堆溢出

堆内存中主要存放对象、数组等，只要不断地创建这些对象，并且保证 GC Roots 到对象之间有可达路径来避免垃圾收集回收机制清除这些对象，当这些对象所占空间超过最大堆容量时，就会产生 OutOfMemoryError 的异常。堆内存异常示例如下：

```java
/**
 * 设置最大堆最小堆：-Xms20m -Xmx20m
 */
public class HeapOOM {
    
    static class OOMObject {}
    
    public static void main(String[] args) {
        List<OOMObject> oomObjectList = new ArrayList<>();
        while (true) {
            oomObjectList.add(new OOMObject());
        }
    }
}
```

**运行后会报异常，在堆栈信息中可以看到**

`java.lang.OutOfMemoryError: Java heap space` 的信息，说明在堆内存空间产生内存溢出的异常。

新产生的对象最初分配在新生代，新生代满后会进行一次 `Minor GC `，如果 `Minor GC` 后空间不足会把该对象和
新生代满足条件的对象放入老年代，老年代空间不足时会进行 `Full GC` ，之后如果空间还不足以存放新对象则抛
出 `OutOfMemoryError` 异常。

**常见原因：**

- 内存中加载的数据过多，如一次从数据库中取出过多数据；
- 集合对对象引用过多且使用完后没有清空；
- 代码中存在死循环或循环产生过多重复对象；
- 堆内存分配不合理

## 虚拟机栈和本地方法栈溢出

由于HotSpot虚拟机中并不区分虚拟机栈和本地方法栈， 因此对于HotSpot来说， -Xoss参数（设置本地方法栈大小） 虽然存在， 但实际上是没有任何效果的， 栈容量只能由-Xss参数来设定。 关于虚拟机栈和本地方法栈， 在《Java虚拟机规范》 中描述了两种异常：

1. 如果线程请求的栈深度大于虚拟机所允许的最大深度， 将抛出 `StackOverflowError` 异常。
2. 如果虚拟机的栈内存允许动态扩展， 当扩展栈容量无法申请到足够的内存时， 将抛出 `OutOfMemoryError` 异常。

《Java虚拟机规范》 明确允许Java虚拟机实现自行选择是否支持栈的动态扩展， 而 `HotSpot` 虚拟机的选择是不支持扩展， 所以除非在创建线程申请内存时就因无法获得足够内存而出现 `OutOfMemoryError` 异常， 否则在线程运行时是不会因为扩展而导致内存溢出的， 只会因为栈容量无法容纳新的栈帧而导致 `StackOverflowError` 异常。

 为了验证这点， 我们可以做两个实验， 先将实验范围限制在单线程中操作， 尝试下面两种行为是否能让 `HotSpot` 虚拟机产生 `OutOfMemoryError` 异常： 
 
 使用-Xss参数减少栈内存容量。 **结果：** 抛出 `StackOverflowError` 异常， 异常出现时输出的堆栈深度相应缩小。定义了大量的本地变量， 增大此方法帧中本地变量表的长度。 **结果：** 抛出 `StackOverflowError` 异常， 异常出现时输出的堆栈深度相应缩小。
 
首先， 对第一种情况进行测试

> 虚拟机栈和本地方法栈测试（作为第1点测试程序）

```java
/**
 * VM Args： -Xss128k
 */
public class JavaVMStackSOF {
    private int stackLength = 1;
    public void stackLeak() {
        stackLength++;
        stackLeak();
    }
    public static void main(String[] args) throws Throwable {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}        
```

运行结果：


```java

stack length:989
Exception in thread "main" java.lang.StackOverflowError
at com.lagou.unit.JavaVMStackSOFTest02.stackLeak(JavaVMStackSOFTest02.java:9)
at com.lagou.unit.JavaVMStackSOFTest02.stackLeak(JavaVMStackSOFTest02.java:10)
at com.lagou.unit.JavaVMStackSOFTest02.stackLeak(JavaVMStackSOFTest02.java:10)

//……后续异常堆栈信息省略
```

对于不同版本的Java虚拟机和不同的操作系统， 栈容量最小值可能会有所限制， 这主要取决于操 作系统内存分页大小。 譬如上述方法中的参数-Xss128k可以正常用于32位Windows系统下的JDK 6， 但 是如果用于64位Windows系统下的JDK 11， 则会提示栈容量最小不能低于180K， 而在Linux下这个值则 可能是228K， 如果低于这个最小限制， HotSpot虚拟器启动时会给出如下提示：

```text
The Java thread stack size specified is too small. Specify at least 228k
```

我们继续验证第二种情况， 这次代码就显得有些“丑陋”了， 为了多占局部变量表空间， 不得不定义一长串变量，具体如代码

```java
public class JavaVMStackSOF {
    private static int stackLength = 0;
    public static void test() {
        long unused1, unused2, unused3, unused4, unused5,
        unused6, unused7, unused8, unused9, unused10,
        unused11, unused12, unused13, unused14, unused15,
        unused16, unused17, unused18, unused19, unused20,
        unused21, unused22, unused23, unused24, unused25,
        unused26, unused27, unused28, unused29, unused30,
        unused31, unused32, unused33, unused34, unused35,
        unused36, unused37, unused38, unused39, unused40,
        unused41, unused42, unused43, unused44, unused45,
        unused46, unused47, unused48, unused49, unused50,
        unused51, unused52, unused53, unused54, unused55,
        unused56, unused57, unused58, unused59, unused60,
        unused61, unused62, unused63, unused64, unused65,
        unused66, unused67, unused68, unused69, unused70,
        unused71, unused72, unused73, unused74, unused75,
        unused76, unused77, unused78, unused79, unused80,
        unused81, unused82, unused83, unused84, unused85,
        unused86, unused87, unused88, unused89, unused90,
        unused91, unused92, unused93, unused94, unused95,
        unused96, unused97, unused98, unused99, unused100;
        stackLength ++;
        test();
        unused1 = unused2 = unused3 = unused4 = unused5 =
        unused6 = unused7 = unused8 = unused9 = unused10 =
        unused11 = unused12 = unused13 = unused14 = unused15 =
        unused16 = unused17 = unused18 = unused19 = unused20 =
        unused21 = unused22 = unused23 = unused24 = unused25 =
        unused26 = unused27 = unused28 = unused29 = unused30 =
        unused31 = unused32 = unused33 = unused34 = unused35 =
        unused36 = unused37 = unused38 = unused39 = unused40 =
        unused41 = unused42 = unused43 = unused44 = unused45 =
        unused46 = unused47 = unused48 = unused49 = unused50 =
        unused51 = unused52 = unused53 = unused54 = unused55 =
        unused56 = unused57 = unused58 = unused59 = unused60 =
        unused61 = unused62 = unused63 = unused64 = unused65 =
        unused66 = unused67 = unused68 = unused69 = unused70 =
        unused71 = unused72 = unused73 = unused74 = unused75 =
        unused76 = unused77 = unused78 = unused79 = unused80 =
        unused81 = unused82 = unused83 = unused84 = unused85 =
        unused86 = unused87 = unused88 = unused89 = unused90 =
        unused91 = unused92 = unused93 = unused94 = unused95 =
        unused96 = unused97 = unused98 = unused99 = unused100 = 0;
    }
        
    public static void main(String[] args) {
        try {
            test();
        }catch (Error e){
            System.out.println("stack length:" + stackLength);
            throw e;
        }
    }
}

```

运行结果：

```java
stack length:4582
Exception in thread "main" java.lang.StackOverflowError
at com.lagou.unit.JavaVMStackSOFTest03.test(JavaVMStackSOFTest03.java:27)
at com.lagou.unit.JavaVMStackSOFTest03.test(JavaVMStackSOFTest03.java:27)
at com.lagou.unit.JavaVMStackSOFTest03.test(JavaVMStackSOFTest03.java:27)
//……后续异常堆栈信息省略
```

实验结果表明： 无论是由于栈帧太大还是虚拟机栈容量太小， 当新的栈帧内存无法分配的时候， HotSpot虚拟机抛出的都是StackOverflowError异常。 可是如果在允许动态扩展栈容量大小的虚拟机 上， 相同代码则会导致不一样的情况。 譬如远古时代的Classic虚拟机， 这款虚拟机可以支持动态扩展 栈内存的容量， 在Windows上的JDK 1.0.2运行代码清单2-5的话（如果这时候要调整栈容量就应该改 用-oss参数了） ， 得到的结果是：

```java
stack length:3716
java.lang.OutOfMemoryError
at org.fenixsoft.oom. JavaVMStackSOF.leak(JavaVMStackSOF.java:27)
at org.fenixsoft.oom. JavaVMStackSOF.leak(JavaVMStackSOF.java:28)
at org.fenixsoft.oom. JavaVMStackSOF.leak(JavaVMStackSOF.java:28)
//……后续异常堆栈信息省略
```

可见相同的代码在Classic虚拟机中成功产生了OutOfMemoryError而不是StackOver-flowError异常。 如果测试时不限于单线程， 通过不断建立线程的方式， 在 `HotSpot` 上也是可以产生内存溢出异常 的， 具体如下代码清单所示。

但是这样产生的内存溢出异常和栈空间是否足够并不存在任何直接的关系， 主要取决于操作系统本身的内存使用状态。 甚至可以说， 在这种情况下， 给每个线程的栈分配的内存越大， 反而越容易产生内存溢出异常。 原因其实不难理解， 操作系统分配给每个进程的内存是有限制的， 譬如32位Windows的单个进程 最大内存限制为2GB。HotSpot虚拟机提供了参数可以控制Java堆和方法区这两部分的内存的最大值， 那剩余的内存即为2GB（操作系统限制） 减去最大堆容量， 再减去最大方法区容量， 由于程序计数器 消耗内存很小， 可以忽略掉， 如果把直接内存和虚拟机进程本身耗费的内存也去掉的话， 剩下的内存 就由虚拟机栈和本地方法栈来分配了。 因此为每个线程分配到的栈内存越大， 可以建立的线程数量自然就越少， 建立线程时就越容易把剩下的内存耗尽， 代码如下

**创建线程导致内存溢出异常**

```java
/**
 * VM Args： -Xss2M （这时候不妨设大些， 请在32位系统下运行）
 */
public class JavaVMStackOOM {
    private void dontStop() {
        while (true) {
        }
    }
    
    public void stackLeakByThread() {
        while (true) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    dontStop();
                }
            });
            thread.start();
        }
    }
    
    public static void main(String[] args) throws Throwable {
        JavaVMStackOOM oom = new JavaVMStackOOM();
        oom.stackLeakByThread();
    }
}

```

重点提示一下， 如果读者要尝试运行上面这段代码， 记得要先保存当前的工作， 由于在 Windows平台的虚拟机中， Java的线程是映射到操作系统的内核线程上， 无限制地创建线程会对操 作系统带来很大压力， 上述代码执行时有很高的风险， 可能会由于创建线程数量过多而导致操作系统 假死。

在32位操作系统下的运行结果

```java
Exception in thread "main" java.lang.OutOfMemoryError: unable to create native thread
```

出现 `StackOverflowError` 异常时， 会有明确错误堆栈可供分析， 相对而言比较容易定位到问题所在。 如果使用 `HotSpot` 虚拟机默认参数， 栈深度在大多数情况下（因为每个方法压入栈的帧大小并不是 一样的， 所以只能说大多数情况下） 到达 `1000~2000` 是完全没有问题， 对于正常的方法调用（包括不能 做尾递归优化的递归调用） ， 这个深度应该完全够用了。 但是， 如果是建立过多线程导致的内存溢 出， 在不能减少线程数量或者更换64位虚拟机的情况下， 就只能通过减少最大堆和减少栈容量来换取更多的线程。 这种通过“减少内存”的手段来解决内存溢出的方式， 如果没有这方面处理经验， 一般比 较难以想到， 这一点读者需要在开发32位系统的多线程应用时注意。 也是由于这种问题较为隐蔽， 从 `JDK 7 `起， 以上提示信息中 “`unable to create native thread`”后面， 虚拟机会特别注明原因可能是 “`possibly out of memory or process/resource limits reached`”。

## 方法区和运行时常量池溢出

由于运行时常量池是方法区的一部分， 所以这两个区域的溢出测试可以放到一起进行。前面曾经提到 `HotSpot` 从 JDK 7 开始逐步 “去永久代” 的计划， 并在JDK 8中完全使用元空间来代替永久代的背景故事， 在此我们就以测试代码来观察一下， 使用 `“永久代”` 还是 `“元空间”` 来实现方法区， 对程序有什么 实际的影响。

`String::intern()` 是一个本地方法， 它的作用是如果字符串常量池中已经包含一个等于此String对象的 字符串， 则返回代表池中这个字符串的String对象的引用； 否则， 会将此String对象包含的字符串添加到常量池中， 并且返回此String对象的引用。 在JDK 6或更早之前的 `HotSpot` 虚拟机中， 常量池都是分配在永久代中， 我们可以通过 `-XX：PermSize` 和 `-XX： MaxPermSize` 限制永久代的大小， 即可间接限制其中常量池的容量， 具体实现如代码清单。

> 运行时常量池内存溢出

```java
/**
 * VM Args： -XX:PermSize=6M -XX:MaxPermSize=6M
 */
public class RuntimeConstantPoolOOM {
    public static void main(String[] args) {
        // 使用Set保持着常量池引用， 避免Full GC回收常量池行为
        Set<String> set = new HashSet<String>();
        // 在short范围内足以让6MB的PermSize产生OOM了
        short i = 0;
        while (true) {
            set.add(String.valueOf(i++).intern());
        }
    }
}
```

**运行结果：**

```java
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
at java.lang.String.intern(Native Method)
at org.fenixsoft.oom.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java: 18)
```

从运行结果中可以看到， 运行时常量池溢出时， 在 `OutOfMemoryError` 异常后面跟随的提示信息 是 `“PermGenspace”`， 说明运行时常量池的确是属于方法区（即JDK 6的HotSpot虚拟机中的永久代） 的 一部分。

而使用JDK 7或更高版本的JDK来运行这段程序并不会得到相同的结果， 无论是在JDK 7中继续使 用`-XX：MaxPermSize`参数或者在JDK 8及以上版本使用`-XX： MaxMeta-spaceSize`参数把方法区容量同 样限制在6MB， 也都不会重现JDK 6中的溢出异常， 循环将一直进行下去， 永不停歇。 出现这种变 化， 是因为自JDK 7起， 原本存放在永久代的字符串常量池被移至Java堆之中， 所以在JDK 7及以上版本， 限制方法区的容量对该测试用例来说是毫无意义的。 这时候使用-Xmx参数限制最大堆到6MB就能 够看到以下两种运行结果之一， 具体取决于哪里的对象分配时产生了溢出：

```java
// OOM异常一：
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
at java.base/java.lang.Integer.toString(Integer.java:440)
at java.base/java.lang.String.valueOf(String.java:3058)
at RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:12)

// OOM异常二：
//根据Oracle官方文档，默认情况下，如果Java进程花费98%以上的时间执行GC，并且每次只有不到2%的堆被恢复，则JVM抛出此错误
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
at java.lang.Integer.toString(Integer.java:401)
at java.lang.String.valueOf(String.java:3099)
at com.lagou.unit.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:17)
```

> 方法区内存溢出

方法区的其他部分的内容， 方法区的主要职责是用于存放类型的相关信息， 如类名、 访问修饰符、 常量池、 字段描述、 方法描述等。 对于这部分区域的测试， 基本的思路是运行时产生大量的类去填满方法区， 直到溢出为止。虽然直接使用Java SE API也可以动态产生类（如反射时的 `eneratedConstructorAccessor` 和 `动态代理` 等） ， 但在本次实验中操作起来比较麻烦。 在代码清单

**借助CGLib使得方法区出现内存溢出异常**

```java
/**
 * VM Args： -XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class JavaMethodAreaOOM {
    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy)throws ThrowException{
                    return proxy.invokeSuper(obj, args);
                }
            });
            enhancer.create();
        }
    }
    
    static class OOMObject {}
}
```

在JDK 6中的运行结果

```java
Caused by: java.lang.OutOfMemoryError: PermGen space
at java.lang.ClassLoader.defineClass1(Native Method)
at java.lang.ClassLoader.defineClassCond(ClassLoader.java:632)
at java.lang.ClassLoader.defineClass(ClassLoader.java:616)
... 8 more
```

**方法区溢出** 也是一种常见的内存溢出异常， 一个类如果要被垃圾收集器回收， 要达成的条件是比较苛刻的。 在经常运行时生成大量动态类的应用场景里， 就应该特别关注这些类的回收状况。 

这类场景除了之前提到的程序使用了 `CGLib字节码增强` 和 `动态语言` 外， 常见的还有： `大量JSP或动态产生JSP文件的应用`（JSP第一次运行时需要编译为Java类） 、 `基于OSGi的应用`（即使是同一个类文件，被不同的加载器加载也会视为不同的类） 等。 

在JDK 8以后， `永久代`便完全退出了历史舞台， `元空间`作为其替代者登场。 在默认设置下， 前面列举的那些正常的动态创建新类型的测试用例已经很难再迫使虚拟机产生方法区的溢出异常了。 不过为了让使用者有预防实际应用里出现类似于代码清单2-9那样的破坏性的操作， HotSpot还是提供了一 些参数作为元空间的防御措施， 主要包括：

- -XX： MaxMetaspaceSize： 设置元空间最大值， 默认是-1， 即不限制， 或者说只受限于本地内存 大小。

- -XX： MetaspaceSize： 指定元空间的初始空间大小， 以字节为单位， 达到该值就会触发垃圾收集 进行类型卸载，同时收集器会对该值进行调整： 如果释放了大量的空间， 就适当降低该值； 如果释放 了很少的空间， 那么在不超过-XX： MaxMetaspaceSize（如果设置了的话） 的情况下， 适当提高该值。

- -XX： MinMetaspaceFreeRatio： 作用是在垃圾收集之后控制最小的元空间剩余容量的百分比， 可 减少因为元空间不足导致的垃圾收集的频率。 类似的还有-XX： Max-MetaspaceFreeRatio， 用于控制最 大的元空间剩余容量的百分比。


## 直接内存溢出

直接内存（Direct Memory） 的容量大小可通过`-XX： MaxDirectMemorySize`参数来指定， 如果不去指定， 则默认与Java堆最大值（由-Xmx指定） 一致， 越过了`DirectByteBuer`类直接通 过反射获取`Unsafe`实例进行内存分配（`Unsafe`类的`getUnsafe()`方法指定只有引导类加载器才会返回实例， 体现了设计者希望只有虚拟机标准类库里面的类才能使用`Unsafe`的功能， 在JDK 10时才将`Unsafe` 的部分功能通过`VarHandle`开放给外部使用） ， 因为虽然使用`DirectByteBuer`分配内存也会抛出内存溢出异常， 但它抛出异常时并没有真正向操作系统申请分配内存， 而是通过计算得知内存无法分配就会在代码里手动抛出溢出异常，真正申请分配内存的方法是`Unsafe::allocateMemory()`。

```java
/**
 * VM Args： -Xmx20M -XX:MaxDirectMemorySize=10M
 */
public class DirectMemoryOOM {
    private static final int _1MB = 1024 * 1024;
    public static void main(String[] args) throws Exception {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true) {
            unsafe.allocateMemory(_1MB);
        }
    }
}
```

运行结果：

```java
Exception in thread "main" java.lang.OutOfMemoryError
at sun.misc.Unsafe.allocateMemory(Native Method)
at org.fenixsoft.oom.DMOOM.main(DMOOM.java:20)
```

由直接内存导致的内存溢出， 一个明显的特征是在`Heap Dump`文件中不会看见有什么明显的异常情况， 如果发现内存溢出之后产生的Dump文件很小，而程序中又直接或间接使用了 DirectMemory（`典型的间接使用就是NIO` 或 `Unsafe`） ，那就可以考虑重点检查一下直接内存方面的原因了。

（本篇完）

?> ❤️ 您也可以参与梳理，快来提交 [issue](https://github.com/senlypan/jvm-docs/issues) 或投稿参与吧~