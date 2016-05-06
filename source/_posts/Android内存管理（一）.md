title: Android内存管理（一）
tags:
- Android内存管理
- 进程内存
categories: Android内存管理

---

内存管理是所有操作系统的重点，特别是Android、iOS等。移动设备内存较小，因此需要对内存的分配与回收进行更加严格的控制。Android的Dalvik虚拟机扮演了常规的垃圾回收的角色，但在应用开发过程中，仍然需要重视内存分配与释放的时机与地点，不然在比较耗费内存的应用中，OOM可能会时常伴随你的左右~~

可能有童鞋会说，在Android4.4以后，Google已经不再推荐使用Dalvik虚拟机，而采用了新的ART虚拟机。虽然Dalvik支持JIT（Just In Time）机制，并将高频执行的dex字节码翻译成本地机器码，但这种翻译是在应用程序运行过程中发生的，相比Dalvik，ART采用了AOT（Ahead Of Time）机制，在应用程序安装时，就将apk中的dex字节码翻译成本地机器码，ART虚拟机执行的是本地机器码。     

不管Dalvik还是ART，内存分配和垃圾回收都是虚拟机架构设计中的关键部分，有很多共性特征。