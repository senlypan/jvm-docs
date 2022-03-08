# 引用计数算法

> 作者: 潘深练
>
> 更新: 2022-03-08

## 什么是引用计数算法

引用计数算法可以这样实现：给每个创建的对象添加一个`引用计数器`，每当此对象被某个地方引用时，计数值`+1`，引用失效时`-1`，所以当计数值为`0`时表示对象已经不能被使用。引用计数算法大多数情况下是个比较不错的算法，简单直接，也有一些著名的应用案例。

但是对于Java虚拟机来说，并不是一个好的选择，因为它很难解决对象直接相互`循环引用`的问题。

### 优点：

实现简单，执行效率高，很好的和程序交织。

### 缺点：

无法检测出循环引用。

> 譬如有A和B两个对象，他们都互相引用，除此之外都没有任何对外的引用，那么理论上A和B都可以被作为垃圾回收掉，但实际如果采用引用计数算法，则A、B的引用计数都是1，并不满足被回收的条件，如果A和B之间的引用一直存在，那么就永远无法被回收了

```java
public class App {
  public static void main(String[] args) {
    Test object1 = new Test();
    Test object2 = new Test();
    object1.object = object2;
    object2.object = object1;
    object1 = null;
    object2 = null;
 }
}
class Test {
  public Test object = null;
}
```

这两个对象再无任何引用， 实际上这两个对象已经不可能再被访问， 但是它们因为互相引用着对方， 导致它们的引用计数都不为零，引用计数算法也就无法回收它们 。

**这只是个例子，其实在 java 程序中这两个对象仍然会被回收，因为java中并没有使用引用计数算法。**

（本篇完）

?> ❤️ 您也可以参与梳理，快来提交 [issue](https://github.com/senlypan/jvm-docs/issues) 或投稿参与吧~