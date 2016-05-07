title: Android内存泄露与线程安全
tags:
- Android内存管理
- 进程内存
categories: Android内存管理

---
本次分享主要结合上周Coverity扫描盘古源码发现的问题和平时的实践，对**内存泄露**和**线程安全**这两个问题进行一些说明。扫描发现的BUG大致分为四类：1）空指针；2）除0；3）内存、资源泄露；4）线程安全。第一、二个问题属于编码考虑不周，第三、四个问题则需要更深入的分析。

 1. 内存泄露
 2. 线程安全


----------
## 一、内存泄露
首先，android中的堆内存分为Native Heap和Dalvik Heap。C/C++申请的内存空间在Native Heap中，而Java申请的内存空间则在Dalvik Heap中。
### 1、查看内存占用

 - 命令行：adb shell dumpsys meminfo **packageName**

![](/img/20160507-1.png)

 - 通过Android Studio的Memory Monitor查看内存中Dalvik Heap的实时变化

![](/img/20160507-2.png)
 


 
