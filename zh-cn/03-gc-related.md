# GC相关概念

![访问统计](https://visitor-badge.glitch.me/badge?page_id=senlypan.jvm.03-gc-related&left_color=blue&right_color=red)

> 作者: 潘深练
>
> 更新: 2022-03-02

## ✨ 内容已经在梳理，持续发布
?> ❤️ 您也可以参与梳理，快来提交 [issue](https://github.com/senlypan/jvm-docs/issues) 或投稿参与吧~




## 对象/头/域
## 指针
## Mutator
- 生产垃圾的角色，也就是我们的应用程序，垃圾制造者，通过 Allocator 进行 allocate 和 free。
## 堆
## 活动对象/非活动对象
## 分配
## 分块
## Root
## 伪代码
## 赋值器与回收器
## 存活性、正确性以及可达性
## 原子操作
## 集合、多集合、序列以及元组
## 评价标准
**评价 GC 算法的性能时，我们采用以下 4 个标准。**
- 吞吐量
- 最大暂停时间
- 堆使用效率
- 访问的局部性


（待补充）